---
title:    Enum utilities
document: DXXXX
date:     2025-12-10
audience: EWG, LEWG
author:
 - name:  Matthias Wippich
   email: <mfwippich@gmail.com>
toc: true
toc-depth: 2
---


# Abstract                                                                      {-}

This paper proposes several utilities for enumerations.


\newpage
# Revision history                                                              {-}
### R0 December 2025                                                            {-}

Original version of the paper.


\newpage
# Introduction                                                                  {#intro}

C++ 26 gives us reflection capabilities that can be used with enumerations. However, we currently lack several common queries, shifting the burden onto users. 


# New queries
## `is_fixed_enum_type`
In C++, the semantics of unscoped enumerations without fixed underlying type are different from other enumerations. In such cases, the underlying type is a hypothetical minimum-width integer type that can represent all enumerators.

Currently we do not have a type trait or reflective query to ask whether an enumeration has a fixed underlying type. For completeness, this paper proposes exposing a metafunction `std::meta::is_fixed_enum_type` and a matching type trait `std::is_fixed_enum`.

This is already implementable in C++ today:
```cpp
namespace std {
template <typename T>
struct is_fixed_enum {
  static_assert(std::is_enum_v<T>);
  constexpr static bool value = require { T{0}; };
};

template <typename T>
constexpr bool is_fixed_enum_v = is_fixed_enum<T>::value;

namespace meta {
  consteval bool is_fixed_enum_type(std::meta::info r) {
    return extract<bool>(substitute(^^isfixed_enum_v, {r}));
  }
} // namespace meta
} // namespace std
```

This works, because enumerations with a fixed underlying type can be direct-list-initialized thanks to the carve-out in [[dcl.init.list]/3.8](https://eel.is/c++draft/dcl.init.list#3.8). Conversely, enumerations without fixed underlying type are not - so we can simply test for that.

## Membership and range checks
### Membership
Contracts and reflection have a couple of interesting interactions. To validate inputs, we could for example use a precondition that checks whether an argument of enum type actually corresponds to an enumerator of that enum.

This can be expressed reflectively:
```cpp
template <typename T, typename V>
  requires std::is_enum_v<T> and
           (std::same_as<V, T> or std::convertible_to<V, std::underlying_type_t<T>>)
constexpr bool in_enum(V value) {
  constexpr static auto enumerators =
      define_static_array(enumerators_of(^^T) | std::views::transform([](std::meta::info r) {
                            return extract<T>(constant_of(r));
                          }));
  return std::ranges::contains(enumerators, value);
}
```

Since this is a rather common query, this paper proposes to standardize it as `std::in_enum`.

### Representability
Since not all semantically valid values for a given enum must necessarily correspond to an enumerator, `in_enum` is not sufficient for all uses.

So.. what does it mean for an a value to be in range of an enum?

1. A value `V` can be considered in range of an enum `E` if it lies between the smallest and the largest enumerator. However, this'll yield surprising results if there are gaps between the enumerators. `in_enum` may be a more appropriate query in such cases.

2. For a flag-like enum `E`, a value `V` is in range of `E` if it is larger or equal to `0` and smaller or equal to the largest value representable in a hypothetical minimal-width integer type that can represent all enumerators (sounds familiar?). This will yield an appropriate range that contains all enumerators and combinations thereof.

3. A value `V` can be considered in range of an enum `E` if it is representable by the underlying type of the enum.

This paper proposes a combination of 2 and 3. That is, `std::in_range<E>(v)` checks if a value `v` is representable in some enum `E` and restricting that range for flag-like enums (more on that later).


In order to check representability of some value in an enum `E`, you can almost get away with simply deferring to `std::in_range<U>` with the `U` being the underlying type of `E`. However, for the aforementioned reasons this will not work correctly with unscoped enumerations without fixed underlying type.

```cpp
enum A : unsigned { }; // [0, UINT_MAX]
enum B { };            // [0,1]

static_assert(std::in_range<std::underlying_type_t<A>>(2));
static_assert(std::in_range<std::underlying_type_t<B>>(2)); // oops
```
[Compiler Explorer](https://compiler-explorer.com/z/qnEec1bEh)

To address that, this paper proposes to partially specialize `numeric_limits` for enumeration types and add an overload for `std::in_range`. This gives us the ability to treat unscoped enumerations without fixed underlying types differently, yielding appropriate `min`/`max`. The same carve-out can be used for flag-like enums to restrict to the semantically correct range.


Here's an example using both queries:
```cpp
enum struct Channels {
  A, B, C, D
};

enum Flags { // flag-like
  NONE          = 0,
  ACKNOWLEDGE   = 1 << 0,
  HIGH_PRIORITY = 1 << 1,
  TRACE         = 1 << 2,
  DEFAULTS      = ACKNOWLEDGE | TRACE
};

void dispatch(Channels channel, void* data, Flags flags = DEFAULTS) 
  pre(std::in_enum<Channels>(channel)) // check if the value corresponds to an enumerator
  pre(std::in_range<Flags>(flags))     // combinations are fine, check representability
{
  // ...
}
```

# Flag-like enums
In that last example there is a flag-like enum `Flags`. To get `in_range` to check against the appropriate range [0, 7] instead of the value range of the underlying type, this code uses an unscoped enum with no fixed underlying type.

This is rather subtle and enums with fixed underlying type or even scoped enumerations may be preferred. Having a standardized way to say "this is a flag-like enum, you may OR enumerators" is useful for a couple of reasons. With this information we can improve stringification (and therefore also diagnostics), perform more narrow representability checks regardless of enum kind and help static analyzers (ie by catching dubious bitwise operations with enums that aren't flag-like).

At the time of writing, GCC and Clang already support a vendor attribute [`[[gnu::flag_enum]]`](https://gcc.gnu.org/onlinedocs/gcc/Common-Attributes.html#index-flag_005fenum-type-attribute) and respectively [`[[clang::flag_enum]]`](https://clang.llvm.org/docs/AttributeReference.html#flag-enum) to help with diagnostics. 

## Stringification
For example, consider:
```cpp
enum FileMode {
  BINARY = 0,
  TEXT   = 1 /*<< 0*/,
  READ   = 1   << 1,
  WRITE  = 1   << 2,
  RW     = READ | WRITE
};
```
Stringifying named combinations such as `RW` is still easy - we have an enumerator with that exact value. However, stringifying other flag combinations such as `TEXT | READ` will yield something like `Foo(3)`. While that is still a correct representation of `FileMode::RW`, this'll now require a little bit of thinking to figure out which flags must be combined to get `3`.

Unfortunately we can neither reliably detect such flag-like enumerations automatically, nor would it be correct to just stringify every arbitrary enum value under the assumption that enumerators are combinable.

## Attribute or Annotation?
In C++26 we gained the ability to put annotations on all sorts of entities, including enumerations. Since `[[=std::flag_enum]]` is rather verbose, this paper proposes to introduce an annotation `[[flag_enum]]` for this purpose instead.

::: cmptable
### Attribute
```cpp
enum [[flag_enum]] FileMode {
  BINARY = 0,
  TEXT   = 1 /*<< 0*/,
  READ   = 1   << 1,
  WRITE  = 1   << 2,
  RW     = READ | WRITE
};
```

### Annotation
```cpp
enum [[=std::flag_enum]] FileMode {
  BINARY = 0,
  TEXT   = 1 /*<< 0*/,
  READ   = 1   << 1,
  WRITE  = 1   << 2,
  RW     = READ | WRITE
};
```
:::



## Conditionally supporting bitwise operations
If the `[[flag_enum]]` attribute is made unignorable, we could improve usability for flag-like scoped enumerations by synthesizing bitwise operators for them. While this seems interesting in theory, this paper does not propose it at this time.

# Stringification
- `to_string`/`from_string`
- `format`
- `iostreams`

\newpage
# Wording                                                                       {#wording}

Make the following changes to the C++ Working Draft.  All wording is relative
to [@N5014], the latest draft at the time of writing.

:::add
### [dcl.attr.flag_enum]{.sref} Flag-like enumeration attribute
[1]{.pnum}
The _attribute-token_ `flag_enum` may be applied to the declaration of an enumeration. 
The attribute specifies that the enumerators of the enumeration represent flags or combinations thereof.

:::

### [limits.syn]{.sref} Header <limits> synopsis                        {-}

```
// all freestanding
namespace std {
  // [round.style], enumeration float_round_style
  enum float_round_style;

  // [numeric.limits], class template numeric_limits
  template<class T> class numeric_limits;

  template<class T> class numeric_limits<const T>;
  template<class T> class numeric_limits<volatile T>;
  template<class T> class numeric_limits<const volatile T>;

  template<> class numeric_limits<bool>;

  template<> class numeric_limits<char>;
  template<> class numeric_limits<signed char>;
  template<> class numeric_limits<unsigned char>;
  template<> class numeric_limits<char8_t>;
  template<> class numeric_limits<char16_t>;
  template<> class numeric_limits<char32_t>;
  template<> class numeric_limits<wchar_t>;

  template<> class numeric_limits<short>;
  template<> class numeric_limits<int>;
  template<> class numeric_limits<long>;
  template<> class numeric_limits<long long>;
  template<> class numeric_limits<unsigned short>;
  template<> class numeric_limits<unsigned int>;
  template<> class numeric_limits<unsigned long>;
  template<> class numeric_limits<unsigned long long>;

  template<> class numeric_limits<float>;
  template<> class numeric_limits<double>;
  template<> class numeric_limits<long double>;
```
:::add
```
  template <typename T>
    requires std::is_enum_v<T>
  class numeric_limits<T>;
``` 
:::
```
}
```

\newpage

### [numeric.special]{.sref} `numeric_limits` specializations                   {-}

::: add

[4]{.pnum}

The partial specialization for an enumeration type E matches the specialization for the underlying type of E, with the following exceptions:

```
static constexpr bool min() noexcept;
static constexpr bool max() noexcept;
```

[4.1]{.pnum}

If E is an unscoped enumeration without fixed underlying type, `min()` is the smallest and `max()` the largest value representable by the smallest bit-field large enough to hold all the values of E ([dcl.enum]{.sref}). 

Otherwise, `min()` and `max()` match `min()` and `max()` for the underlying type of E.


```
static constexpr bool digits = @_see-below_@;
static constexpr bool digits10 = digits * 3 / 10;
```

[4.2]{.pnum}

If E is an unscoped enumeration without fixed underlying type, `digits` is the width of the smallest bit-field large enough to hold all values of E ([dcl.enum]{.sref}). 

Otherwise, `digits` is `numeric_limits<underlying_type_t<E>>::digits`.
:::

\newpage

### [meta.syn]{.sref} Header `<meta>` synopsis                        {-}

```
  // associated with [meta.unary.prop], type properties
  ...
  consteval bool is_scoped_enum_type(info type);
```
:::add
```
  consteval bool is_fixed_enum_type(info type);
```
:::

### [meta.unary.prop]{.sref} Type properties                        {-}

Add to table [tab:meta.unary.prop]{.sref}:
```
template<class T>
struct is_fixed_enum;
```
_Condition_: T is an enumeration type with a fixed underlying value.
_Precondition_: T is an enumeration type.

\newpage

### [utility.syn]{.sref} Header `<utility>` synopsis                        {-}

TODO insert in_enum

### [utility.intcmp]{.sref} Integer comparison functions                        {-}

```
template<class R, class T>
  constexpr bool in_range(T t) noexcept;
```

[9]{.pnum}

_Mandates:_ [If R is an enumeration, T is R or convertible to the underlying type of R.]{.add}
[B]{.rm}[Otherwise, b]{.add}oth T and R are standard integer types or extended integer types ([basic.fundamental]{.sref}).

[10]{.pnum}

_Effects:_ Equivalent to: 
```
  return cmp_greater_equal(t, numeric_limits<R>::min()) &&
       cmp_less_equal(t, numeric_limits<R>::max());
```
TODO must cast to underlying type if T is enum

\newpage
# Acknowledgements                                                              {#ack}

Thanks to Michael Park for the pandoc-based framework used to transform this
document's source from Markdown.

Thanks to Peter Bindels for motivating this paper, Brian Bi for suggesting
reuse of `numeric_limits`, Peter Dimov and Will Wray for providing an implementation
of `has_fixed_underlying_type` and numerous other awesome people for giving feedback.

---
references:
  - id: N5014
    citation-label: N5014
    author: Thomas Köppe
    title:  Working Draft, Programming Languages --- C++
    URL:    https://wg21.link/N5014

---
