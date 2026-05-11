---
title:    Removing Digraphs
document: Dxxxx
date:     2025-05-12
audience: EWG, CWG
author:
 - name:  Matthias Wippich
   email: <mfwippich@gmail.com>
toc: true
toc-depth: 2
---


# Abstract                                                                      {-}

This paper proposes removal of digraphs from the C++ language.


\newpage
# Revision history                                                              {-}
### R0 May 2025                                                                 {-}

Original version of the paper.


\newpage
# Introduction                                                                  {#intro}

Digraphs are a complicated solution to a very old problem, that cause more problems
than they solve in a modern environment. Digraphs severely limit the design space of C++,
although as we have seen with @P2996 we are already fine with special-casing our way out
of this pickle.

This however introduces an interesting problem:
If you need to use a source encoding that requires use of digraphs, you **cannot use all of C++26**.

We therefore propose to remove digraphs from the language entirely.

# History
ISO646 (insufficient at the time, there's ISO 646 encodings that don't support all digraphs), trigraphs

# Code survey

Naturally we want to avoid breaking code. This raises an important question: Is there 
code out there, encoded in a way that _needs_ use of digraphs **and** targeting a relatively
modern C++ dialect?

Thanks to GitHub Code search it is possible to find all open-source C++ code matching
a specific pattern. So, here's some data:

- unfiltered
- filtered comment/string pass
- mixed usage
- has trigraphs?
- usage guarded by macro?

# Design Space {#design}
As mentioned before, digraphs severely limit the design space of C++. This isn't an entirely 
new insight, in fact we've ran into issues because of digraphs already and continue to run
into new issues just because digraphs are still a thing.

This leads to a fragmented language - some parts you can write if you need to use digraphs, 
some you don't. At the same time we're accumulating workarounds, which leads to valuable
committee time being spent on deciding how to deal with digraphs and quite some wording bloat.

## Splicers                                                                      {#splicers}
Splicers from @P2996 were accepted with the proposed syntax `[: expr :]`. The alternate digraph
representation `<:` of `[` (and similarly `:>` for `]`) cannot be used for splicers.

The reason for this is rather simple. Writing `[:expr:]` with digraphs would result in an ambiguity 
- is `<::expr` the start of a splicer or a comparison with `::expr`?

## Interpolated string literals {#fstrings}
TODO: mention this continues to come up, ie is `f"foo { bar %> baz"` the digraph `%>` (equivalent to `}`)terminating the f-string replacement field?
The design problems stemming from digraphs do not end there. In some of the recent discussions
around fstrings (@P3412, @P3951) an interesting peculiarity was noted. Consider the following code
```cpp
f"foo { bar %> baz"
```
When we parse this, we must switch to token mode as soon as we see `{` and return to literal parsing mode as soon as we see the corresponding `}`. 
However, `%>` is the digraph for `}`, which raises the uncomfortable question: Should this terminate the field?

Once again we need a workaround to disallow digraphs. Once again we introduce a feature that you can't use if you need to use digraphs.

# Compatibility {#compatibility}
In C++14 we removed support for trigraphs. Since this has been quite a while back now, it is fair
to assume that mitigations for users that required use of trigraphs but wanted to target anything
beyond C++11 are in place. The same strategy should work for digraphs.

If we look at other languages, we can see similar strategies to deal with such situations. For instance,
Python allows defining completely custom decoders for arbitrary source encodings that are controlled
via a magic comment. Such decoders must emit valid UTF-8 encoded Python source code, but are free to produce
that in any means they deem appropriate. This can range from plain decoders to expansion of replacement sequences
and complex macro systems.


\newpage
# Wording                                                                       {#wording}

Make the following changes to the C++ Working Draft.  All wording is relative
to [@N5008], the latest draft at the time of writing.

### [lex.digraph]{.sref} Alternative tokens                                     {-}
Rename to `lex.alt`.

Replace Table 3
::: add
Alternative | Primary | Alternative | Primary
---------------------------------------------
`and` | `&&` | `and_eq` | `&=`
`bitor` | `|` | `or_eq` | `|=`
`or` | `||` | `xor_eq` | `^=`
`xor` |  `^` | `not` | `!`
`compl` | `~` | `not_eq` | `!=`
`bitand` | `&`
:::

This removes the first column
::: remove
Alternative | Primary
---------------------
`<%` | `{`
`%>` | `}`
`<:` | `[`
`:>` | `]`
`%:` | `#`
`%:%:` | `##`
:::


\newpage
# Acknowledgements                                                              {#ack}

Thanks to Michael Park for the pandoc-based framework used to transform this
document's source from Markdown.

---
references:
  - id: N5008
    citation-label: N5008
    author: Thomas Köppe
    title:  Working Draft, Programming Languages --- C++
    URL:    https://wg21.link/n5008

  - id: P2996
    citation-label: P2996
    author: Wyatt Childers, Peter Dimov, Dan Katz, Barry Revzin, Andrew Sutton, Faisal Vali, Daveed Vandevoorde
    title: "Reflection for C++26"
    URL: https://wg21.link/p2996

---
