---

layout: blog
title: "A Callable with a Budget"
date: 2026-07-14
---

`std::function` is the type-erased callable everyone reaches for and nobody quite trusts. The mistrust is earned on two counts. First, allocation: small captures fit in its internal buffer (of unspecified, implementation-varying size), but one capture too many and construction quietly hits the heap — a performance cliff whose location you can't see and that moves between standard libraries. Second, copyability: `std::function` requires its target to be copyable, which is why every codebase that captures a `unique_ptr` in a lambda has a war story, and why C++23 added `std::move_only_function` — which fixes the copy requirement but *still allocates* when the target outgrows its buffer.

Corvid's [`fixed_function`](https://github.com/stevensudit/Corvid/blob/main/corvid/meta/fixed_function.h) is what you get when both problems are solved by refusal: a move-only, type-erased callable that *never* allocates, because a functor that doesn't fit is a compile error, not a heap hit.

```cpp
using callback_t = corvid::fixed_function<64, void(int)>;

callback_t cb = [p = std::move(some_unique_ptr)](int x) { p->handle(x); };
cb(42);
```

The first template argument deserves a careful look, because it's a small design decision with outsized consequences: `SZ` is the **total instance size** in bytes, not the payload capacity. The object holds two function pointers (invocation and lifetime management — more on the second below) plus inline storage, so a `fixed_function<64, ...>` gives you `64 - 2*sizeof(void*)` = 48 bytes of functor space on a 64-bit platform. Why specify the outside rather than the inside? Because the outside is what you're actually budgeting. A `std::array<fixed_function<64, void(int)>, 1024>` of callbacks occupies exactly 64 KiB, cache-line math works in your head, and a struct containing one has a size you chose rather than derived. The class you're describing is a *slot*, and slots are sized from the outside. (The header notes padding can still intervene; alignment is `max_align_t`.)

## The error messages are the interface

Construction from a callable runs a gauntlet of `static_assert`s, and this gauntlet is the soul of the class. Too big for the storage: compile error, in those words. Alignment beyond `max_align_t`: compile error. But the last two asserts are the ones that show real design maturity:

The functor's move constructor must be `noexcept` — asserted, with a message explaining *why*: inline storage means the functor itself relocates when the `fixed_function` moves, so a throwing move would make the wrapper's move constructor throwing, poisoning every container it sits in. `std::function` dodges this by heap-allocating large targets and moving a pointer; `fixed_function` can't dodge, so it demands. Same for the destructor. These aren't limitations discovered at runtime three layers deep in a `std::vector` reallocation — they're stated at the exact line where you handed over the offending lambda.

And then there's my favorite: if the signature's return type `RP` is a reference but the callable returns a prvalue, construction fails with a message ending "every call would produce a dangling reference." That's a whole category of use-after-free — binding a temporary to `int&` through a type-erasure boundary where no compiler warning can follow — converted into a compile-time sentence that names the crime. Type erasure usually *launders* this bug; here the erasure boundary is exactly where it gets caught.

## Two pointers, three jobs

The implementation is a compact idiom worth stealing. Type erasure needs three operations on the hidden functor: invoke it, move it, destroy it. The classic vtable approach spends a pointer to a static table; `std::function` implementations vary. `fixed_function` spends two direct function pointers and a convention:

```cpp
invoke_fn_t  invoke_;    // RP (*)(void*, ARGS...)
lifespan_fn_t lifespan_; // void (*)(void* from, void* to) noexcept
```

`lifespan_` does double duty by parameter contract: called with a destination, it move-constructs `*from` into `to` and destroys `from`; called with `to == nullptr`, it just destroys. One pointer, both lifetime operations, and it moonlights as the empty-state flag (`operator bool` is just `lifespan_ != nullptr`). Meanwhile the *empty* state doesn't null out `invoke_` — it points it at a static function that throws `std::bad_function_call`. That's a deliberate trade: calling an empty `fixed_function` still throws, per `std::function` convention, but the hot path pays no branch for it. Invocation is unconditionally "load pointer, call" — the empty check is encoded in *which function the pointer aims at*, a branchless dispatch trick that's obvious in retrospect and absent from most hand-rolled erasure.

One more deviation from `std::function`, small and principled: `operator()` is non-const. `std::function` lets you invoke a mutable-state functor through a `const` reference — a const-correctness hole formally acknowledged for years. `fixed_function` closes it by the simplest means available: if you hold it const, you don't call it.

## The family declaration

A convenience wrapper at the bottom addresses how these types actually get used in a codebase:

```cpp
using my_fns     = corvid::fixed_function_of<64>;
using callback_t = my_fns::type<void(int)>;
using pred_t     = my_fns::type<bool(int)>;
```

`fixed_function_of<SZ>` pins the budget once and leaves the signature open, so a subsystem declares its slot size in one place and derives its whole callable vocabulary from it. When the budget changes, it changes once.

## Where it sits

Against the field: `std::function` — copyable, allocates, hidden buffer size. `std::move_only_function` — move-only, still allocates past its buffer, and its empty-call behavior is undefined rather than throwing. The `inplace_function` proposal (which the header cites as kin) — never allocates, but specifies *capacity* rather than total size, and last I checked still hadn't landed anywhere official. `fixed_function` is the corner of the design space where the size is total, the failure is at compile time with a message that explains itself, and the anxieties — the throwing move, the dangling reference return, the const-invocation lie — are each individually hunted down.

It won't hold arbitrary callables; that's the deal. You trade "anything fits" for "everything that fits is fast, and everything that doesn't says so at compile time." In a callback slot on a hot path, that's the correct trade, and it's Apache-2.0 at [`corvid/meta/fixed_function.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/meta/fixed_function.h).
