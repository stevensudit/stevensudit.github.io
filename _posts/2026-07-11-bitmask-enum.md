---

layout: blog
title: "The Other Kind of Enum"
date: 2026-07-11
---

In an [earlier post](/2026/07/09/sequence-enum.html) I described how Corvid restores safe, opt-in operators to scoped enums whose values are *sequential*: mutually exclusive options like days of the week. I claimed, in passing, that most enum bugs come from code that never decided which kind of enum it had. This post is about the other kind, and about [`bitmask_enum.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/enums/bitmask_enum.h), which treats it with the different respect it deserves.

A bitmask enum is a set of independent flags. Where a sequence enum answers "which one?", a bitmask answers "which ones?", and every operation that makes sense for one is nonsense for the other. You don't increment a set of flags; you don't OR together two weekdays. C's single `enum` construct served both masters and therefore neither, which is how codebases end up with `(Flags)(f | READABLE)` casts scattered like confetti and a lurking uncertainty about whether `f + 1` means anything. The standard even acknowledges the distinction: the [BitmaskType](https://en.cppreference.com/w/cpp/named_req/BitmaskType) named requirements describe what a flags type should support, and the standard library hand-rolls those operators for types like `std::launch` and `std::filesystem::perms`, over and over, by hand.

Corvid's registration will look familiar from the sequence post: a `consteval` function found by ADL, no macros.

```cpp
enum class rgb { red = 4, green = 2, blue = 1 };
consteval auto corvid_enum_spec(rgb*) {
  return corvid::bitmask::make_bitmask_enum_spec<rgb, "red,green,blue">();
}
```

The names are listed most-significant-bit first, and the spec derives the valid-bit mask from them at compile time. From here, `rgb` satisfies BitmaskType: `|`, `&`, `^`, `~` and their assignment forms all exist, gated by a `BitmaskEnum` concept so they apply only to registered enums. But the interesting parts are where the header goes beyond the named requirements, and where it deliberately deviates.

## Addition as union, subtraction as difference

The header defines `operator+` as `|` and `operator-` as `l & ~r`. At first glance that's redundant sugar; in use, it's a readability instrument. `flags + rgb::red` reads as "with red"; `flags - rgb::red` reads as "without red", and there is no bitwise idiom for "without" that a reviewer parses at a glance, because `f & ~r` requires mentally executing a complement. Set union and set difference are the operations you actually mean when composing flags, so they get the symbols your reader already knows from set theory. On top of these sit predicated helpers — `set_if`, `clear_if`, `set_to` — that absorb the `if (cond) flags |= x;` pattern into a single expression, plus the query quartet of `has` (any), `has_all`, `missing`, and `missing_all`, which replace the classic and classically unreadable `(f & m) == m` family.

## The complement problem

Here is the subtle trap in every hand-rolled bitmask: `operator~`. Complementing a value flips *all* the bits of the underlying type, including the ones your enum never defined. If `rgb` uses three bits of an `int`, then `~rgb::red` has twenty-nine garbage bits set, and they'll happily survive masking operations until they surface somewhere surprising: a serialized value, a switch, an equality comparison that should have matched.

The header offers a layered answer. `flip` is the safe complement: it XORs against the valid-bit mask, flipping only the bits that exist. `~` remains the raw complement by default, because BitmaskType says so and because sometimes you're building an intermediate mask. But register the enum with `wrapclip::limit` and `~` *becomes* `flip`, while `make` (the integer-to-enum cast) becomes `make_safely`, which masks off invalid bits on the way in. The header is candid that this is "a subtle violation of BitmaskType requirements", since a conforming `~` must complement everything, and equally candid that it's opt-in. That's the right posture: the default is standard-conforming, the safety is a choice, and the documentation admits the trade off instead of hiding it.

## Names for bits, or names for values

Registration comes in two flavors, and the distinction is worth a moment. *Bit names* name the individual flags; a value like `rgb::red | rgb::blue` prints as `red + blue`, assembled by walking the bits. Placeholders let you shape the mask: a hyphen marks a bit invalid, a `?` marks it valid-but-unnamed, so `"red,-,?,blue"` is a sparse mask with an anonymous bit in the middle. Any set bits that have no name, whether invalid or anonymous, print as a hex residual tacked on the end, so `red + 0x8` tells you exactly what you're looking at: one named flag and one mystery bit, no information destroyed.

*Value names*, via `make_bitmask_enum_values_spec`, instead name every combination, in sequence from zero. For a three-bit color mask, that's all eight: `"black,blue,green,cyan,red,magenta,yellow,white"`. Now `rgb::red | rgb::blue` prints as `magenta`, because the *combination* is the meaningful unit, not the bits. A `static_assert` insists the name list covers every valid combination, so you can't accidentally register a lookup table with holes.

Parsing runs the other way with the same vocabulary: `lookup` accepts `"red + blue"`, or names mixed with decimal or `0x`-prefixed hex, splits on `+`, ORs the pieces, and rejects malformed input (leading, trailing, or doubled `+`) without touching the output value. Registration buys you round-tripping a flags field through human-readable text, ideal for config files, logs, debug commands.

## Honest edges

Two comments in this header exemplify why I enjoy reading it. The first, on `range_length`: a mask spanning all 64 bits has 2⁶⁴ combinations, which doesn't fit in the return type, so the count wraps to 0. This is described as "confusing but technically correct, which is the best kind of correct." The second, buried in the compile-time mask builder, is a precise little UB note: the loop's final left shift pushes the top bit out of a `uint64_t`, and the comment explains why that's *defined* (unsigned shift is modular; UB requires a shift *count* at or beyond the type's width, not bits falling off the end). Most code either avoids the construct out of superstition or uses it in ignorance; annotating the exact boundary of the rule is rarer and better.

There's also a warning worth repeating here because it will bite someone: define your bitmask enums over an *unsigned* underlying type. The default underlying type of an `enum class` is `int`, and if your valid mask includes the high bit, `max_value()`, which doubles as the all-valid-bits constant, comes out negative. Technically correct again, but the kind of technically correct you want no part of.

## The taxonomy, completed

With this header and its sequence sibling, the claim from the first post cashes out. Declaring which kind of enum you have is a one-time, compile-time act; from then on, the *type system* knows that weekdays increment and don't OR, that flags OR and don't increment, and that neither converts to `int` behind your back. The operators aren't the point. The decision is the point; the operators are just how the compiler holds you to it.

The code is Apache-2.0 at [`corvid/enums/bitmask_enum.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/enums/bitmask_enum.h).
