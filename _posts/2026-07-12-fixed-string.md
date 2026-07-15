---

layout: blog
title: "Strings as Types"
date: 2026-07-12
---

Readers of the [sequence-enum post](/2026/07/09/sequence-enum.html) may have noticed something quietly impossible-looking in the registration syntax:

```cpp
make_sequence_enum_spec<tiger_pick, "eeny,meany,miny,moe">();
```

That's a string literal *as a template argument*. Template parameters are supposed to be types and compile-time values; a string literal is a reference to an array with no portable identity across translation units, which is why `template<const char*>` has never been a usable thing. And yet there it is, and the compile-time name tables in both enum posts depend on it. This post is about the ninety-line class that makes it work: [`fixed_string`](https://github.com/stevensudit/Corvid/blob/main/corvid/strings/fixed_string.h).

C++20 opened the door with a rule change: a non-type template parameter can be of any *structural type*: roughly, a literal type whose non-static data members are all public and themselves structural, with defaulted comparison semantics that let two values be identified as "the same" by the compiler. A `char` array member qualifies. So a class holding its characters *by value* in a public array can be a template parameter, and two instantiations on equal character sequences are the same type. The string's content becomes part of the type's identity. That's the entire trick, and everything in `fixed_string` is in service of satisfying it. Including its most eyebrow-raising line, which we'll get to.

## The shape of it

```cpp
template<CharType CharT, std::size_t N>
struct basic_fixed_string { /* ... */ CharT do_not_use[N + 1]{}; };

template<CharType CharT, std::size_t N>
basic_fixed_string(CharT const (&)[N]) -> basic_fixed_string<CharT, N - 1>;
```

`N` is the length *excluding* the terminator, but the buffer is `N + 1` and zero-initialized, so a `fixed_string` is always terminated, a fact with a payoff below. The deduction guide is what makes `fixed_string{"hello"}` (and the bare literal in a template-argument position) work: a string literal binds as a reference to `const char[6]`, the guide deduces `N = 5`, and the constructor copies the characters into the member array. Copies, not references: that's the point. Once the characters live in the object by value, the object is self-contained and structural, and the compiler can mangle it into a symbol name.

There are two more constructors, both earning their keep in the enum machinery. One takes a pointer plus `std::integral_constant<size_t, N>` — a size carried as a *type tag* — for contexts where you have compile-time characters but no array to bind a reference to, such as building a new string inside a `consteval` function. The other takes exactly `N` individual characters as a pack, with a `requires` clause enforcing the count. Each has its own deduction guide, so `N` is never spelled by hand.

## About that member name

The character array is named `do_not_use`, is public, and is non-const, and the comment on it reads: "This can't be made private or const, but do not ever use it." All three facts are forced. Structural types require public members: make it private and the class can no longer be a template parameter at all. Make it `const` and the defaulted copy machinery and constructor-by-loop break. So the invariant that you'd normally enforce with access control gets enforced with the oldest tool in the box: naming a thing so that using it is self-documenting misconduct. It's funny, but it's also a precise illustration of the structural-type rules: the language demands the member be public, and the language provides no way to say "public for the compiler's identity purposes, not for you."

Accessors provide the sanctioned surface: `view()` and an implicit conversion to `string_view`, iteration, indexing, `size`. And because the buffer is guaranteed terminated, there's `cview()`, returning a [`cstring_view`](/2026/07/10/cstring-view.html) — the terminated view from two posts ago. Note the direction of the guarantee: `fixed_string` doesn't *check* termination the way `cstring_view`'s constructor must, it *provides* it by construction, so the conversion is unconditional. The types compose because each one's invariant feeds the next one's precondition.

## Compile-time concatenation

`operator+` concatenates two fixed strings into a `fixed_string<N1 + N2>` — the *lengths add in the type system*. Since it's all `constexpr`, you can assemble strings at compile time and use the result as a template parameter:

```cpp
constexpr auto path = fixed_string{"corvid/"} + fixed_string{"enums"};
// decltype(path) is basic_fixed_string<char, 13>
```

This is how the enum registry can accept a name list, split it, transform it (the `make_nulled` trick from the first post, which rewrites delimiters as NULs), and hand the result around: every intermediate is a value of structural type, sized exactly, living entirely in the compiler. Comparison operators work across different lengths (`==` and `<`=`>` against any `basic_fixed_string<CharT, N2>`), because two strings of different capacity can still be meaningfully compared even though they can never be equal types.

One conceptual adjustment for newcomers: every distinct string is a distinct *value*, and every distinct length is a distinct *type*. `fixed_string<5>` and `fixed_string<6>` are unrelated classes; `fixed_string{"hello"}` and `fixed_string{"world"}` are different values of the same class, and templates instantiated on them are different instantiations. That's not a limitation, it's the feature. It's what lets `enum_name`'s `consteval` constructor reject a typo'd string at compile time: the string is *right there* in the template argument, fully inspectable.

## Another ghost of the committee

Corvid regulars will sense a theme. The header's own comment notes that, like `cstring_view`, this class "likely owes its existence to a dropped committee proposal": [P0259](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0259r0.pdf), `fixed_string`, by Andrew Tomazos and Michael Price, from 2016, *before* the structural-NTTP rules even landed. The idea has since been revived as [P3094](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3094r0.html), and if `std::fixed_string` ever ships, this header becomes a shim and eventually a memory. Unlike the revanchism of `cstring_view`, this class is a placeholder occupation: holding the territory until the rightful owner shows up. In the meantime, every reflection-adjacent library in the ecosystem has written its own copy of this class, which is perhaps the strongest argument P3094 could cite.

Ninety lines, and it's the load-bearing wall under compile-time enum names, compile-time string validation, and every NTTP-string trick in the library. The code is Apache-2.0 at [`corvid/strings/fixed_string.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/strings/fixed_string.h).
