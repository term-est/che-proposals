---
title:    Attributed friend declarations
document: Dxxxx
date:     2025-07-12
audience: EWG
author:
 - name:  Matthias Wippich
   email: <mfwippich@gmail.com>
toc: true
toc-depth: 2
---


# Abstract                                                                      {-}

This paper proposes lifting the restrictions on attributes appearing on
friend declarations.

# Revision history                                                              {-}
### R0 July 2025                                                                {-}

Original version of the paper.


\newpage
# Introduction                                                                  {#intro}
Way back in C++11 we standardized attributes (see @N2761).

Even in the first revision of the attribute paper (@N2236), attributes were not 
allowed to appear on friend declarations. This proposal aims to lift that 
restriction to align standard attributes better with `__attribute__` as
implemented today, almost two decades later.

# Status Quo                                                                    {#status}

Sentence 3 of [[dcl.attr.grammar]/5](https://eel.is/c++draft/dcl.attr.grammar#5) says

> If an attribute-specifier-seq appertains to a friend declaration ([[class.friend]](https://eel.is/c++draft/class.friend)), 
> that declaration shall be a definition. 

Therefore, both spellings in the following code must be rejected:

```cpp
struct Foo {
  friend void bar [[noreturn]]();  // 1
  [[noreturn]] friend void bar();  // 2
};

[[noreturn]] void bar() {
  std::unreachable();
}
```

So, what do compilers actually do? [Compiler Explorer](https://godbolt.org/z/qvYo971f4)
has an answer.

GCC and Clang reject both `1` and `2`. MSVC accepts both. Additionally _only_ 
MSVC accepts not having the attribute attached to the declaration at all ([Compiler Explorer](https://godbolt.org/z/r5evdq43P)),
while both GCC and Clang complain that the first declaration lacks the attribute. 

To make the above code legal, we would therefore have to write
```cpp
[[noreturn]] friend void bar();

struct Foo {
  friend void bar();
};

[[noreturn]] void bar() {
  std::unreachable();
}
```

Now the friend declaration is no longer the first declaration of `bar` and all
compilers are happy.

As mentioned before, GCC and Clang do complain about the lack of the attribute
on the first declaration of `bar`. With Clang this complaint is a hard error,
with GCC it is only a warning. This got downgraded from an error due to 
@100596 (also see @6f75d62c) to allow the following code.

```cpp
__attribute__((noreturn)) friend void bar();
```

Allowing standard C++ attributes to appear on friend declarations would therefore 
better align with existing practice.

\newpage
# Wording                                                                       {#wording}

Make the following changes to the C++ Working Draft.  All wording is relative
to [@N5008], the latest draft at the time of writing.

## [dcl.attr.grammar]
Modify sentence 5 as follows.

> Each attribute-specifier-seq is said to appertain to some entity or statement, identified by the syntactic
> context where it appears (Clause 8, Clause 9, 9.3). If an attribute-specifier-seq that appertains to some
> entity or statement contains an attribute or alignment-specifier that is not allowed to apply to that entity or
> statement, the program is ill-formed. [If an attribute-specifier-seq appertains to a friend declaration (11.8.4),
> that declaration shall be a definition]{.rm}

## [cpp.predefined]
Bump the feature test macro value of `__cpp_attributes` in [tab:cpp.predefined.ft]
to current year and month at the time of adoption.

| Macro Name         | Value                              |
|--------------------|------------------------------------|
| `__cpp_attributes` | [200809L]{.rm} [20XXXXL]{.add} |


\newpage
# Acknowledgements                                                              {#ack}

Big thanks to Can Çağrı for providing an example implementation of this change
for Clang. It can be found on [Github](https://github.com/term-est/llvm-project/tree/friend-attributes).

---
references:
  - id: N5008
    citation-label: N5008
    author: Thomas Köppe
    title:  Working Draft, Programming Languages --- C++
    URL:    https://wg21.link/n5008
  - id: 100596
    citation-label: Bug 100596
    URL: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100596
  - id: 6f75d62c
    citation-label: GCC commit 6f75d62c
    URL: https://gcc.gnu.org/cgit/gcc/commit/?id=adcb497bdba499d161d2e5e8de782bdd6f75d62c

---
