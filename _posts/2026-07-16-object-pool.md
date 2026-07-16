---

layout: blog
title: "Borrowed, Not Owned: An Object Pool with a Memory"
date: 2026-07-16
---

Object pools have a reputation as the simplest of the "systems programming" containers: preallocate N slots, hand them out, take them back, skip the allocator. The reputation is accurate right up until a reference to a pooled object outlives the borrow — and then the pool's defining feature, *reuse*, becomes its defining hazard: the slot you're still pointing at is now somebody else's object. Same address, new tenant. This is the ABA problem wearing a lifetime-bug costume, and it's why pool code reviews get tense.

Corvid's [`object_pool`](https://github.com/stevensudit/Corvid/blob/main/corvid/containers/utils/object_pool.h) is a thread-safe, fixed-capacity pool whose interesting contribution is that it takes this hazard as the design center rather than a footnote. The cast of characters:

```cpp
corvid::object_pool<Session, 256> pool;

auto h = pool.borrow();       // RAII handle; empty if pool exhausted
h->process(request);          // pointer semantics
// h destroyed → slot returns to the free list (LIFO, for cache warmth)
```

`borrowed` is the owning handle — moveable, resettable, invocable if `T` is callable. So far, conventional. The distinctive piece is the third type:

```cpp
auto t = corvid::object_pool<Session, 256>::token{h};  // non-owning
// ... later, possibly much later ...
if (auto h2 = t.borrow(pool)) { /* slot still mine to claim */ }
```

## A weak_ptr that weighs eight bytes

`token` is to `borrowed` roughly what `std::weak_ptr` is to `shared_ptr`: a non-owning reference that can *attempt* escalation and fail gracefully if the world moved on. But where `weak_ptr` carries two pointers and pings a heap-allocated control block, a token stores an index and a 32-bit generation counter — for small pools, the index is a `uint8_t`, chosen automatically from `N` — and no pointer to the pool at all. You must present the pool to dereference or escalate, which sounds like an inconvenience and is actually a discipline: the token can't dangle *into* a destroyed pool because it can't reach any pool on its own. (It also can't tell two pools of the same `T` apart, which is what the `TAG` template parameter is for — nominal typing as pool identity.)

The generation counter is where the ABA defense lives. Each slot carries an atomic 32-bit value: the low 31 bits are a version, incremented every time the slot is released, wrapping from 2³¹−1 back to 1 because 0 is reserved as invalid; the *high bit* is a borrowed flag. A token snapshots the version at creation. `token::borrow` then succeeds only via a compare-exchange that atomically requires *both* "not currently borrowed" (high bit clear) and "same version as my snapshot" — one CAS, two questions. If the slot was returned and re-lent since your token was minted, the version moved and you get an empty handle instead of someone else's session. `get_ptr` offers the weaker, faster check — version match without claiming — and its comment says the quiet part loudly: since you don't own the slot, the answer is inherently racy the instant you receive it; "this may be fine or it may be a mistake: you get to decide."

All of this is *optional*. Instantiate with `generation_scheme::unversioned` and the generation array vanishes (via `[[no_unique_address]]`, it literally takes no storage), tokens shrink to a bare index, `as_int` — which packs version and index into a `uint64_t` for the trip through a C callback's `void*` or a wire format — gets simpler, and you inherit every footgun the versioning existed to prevent: no staleness detection, and nothing stopping two tokens from borrowing the same slot into a double-return. The header doesn't soften this; the unversioned caveats are stated at each method they afflict. Pay for the memory or accept the risk, but do it knowingly.

## The contracts most pools don't write down

What elevates this header, for my money, is the documentation of its own sharp edges — the places where most pool implementations are silent and users find out empirically.

**Shutdown** is a real protocol, not a destructor comment. `shutdown()` is idempotent, blocks future borrows (the free list is emptied; outstanding tokens are invalidated by bumping every borrowed slot's version so no snapshot matches), and lets already-borrowed handles return safely afterward. And then this, spelled out in full: because the return callback runs *outside* the pool mutex on the normal path (to keep the critical section short) but *under* it during shutdown, a late return racing a shutdown can fire the return callback **twice** on the same slot — once with the real object, once with the freshly reset `T{}`. Therefore the contract: your `ReturnCb` must be idempotent and safe on a default-constructed `T`. That is exactly the sentence that's missing from the pool you're currently using, and its absence is a production incident with a future date on it.

**Callbacks** get their own mutex, separate from the pool's, and both `BorrowCb` and `ReturnCb` are `static_assert`-ed `noexcept` — the pool refuses at compile time to be put in the position of a throwing callback mid-bookkeeping. The guidance on what to *do* in a return callback is also unusually concrete: release locks, don't release buffers — a pooled `std::string` should get `clear()` (keeps capacity), because deallocating in a pool defeats the pool.

**Escape hatches** exist for the C-interop reality that some APIs only ferry a `void*`: `detach` strips a handle down to a raw pointer (you now own the return obligation — the comment invokes the appropriate Spider-Man clause), and `reattach` reconstitutes a handle, validating that the pointer genuinely belongs to this pool's slot array, correct alignment included, before trusting it.

And one honest wart, self-reported: `borrowed`'s copy constructor exists and *throws*. Not deleted — throwing. It's there so `borrowed` can ride inside `std::function`, which demands copyability it will never exercise; the comment names `std::move_only_function` as the thing whose arrival makes this hack deletable. Documenting a workaround with its expiration condition attached is exactly how temporary sins should be committed. (Regular readers will note this is the same C++23 gap that [`fixed_function`](/2026/07/14/fixed-function.html) exists to fill from the other side.)

## What it declines to do

The header ends with two implementation notes that are really a philosophy statement. A lock-free free list is possible — atomic stack push/pop on the indices — and is declined: contention is unlikely, the lock is held for a short fixed span, and "lock-free doesn't guarantee speed." Likewise an intrusive free list interleaved with the slots (better locality, resizability) is sketched, weighed against the loss of packed 8-bit indices and the false-sharing risk, and tabled pending benchmarks. Even the known false-sharing exposure of the generation array is documented as a *tunable-in-waiting* rather than papered over with speculative cache-line padding.

Both times, the same verdict: not without benchmarks. A container that documents the optimizations it *didn't* do, and the evidence that would justify them, is telling you something about the optimizations it did do — namely, that they were also weighed rather than performed as ritual. Fixed capacity, a mutex, generation counters: everything present is there because the alternative was measured or reasoned to be worse, and everything absent has its readmission criteria on file.

Apache-2.0, at [`corvid/containers/utils/object_pool.h`](https://github.com/stevensudit/Corvid/blob/main/corvid/containers/utils/object_pool.h).
