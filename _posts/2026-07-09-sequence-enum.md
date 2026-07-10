---

layout: blog
title: "Giving enum class Back What It Took Away"
date: 2026-07-09
---

When `enum class` landed in C++11, its whole point was subtraction. Classic C enums leaked their enumerators into the enclosing scope, converted to `int` whenever they felt like it, and cheerfully participated in arithmetic that made no sense. Scoped enums fixed all of that by forbidding it all: no implicit conversions, no operators, no leakage. The committee gave us a type that couldn't misbehave because it couldn't do much of anything.

Which is fine, right up until you have a scoped enum whose values are genuinely sequential — days of the week, states in a pipeline, a byte — and you want to write `++day` or `state + 2`. Then you're back to the `static_cast` two-step, littering call sites with casts that are individually harmless and collectively a code smell, because every cast is a place where the type system has been told to look the other way.

The `sequence_enum` facility in my [Corvid library](https://github.com/stevensudit/Corvid) resolves this tension by giving scoped enums their operators back — but deliberately, per-enum, and only the operators that actually make sense. The irony is intentional: it restores exactly what `enum class` was designed to remove, except this time the compiler is in on it.

## The two-minute tour

You opt an enum in by declaring a registration function in the enum's own namespace:

```cpp
enum class tiger_pick { eeny, meany, miny, moe };

consteval auto corvid_enum_spec(tiger_pick*) {
  return corvid::sequence::make_sequence_enum_spec<tiger_pick,
      "eeny,meany,miny,moe">();
}
```

That's the entire ceremony. From here on, `tiger_pick` supports increment, decrement, and heterogeneous addition and subtraction, plus name lookup in both directions:

```cpp
auto t = tiger_pick::eeny;
++t;                        // meany
t += 2;                     // moe
std::cout << enum_as_view(t);  // prints "moe"
```

A few deliberate limitations are worth noticing before we look under the hood. There is no `tiger_pick + tiger_pick`: sequential values are conceptually mutually exclusive options, and adding Tuesday to Thursday is not a Saturday, it's a category error. Only *heterogeneous* arithmetic — enum plus underlying integer — is provided. Conversion back to the underlying type is likewise never implicit; you dereference with `operator*`, borrowing the precedent of `std::optional`:

```cpp
int n = *t;  // explicit, greppable, honest
```

So the design restores capability without restoring the old failure modes. The unscoped-enum bugs came from *implicit* conversion and *nonsensical* arithmetic; what comes back here is explicit conversion and meaningful arithmetic, gated by a concept (`SequentialEnum`) so the operators only exist for enums that asked for them.

## Registration without macros

Anyone who has used an enum-reflection library knows the usual trade: either you wrap your enum in a macro that generates the boilerplate, or you use compiler-specific tricks like `__PRETTY_FUNCTION__` parsing to divine the names. Corvid does neither. Registration is a `consteval` function found by argument-dependent lookup:

```cpp
template<ScopedEnum E>
constexpr inline auto enum_spec_v<E> =
    corvid_enum_spec(static_cast<E*>(nullptr));
```

The pointer is never dereferenced; it exists purely to carry the type into ADL, so your `corvid_enum_spec(tiger_pick*)` overload — a non-template exact match — outranks the library's generic fallback. Every enum gets a spec; registered ones get a spec that enables the machinery. Since the whole resolution happens at compile time through a `constexpr` variable template, there is no runtime registry, no static initialization order to worry about, and no macro anywhere.

The names themselves are handled with a trick I'm rather fond of. The name list arrives as a `fixed_string` non-type template parameter, which means it has static storage duration. At compile time, `make_nulled` produces a length-preserving copy with every delimiter replaced by a NUL:

```
"eeny,meany,miny,moe"  →  "eeny\0meany\0miny\0moe\0"
```

Each name is now an independently null-terminated span *in place*, so the parsed names are the backing for `cstring_view` instances that guarantee null termination, pointing directly into that static buffer. No allocation, no copying, no runtime parsing, and every name is safe to hand to a C API. The parse itself runs in `consteval`, so a malformed registration string is a compile error, not a surprise.

## Sparse enums: the segment table

A dense name list indexed from `minseq` covers most enums, but real-world enums — error codes, protocol constants — often have gaps. Registering `{ ok = 0, warning = 100, fatal = 10'000 }` with a dense list would mean ten thousand empty entries.

The segmented form breaks the registration into runs:

```cpp
make_sequence_enum_spec<status, "0,ok|100,warning|10000,fatal">();
```

Each `|`-delimited segment starts with its absolute value, and storage is sized to the *named count*, not the value range. Lookup by value walks the segment table — a handful of entries — and indexes into the packed array; a dense enum is a single segment, so its lookup is O(1). Lookup by name is a linear scan, which sounds alarming until you remember that enums have dozens of values, not millions, and the scan is `constexpr`-friendly, which matters for what comes next.

One detail here shows the cost model was actually thought through: segments must start at least four values past the end of the previous one, enforced at compile time. Why four? Because a gap of one to three values costs about as much stored as empty names inside one segment as it would as a new segment — which also pays for its start value and delimiter. Runs any closer are rejected with instructions to merge them. It's a small thing, but it's the difference between a facility and a design.

## Wrapping: modular arithmetic that's actually exact

By default, walking off the end of the range is undefined behavior, same as any enum arithmetic. But register with `wrapclip::limit` and every operation becomes exact modulo the sequence size:

```cpp
enum class weekday { mon, tue, wed, thu, fri, sat, sun };
// registered with wrapclip::limit

auto d = weekday::sun;
++d;        // mon
d -= 8;     // sun again
```

This is the part of the header where the engineering gets serious, because "wrap the value" is one of those problems that is trivial to state and easy to get subtly wrong. Consider the hazards: the range may hug the extremes of the underlying type, so `hi + 1` overflows before you can test it; the offset may be negative with magnitude `INT64_MIN`, which famously cannot be negated; and the range may span the *entire* underlying type, making the size itself unrepresentable.

The implementation handles each one exactly. All arithmetic is hoisted into `uint64_t`, where wraparound is defined and the widened subtraction `max - min + 1` is exact for any underlying type and signedness. Negative offsets are folded by `seq_residue`, which computes the magnitude as `0 - mag` in unsigned arithmetic — safe even for the type's minimum, where naive negation is UB — and reflects the remainder around the size. The modular add is then phrased as a comparison-and-adjust on offsets already reduced to `[0, size)`:

```cpp
const auto noff = (radd >= size - off) ? radd - (size - off) : off + radd;
```

so no intermediate value can overflow, ever.

And the full-range case gets my favorite resolution in the file: it's handled by *deleting the code*. `seq_actually_need_wrap_v` observes that if the sequence spans the whole underlying type, conversion to that type is already modular — the wrap happens for free in the `static_cast` — so the wrapping branches compile away entirely. `std::byte` with range [0, 255] wraps correctly at zero cost, not because of clever code, but because of the correct observation that no code is needed.

## Names that fail at compile time

The piece most likely to change how your call sites read is `enum_name<E>`. It's a strongly typed wrapper around a string view whose `consteval` constructors validate the name against the registry — at compile time:

```cpp
using color_name = enum_name<color>;

void paint(color_name c);

paint("red");       // OK
paint("reed");      // compile error: not a registered name
paint(color::red);  // OK: name looked up from the value
```

Read that middle line again. A typo in a string literal is a *build failure*. The mechanism is straightforward once the earlier machinery is in place: the constructor is `consteval`, the lookup is the `constexpr` linear scan from the registry, and a failed lookup throws — which, in a `consteval` context, means the program doesn't compile. The result is also interned: the stored view points at the registry's canonical static copy, not your literal, so equal names share identical storage and can be compared by pointer.

Runtime construction is available through checked factories — `intern` throws on unknown names, `try_intern` returns an `optional` — plus an explicit `force` for the caller who accepts a raw, unvalidated view and vouches for its lifetime. (The header notes there is no `try_force`, or `triforce`: only do `force`. I refuse to apologize.) A companion type, `enum_named_value<E>`, carries the resolved enum value alongside the validated name for call sites that need both, resolving the pair in the same `consteval` constructors so neither costs anything at runtime.

If you want the full literal-suffix experience, the class is designed to support a UDL in two lines:

```cpp
consteval color_name operator""_color(const char* s, std::size_t n) {
  return color_name{s, n};
}

auto c = "red"_color;   // OK
auto d = "reed"_color;  // compile error
```

## Enums done right — for this case

I've been careful to say *sequential* throughout, because that's the honest scope of this facility. Corvid's enum support treats sequences and bitmasks as different animals with different registrations, different valid operations, and different safety semantics — `bitmask_enum.h` is the sibling header, and clipping rather than wrapping is its version of `wrapclip::limit`. A sequence enum is a set of mutually exclusive options; a bitmask is a set of independent flags; and most enum bugs in the wild come from code that never decided which one it had.

That, ultimately, is the claim behind "enums done right." Not that scoped enums were a mistake — they weren't — but that forbidding everything was a blunt instrument, and the type system of 2026 is sharp enough for a finer one. With concepts to gate the operators, `consteval` to move validation to the build, ADL to make registration non-invasive, and NTTP strings to make the names free, a scoped enum can finally say what it is and get exactly the operations that follow — no casts, no macros, and no way to spell the bugs the old enums invited.

The code is Apache-2.0 and lives in [`corvid/enums/sequence_enum.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/enums/sequence_enum.h), with the registry plumbing in [`enum_registry.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/enums/enum_registry.h) and [`scoped_enum.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/enums/scoped_enum.h).
