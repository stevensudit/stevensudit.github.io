---

layout: blog
title: "A Revanchist string_view"
date: 2026-07-10
---

For immutable text, `std::string_view` beats `std::string` in every way that matters. It's two words wide, it never allocates, it never copies, it's trivially copyable and `constexpr`-friendly, and it can point into anything — a literal, a buffer, the middle of somebody else's string — without owning or disturbing it. If your text is not going to change, there is exactly one thing `std::string` still has over it: the guarantee that a terminator follows the last character.

That one thing turns out to be a hostage situation. The moment you call a C API — which is to say, the moment you talk to any operating system ever shipped — you need a `const char*`, and `string_view` cannot produce one. It has no `c_str()`, and the omission is honest: the view makes no promise about the byte past its end. So you're marched back to `std::string` and everything you left it for — `std::string{sv}.c_str()`, an allocation and a copy to purchase a single byte — or you pass `sv.data()` and quietly promise yourself the view came from something terminated. The first is the tax; the second is the bug waiting to file its paperwork.

This gap was noticed, of course, and a fix was proposed: Andrew Tomazos's [`std::cstring_view`](http://open-std.org/JTC1/SC22/WG21/docs/papers/2019/p1402r0.pdf), a `string_view` that additionally guarantees termination. The committee, as is its custom with useful things, rejected it. So my [Corvid library](https://github.com/stevensudit/Corvid) includes what the header comment cheerfully describes as a revanchist implementation: [`cstring_view`](https://github.com/stevensudit/Corvid/blob/main/corvid/strings/cstring_view.h), reclaiming the lost territory. This post is about what it took to build it right, because the interesting part isn't the invariant — it's what the invariant does to every corner of the API.

## The invariant, and the constructor problem

The promise is simple to state: for any `cstring_view` with non-null `data()`, the range `[data(), data() + size()]` — note the closed bracket — is valid, and `data()[size()]` is the terminator. As with `std::string`, the terminator exists but isn't counted. Given that, `c_str()` is trivial and total: it never returns `nullptr`, substituting a static empty string when the view is null.

Constructing from a `std::string` or a `const char*` is easy, since both guarantee termination by contract. The hard case is the one that makes or breaks the type: constructing from a `(pointer, length)` pair or a `std::string_view`. The constructor must verify the terminator is present, and it *cannot look*. The byte at `ptr[len]` is outside the range the caller handed over; nothing says it's the terminator, and nothing even says it's dereferenceable. Peeking would be undefined behavior committed in the name of safety. The alternative — just trusting the caller — turns the invariant into an unverifiable implicit precondition, which is precisely the disease this type is meant to cure.

The solution is a small shift of protocol with outsized consequences: when constructing from a length-carrying, termination-agnostic source, *the caller must include the terminator in the length*. The constructor inspects the last character in the now-legal range, confirms it's `'\0'`, and shaves it off:

```cpp
if (sv.back()) throw std::invalid_argument("cstring_view arg");
sv.remove_suffix(1);
```

If the terminator isn't there, that's a logic error and it throws. What was an unverifiable promise is now a verifiable claim: you asserted the buffer extends through a terminator, and the constructor checked. These constructors are also `explicit` — you can convert *to* `std::string_view` implicitly, because that direction only forgets a guarantee, but converting *from* one requires a visible decision, because that direction manufactures a guarantee.

One consequence worth internalizing: an empty-but-non-null `cstring_view` follows the same protocol. A `string_view` of length zero with a non-null pointer has no legal byte to check, so it's rejected; to build the empty (but terminated, but non-null) view, you pass the terminator as the sole character, length one. Empty views with null data are fine as-is — there's nothing to verify about nothing.

## Null is not empty

That last distinction isn't pedantry; it's the second feature. `std::string_view` technically lets you distinguish a default-constructed view (`data() == nullptr`) from a view of an empty string, but nothing in its interface treats that as meaningful. `cstring_view` promotes it to a first-class, `std::optional`-like state, and the motivating example is sitting in every process you've ever run:

```cpp
cstring_view path = std::getenv("PATH");
if (!path) { /* unset — not the same as set-but-empty */ }
```

`getenv` returns `nullptr` for an unset variable and an empty string for a set-but-empty one, and C++ has never had a natural type for that return value. `std::string` erases the distinction; `std::optional<std::string>` preserves it at the cost of an allocation and a mouthful. `cstring_view` holds it losslessly and for free — the null-pointer constructor maps `nullptr` to the null view rather than exhibiting `string_view`'s cheerful undefined behavior — and the optional interface follows: `has_value()`, `operator bool`, a throwing `value()`, `value_or`, and `as_optional()`.

Comparison needed a decision here, and the one taken is pragmatic: `==` treats null and empty as equal, because most code genuinely doesn't care, and a `same()` method exists for code that does. Two states, two predicates, no surprises.

## The API an invariant permits

My favorite detail in the class is an asymmetry. `remove_prefix` is present; `remove_suffix` and `substr` are not. This falls straight out of the invariant: trimming characters off the front leaves the terminator exactly where it was, so a `cstring_view` survives it. Trimming the back, or slicing the middle, would produce a view whose last character has arbitrary bytes after it — the invariant dies, so the operations don't exist. If you need them, you convert to `std::string_view` (implicit, free) and slice that, having consciously stepped down to the weaker guarantee.

This is what invariant-driven API design looks like: the method set isn't "what `string_view` has, minus the scary bits," it's "what the promise permits," derived rather than curated. It's also the answer to anyone who suggests achieving this with inheritance from `std::string_view` — a derived type can't *remove* `substr`, so the invariant would be a comment, not a contract.

## Literals, including one that reads your environment

The friction of the include-the-terminator protocol is real at call sites with literals, so a UDL absorbs it:

```cpp
using namespace corvid::literals;
constexpr auto greeting = "hello"_czsv;
```

The trick is small and satisfying: for `"hello"`, the compiler hands the UDL a length of 5, but string literals are always terminated, so the operator passes `n + 1` to the checking constructor, which verifies the terminator it now legally sees and trims it back off. The operator is `consteval`, so this all evaporates at compile time. There's a companion integer literal, `0_czsv`, that produces the null view and rejects any other operand — and then there's this one, which I'll defend as genuinely useful rather than merely cute:

```cpp
auto path = "PATH"_env;   // getenv, as a literal
```

It returns a `cstring_view`, which — as established above — is the exact shape of `getenv`'s answer, unset-versus-empty distinction intact.

## The family underneath

`cstring_view` isn't a one-off; it's a child of a CRTP base, [`string_view_wrapper`](https://github.com/stevensudit/Corvid/blob/main/corvid/strings/string_view_wrapper.h), that captures the pattern "a value type holding a `string_view` plus a null regime plus an invariant." Its other children include `opt_string_view` (the null regime with no further constraints) and — readers of the [previous post](/2026/07/09/sequence-enum.html) will recognize this one — `enum_name`, whose invariant is that the view names a registered enumerator.

The division of labor in the base is the part worth studying. Read-only and search operations forward to the underlying view and live in the base, since no invariant can object to being looked at. Operations that *return* a value — `value()`, the child-typed `value_or`, `as_optional()` — return the child type rather than degrading to `std::string_view`, so the invariant travels with the result. And the reslicing operations live in the children precisely because, as we saw, their safety depends on which invariant is in force: `remove_prefix` is fine for `cstring_view` but would break `enum_name`; `remove_suffix` breaks both. The base makes no assumption it can't keep on behalf of every child.

The base also contains the file's best war story, down in the `std::formatter` plumbing. The wrappers forward `begin` and `end`, which makes them ranges, which means `std::format` would happily print one as `['h', 'i']`. Disabling range formatting via `std::format_kind` is the documented escape hatch — but the wrapper still matches the standard's own range-keyed partial specialization, and the two constrained specializations don't subsume each other. Ambiguous partial specializations are ill-formed, and clang and MSVC resolve the situation differently. The fix is to restate the standard's constraints verbatim on Corvid's specialization, so it strictly subsumes the standard's and wins everywhere — constraint subsumption deployed not as a curiosity but as a portability tool. If you've never had to make one partial specialization *provably more constrained* than another, I recommend the experience once, in someone else's codebase.

## Reclaimed

None of this is exotic. The type is small, final, allocation-free, and `constexpr` throughout; it's what `string_view` interop with C should have looked like from the start, and very nearly did. The committee may yet come around — rejected papers have risen before. Until then, the territory is held at [`corvid/strings/cstring_view.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/strings/cstring_view.h), Apache-2.0, wide and Unicode variants included.
