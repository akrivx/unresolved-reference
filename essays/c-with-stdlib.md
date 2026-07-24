# C with stdlib: A Two-Tier Model of C++

You've seen this code before. It uses `std::vector`, `std::string`, maybe a `std::mutex` or a `std::thread` — but structurally, it reads like C. Enums instead of class hierarchies. Structs instead of classes with invariants to maintain. Free functions instead of methods. `printf` and `fprintf` instead of `std::cout`. It compiles, it runs correctly, and it makes surprisingly little use of anything you'd call C++ idiom beyond the containers and smart pointers themselves.

Call it C with stdlib. (You'll also hear "C with STL," which is the more common name and the less accurate one — `<mutex>` and `<thread>` were never part of the STL, which historically meant containers, iterators, and algorithms. The style covers more ground than that name implies.)

The usual read on this style is that it's C++ half-adopted: a way station on the road to "real" C++, or a tell that the author never got past the parts of the language that look like C. I think that's backwards, and I want to press two different audiences on it. For engineers: should this style be adopted on purpose, or at least stop being read as unfinished C++? For the people who steer where the language goes next: should the model behind it be something C++ designs for on purpose, rather than something left to convention and third-party forks? Neither question is obvious on its own, so here's one small piece of code, evolved twice, that I think earns both of them an answer.

## Building a Widget in C

Say you're writing a `Widget` that owns three resources: a file, a heap buffer, and a mutex. In C, construction has an awkward property: it can fail partway through, and when it does, you have to release exactly the resources that succeeded and none that didn't. The naive way to write this — an `if` at every step, each with its own inline cleanup — duplicates that release logic at every exit point. The standard fix is `goto cleanup`: one label, one shared teardown path, reached from wherever construction gives up.

Let's also make destruction itself fallible, since a realistic `widget_destroy` should be able to report that closing the file failed:

```c
#include <stdio.h>
#include <stdlib.h>
#include <threads.h>

typedef enum {
    WIDGET_OK = 0,
    WIDGET_ERR_ALLOC,
    WIDGET_ERR_OPEN,
    WIDGET_ERR_LOCK,
    WIDGET_ERR_DESTROY,
} WidgetStatus;

typedef struct {
    FILE* file;
    char* buffer;
    mtx_t lock;
    int lock_initialized;
} Widget;

WidgetStatus widget_destroy(Widget* w) {
    if (!w) return WIDGET_OK;
    WidgetStatus status = WIDGET_OK;

    if (w->lock_initialized) mtx_destroy(&w->lock); // void -- can't report failure at all

    free(w->buffer);

    if (w->file && fclose(w->file) != 0) status = WIDGET_ERR_DESTROY;

    free(w);
    return status;
}

WidgetStatus widget_create(const char* path, Widget** out) {
    *out = NULL;
    Widget* w = calloc(1, sizeof(Widget));
    if (!w) return WIDGET_ERR_ALLOC;

    WidgetStatus status = WIDGET_OK;

    w->file = fopen(path, "r");
    if (!w->file) { status = WIDGET_ERR_OPEN; goto cleanup; }

    w->buffer = malloc(4096);
    if (!w->buffer) { status = WIDGET_ERR_ALLOC; goto cleanup; }

    if (mtx_init(&w->lock, mtx_plain) != thrd_success) { status = WIDGET_ERR_LOCK; goto cleanup; }
    w->lock_initialized = 1;

    *out = w;
    return WIDGET_OK;

cleanup:
    widget_destroy(w);   // its return value is discarded here -- is that right?
    return status;
}
```

This works, and it's not a naive way to write it — `goto cleanup` is the standard, sanctioned pattern for exactly this problem. But look at what it costs once destruction can fail too.

`lock_initialized` exists because `mtx_t` has no empty state. A pointer can be `NULL`; a `mtx_t` can't signal "not yet initialized" on its own, so the struct has to carry a second field just to track how far construction got. That bookkeeping burden isn't even uniform across resource types — it depends entirely on what the underlying resource gives you to check against.

The cleanup path has two errors competing for one return slot. When `mtx_init` fails, `status` is already `WIDGET_ERR_LOCK`. Then `widget_destroy` runs, to release the file and buffer that already succeeded, and it might *also* fail. Its return value is silently discarded on that line. Is that the right call? Maybe — but it's not derivable from anything in the language. It's a policy `widget_create`'s author invented, and a caller has no way to know it without reading the source.

And `mtx_destroy` returns `void`. Some C APIs don't even give you the option to observe a teardown failure. So "what do we do when cleanup fails" isn't a question every dependency lets you ask, let alone answer consistently.

None of this is a knock on C, or on whoever wrote this. Given what the language provides, this is close to the disciplined way to write it. The interesting question is what happens to these three problems when the same `Widget` is written in C++.

## The Same Widget, in C++

Here's the same `Widget`, in C++:

```cpp
#include <fstream>
#include <vector>
#include <mutex>
#include <optional>
#include <string>

struct Widget {
    std::ifstream file;
    std::vector<char> buffer;
    std::mutex lock;
};

std::optional<Widget> make_widget(const std::string& path) {
    std::ifstream file(path);
    if (!file) return std::nullopt;

    std::vector<char> buffer(4096);

    return Widget{std::move(file), std::move(buffer), std::mutex()};
}
```

That's the whole thing. No destructor, no copy or move constructor, nothing written — the Rule of Zero. If the file fails to open, the caller gets `std::nullopt`, and nothing downstream was ever constructed, so there's no partial state to reason about. If a later member's construction had failed instead, everything already built would unwind automatically via its own destructor, in reverse order, with no `goto` and no manual bookkeeping.

One thing worth a short aside, because it looks suspicious the first time you see it: `std::mutex` is neither copyable nor movable, so `return Widget{...}` looks like it should fail to compile — surely returning the aggregate by value requires moving it? It compiles because of guaranteed copy elision (C++17): `Widget{...}` here is a prvalue, so the compiler constructs it directly in the caller's storage rather than building a temporary and moving from it. Each member, including `lock`, is initialized in place; no `std::mutex` object is ever moved, because there's no separate object to move from. Before C++17 this depended on the compiler being willing to elide the copy as a courtesy; now the standard requires it. It's a small detail, but it's a case where a *language* guarantee, not a library primitive, is what makes this composition frictionless.

Compare this to the three problems from the C version. The sentinel field is gone, because `Widget` never exists in a partially-built form the caller can observe. The two-errors-one-slot problem hasn't been solved yet — we'll get to why in a moment — but the mechanism that caused it, a single shared cleanup routine trying to signal for two different failures, isn't there anymore, because there's no cleanup routine at all. And `mtx_destroy`'s void return is no longer your problem, because you never call it; `std::mutex`'s destructor absorbed that decision when it was written.

## This Is "C with stdlib"

Notice what the code above doesn't use. No inheritance, no virtual dispatch, no operator overloading, no custom templates. It's a struct with three members and a free function that returns it wrapped in `optional`. Swap in `printf` for the handful of places this example would otherwise reach for `<<`, and you have a complete instance of C with stdlib: application code that looks almost exactly like C, sitting on top of standard-library types that happen to have constructors and destructors.

That's the pattern, once you notice it: the C++-specific machinery — move constructors, exception safety, allocator-failure handling — isn't absent from this code. It's just not being *written* by this code. Someone wrote it once, for `std::mutex` and `std::ifstream` and `std::vector`, and this `Widget` is just spending it.

Which raises the obvious question: who's doing the hard work, and where did the rest of it go?

## A Seam: Two Failure Modes, One Visible

Here's where the "just compose primitives" story gets a real seam, and it's worth sitting with, because it's not a contrived edge case.

`make_widget` returns `std::optional<Widget>`. Its signature reads as a promise: failures come back as an empty optional, check it and move on. But look again at the body. `std::ifstream file(path)` failing routes into `return std::nullopt`, deliberately. `std::vector<char> buffer(4096)` failing does not — if the allocator throws `std::bad_alloc`, that exception propagates straight out of `make_widget`, including past a caller who assumed the return type covered every failure:

```cpp
if (auto w = make_widget(path)) { /* ... */ }
else { /* handle failure */ }
```

There's no `catch` anywhere in that snippet, and there doesn't need to be, as far as the signature is concerned — the return type told the caller where failures live. It didn't tell the whole truth. And this isn't a rare case: almost every standard container's allocating operations can throw, and almost no calling code guards against it.

The fix is to actually own both channels, with the same `Widget` from before:

```cpp
#include <expected>

enum class WidgetError { Open, Alloc };

std::expected<Widget, WidgetError> make_widget(const std::string& path) noexcept {
    std::ifstream file(path);
    if (!file) return std::unexpected(WidgetError::Open);

    try {
        std::vector<char> buffer(4096);
        return Widget{std::move(file), std::move(buffer), std::mutex()};
    } catch (const std::bad_alloc&) {
        return std::unexpected(WidgetError::Alloc);
    }
}
```

Now both failure modes are visible in the type, whether they started life as a falsy check (`ifstream`) or an exception (`vector`) — `make_widget` is the boundary that absorbs the inconsistency in how different primitives report construction failure, and re-exposes one uniform contract. That's worth choosing deliberately over just letting the exception surface, for reasons beyond "it's more fine-grained": the type documents the contract at the call site instead of relying on the caller having read the implementation; it lets you distinguish an anticipated, recoverable outcome (no file at that path) from something closer to a genuine emergency (real memory exhaustion); and in a hot path where "file missing" is common rather than rare, it avoids paying exception-unwind cost for a routine outcome. (Exceptions still win when you want a failure to sail through many layers of code that have no opinion on it — forcing `expected` through call stack that doesn't care about the error means touching every intermediate return type, `and_then` chains notwithstanding. That's a real cost, not a solved problem, just a different one.)

The `noexcept` on the signature is doing something specific, and it's worth being precise about it: it's not error handling, the `try`/`catch` is. It's an assertion — "every exception this function can produce is already caught above." If that assertion is ever wrong (someone adds a call that throws something other than `bad_alloc` and forgets to widen the `catch`), the result isn't a silently violated contract. It's an immediate `std::terminate` at the exact call site of the mistake, which is a far easier bug to find than a stray exception unwinding through frames that have nothing to do with it.

Notice, too, that this isn't a third failure channel alongside "optional" and "exceptions." It's the same channel, `expected`, with construction-time failures funneled into it regardless of which primitive originally signaled the problem. The real split is somewhere else, and it's the one that resolves the two-errors-one-slot problem from the C version.

## What Happens When Teardown Fails?

We still haven't answered the C version's hardest question: what happens when *destruction* fails? `Widget` has no user-written destructor, so nothing to fail there directly — but its members' destructors run regardless, and one of them, closing the file, is exactly the operation that could fail in the C version.

C++'s answer: destructors are implicitly `noexcept` starting in C++11. If a destructor, or something it calls, tries to let an exception escape, `std::terminate` is called immediately. Full stop, program ends. That's the same fail-fast idea as the `abort()`-on-cleanup-failure discipline from the `mtx_destroy`/`fclose` discussion above, except it's the *default* for every destructor in the language, not a convention each API author has to remember and apply consistently. There's a stricter version of the same rule, too: even a destructor explicitly written to allow exceptions (`noexcept(false)`) will still call `std::terminate` if it throws while the stack is already unwinding because of some other exception in flight — because the language has no defined way to propagate two exceptions at once, and rather than inventing a merge policy, it just forecloses the situation.

That last point is the honest counterpart to the "two errors, one slot" problem from the C version. `widget_destroy`'s return code and `widget_create`'s cleanup-path error were both fighting for the same `WidgetStatus` value, and someone had to decide which one wins — a policy invented per API, undiscoverable without reading source. C++ doesn't resolve that competition; it prevents it from arising, by never letting a destruction failure and a construction failure occupy the same channel in the first place.

One caveat worth keeping, because the story shouldn't oversell itself: not every standard type's destructor treats a failure the way `terminate` would suggest. `std::ofstream`'s destructor calls `close()` internally, and if `close()` fails, that failure is simply discarded — no exception is thrown, and by the time the destructor runs there's no way for calling code to observe the failure at all. That's not a violation of the implicit-noexcept-destructor rule, since nothing actually throws; it's a design choice to prioritize never-throw-from-destructor so strongly that even a real failure gets swallowed rather than escalated. So the accurate claim isn't "C++ solved destruction failure" — it's "C++ turned destruction failure into a single well-known default, terminate or swallow, instead of an ad hoc policy every C API reinvents on its own."

## The Two-Tier Model

Now name what's actually happening across both examples. Two tiers:

**Tier 1** — the authors of `std::mutex`, `std::ifstream`, `std::vector`, and anyone who writes a type like them — absorbs move semantics, exception safety, and construction/destruction lifecycle-failure handling once, carefully, so no one downstream has to redo it.

**Tier 2** — application code, `Widget` and `make_widget` included — composes Tier-1 primitives into structs and functions, and gets a trivial destructor (or none at all) for free, precisely *because* it isn't managing any raw resource itself. This is the Rule of Zero as a direct consequence of tiering, rather than a style preference: your destructor is trivial when, and because, every member's destructor already does the real work.

If this model is right, it predicts something checkable: friction in idiomatic C++ shouldn't be spread evenly across the language, it should cluster at the *seams* between tiers — the places where a Tier-2 author has to reach up and either author a primitive or reconcile two primitives' inconsistent contracts. The `optional`-hides-`bad_alloc` gap above is exactly that kind of seam: an inconsistency between how `ifstream` (falsy check) and `vector` (exception) report construction failure, invisible until you go looking for it.

And here's the resolution of the C version's worst problem, stated as cleanly as I can put it: C forces one channel to carry two different kinds of information; C++ partitions by lifecycle phase instead of by mechanism. Anything that can go wrong *before* a `Widget` exists — a missing file, a failed allocation — is negotiable. You, the Tier-2 author, choose how to report it: a bool, an `optional`, an `expected`, an exception you let escape. That choice is exactly what changed across this article's C++ examples, and none of the choices were wrong, just more or less honest about what they covered. Anything that can go wrong *after* a `Widget` exists, during teardown of something that already succeeded, is not negotiable. The language closes that channel down to one option: throw and terminate, or don't throw at all. It isn't "expected versus exceptions" as competing mechanisms; it's "before the object exists" versus "after," and the language only enforces the partition on one side of that line.

That's why the C version's ambiguity — does the construction error or the destruction error win? — is structurally impossible to ask in the C++ version. There's never a live construction-phase error and a live destruction-phase error competing for the same object at the same time, because a `Widget` that fails to finish constructing never reaches a state where its destructor runs on the partially-built thing. The problem isn't solved by a smarter merge policy. It's dissolved, because the two kinds of failure were never sharing a channel to begin with.

## When There's No Primitive to Lean On

The obvious objection: this whole story depends on Tier 1 having already written the primitive you need. What about a resource with no `<mutex>`-shaped wrapper — a raw file descriptor from some C API, say? Here's what filling that gap looks like:

```cpp
#include <unistd.h>   // close()
#include <utility>    // std::exchange
#include <cstdlib>    // std::abort

class FdGuard {
public:
    explicit FdGuard(int fd) noexcept : fd_(fd) {}
    FdGuard(const FdGuard&) = delete;
    FdGuard& operator=(const FdGuard&) = delete;
    FdGuard(FdGuard&& other) noexcept : fd_(std::exchange(other.fd_, -1)) {}
    FdGuard& operator=(FdGuard&& other) noexcept {
        if (this != &other) { reset(); fd_ = std::exchange(other.fd_, -1); }
        return *this;
    }
    ~FdGuard() { reset(); }
    int get() const noexcept { return fd_; }

private:
    void reset() noexcept {
        if (fd_ >= 0 && close(fd_) != 0) {
            std::abort(); // no caller, however many layers up, can act on a close() failure
        }
        fd_ = -1;
    }
    int fd_ = -1;
};
```

Once `FdGuard` exists, `Widget` just gets a member of this type and stays Tier 2: trivial destructor, no manual cleanup, same as before. The point isn't that this breaks the model. It's that it *locates* the model precisely. Writing `FdGuard` requires the actual Tier-1 skillset: deleted copy, hand-written move, `noexcept` discipline, an explicit policy for what happens when cleanup fails (here, the same abort-on-invariant-violation idiom the C version could have used, but had to apply by convention rather than have the type system enforce it). Someone steps up a tier, writes the primitive once, and composition resumes underneath it exactly as before. The tiering isn't a claim that everyone stays in Tier 2 forever; it's a claim about where the hard work lives and how rarely each person has to visit it.

## A Question for Engineers

Here's the question I actually want to put to people who write C++ day to day: should "C with stdlib" be adopted on purpose, or at least stop being read as a sign of arrested development?

If the two-tier model holds, the answer is yes, and the implications are concrete, not just attitudinal. Code review's default suspicion should flip. A struct with a trivial destructor, composing `std::mutex` and `std::ifstream` and `std::vector`, isn't the code of someone who hasn't gotten to the interesting parts of C++ yet — it's what correctly-tiered code looks like. The thing that deserves a second look in review is a hand-written destructor, copy constructor, or move constructor, because the question it raises is real: is this author deliberately stepping into Tier 1, with the same discipline `FdGuard` needed — deleted copy, explicit `noexcept`, a stated policy for cleanup failure — or is this a raw resource being managed by hand when a standard primitive already does the job?

It should change how the language gets taught, too. Sequences that spend their first weeks on move semantics, operator overloading, and inheritance before ever showing someone write a `Widget`-shaped struct are teaching Tier 1 first and treating Tier 2 as the reward for finishing your vegetables. That's backwards from how most people who write C++ for a living actually spend their time. If most engineers live in Tier 2, most of their early training should look like Tier 2 — composing primitives, trivial destructors, the Rule of Zero as the default outcome rather than the advanced technique — with Tier 1 introduced specifically as "here's what it takes to author a new primitive," not as a gate you clear before you're allowed to write ordinary application code.

## A Question for Language Designers

And here's the question for the people who steer where the language goes next: should the two-tier model be something C++ designs for on purpose, or is it fine to keep leaving it to convention and outside forks?

Right now it's the latter, by default rather than by decision. There's no compiler flag, no core-language feature, no standard attribute that says "this class is Tier 2, refuse to compile a hand-written destructor here." What exists instead is a pattern of external patches: Herb Sutter's cppfront (Cpp2) adds syntax specifically to make unsafe patterns harder to write. Carbon is a from-scratch successor partly motivated by the same gap. Internal style guides lean on `clang-tidy` checks to catch hand-rolled resource management where a standard primitive would do. Each of these is a tacit admission that the split is real and that the language, as specified, has no way to require it.

There's a sharper, more actionable version of the question, too. Every proposal that reduces the risk of authoring a Tier-1 primitive — guaranteed copy elision, `std::expected`, a fully-standardized `unique_resource` — makes Tier 2 safer without a single Tier-2 author changing anything about how they write code. That's a real, measurable lever, and it suggests a prioritization question the committee could ask explicitly rather than treat as a side effect: does this proposal lower the error surface of writing a primitive? There's a real counter-argument worth taking seriously here — standardizing enforcement this deep risks fracturing an already enormous surface area, and letting Cpp2 and Carbon prove ideas out in the wild before ISO C++ commits to anything is plausibly safer than legislating tier discipline directly, given how much existing code the standard has to stay compatible with. But that's a choice, and right now it isn't being made as one. It's just what's left over when nobody decides.

## Prove Me Wrong

The claim underneath both questions, stated plainly: C++ code that looks like C — structs, free functions, enums, `printf`, no inheritance, no hand-written destructors — isn't unfinished C++. It's what C++ looks like at the application layer once the hard lifecycle problems have been solved exactly once, somewhere else, by someone writing a primitive.

If that's right, you should be able to find production code that violates it cleanly: Tier-2-shaped code that genuinely needs hand-written move semantics for reasons that aren't really "this is secretly a primitive," or a case where the construction/destruction partition described above doesn't hold. If you've got one, I want to see it — because both questions above only have teeth if the model underneath them does.
