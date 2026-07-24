# C with stdlib: A Two-Tier Model of C++

You've seen this code before. It uses `std::vector`, `std::string`, perhaps a `std::mutex` or a `std::thread`, but structurally it reads like C. It favors enums over class hierarchies, structs over elaborate classes, and free functions over methods. It may even use `printf` and `fprintf` instead of `std::cout`. It compiles, runs correctly, and uses little of what people usually call C++ idiom beyond standard-library containers and smart pointers.

Call it C with stdlib. You will also hear "C with STL," which is more common but less accurate. `<mutex>` and `<thread>` were never part of the STL, which historically meant containers, iterators, and algorithms. This style covers more ground.

The usual reading is that C with stdlib is C++ half-adopted. It is treated as a stop on the way to "real" C++, or as evidence that the author never got beyond the parts of the language that resemble C. I think that is backwards.

Most application code should not implement resource ownership, move semantics, or exception-safe cleanup. It should compose types that already implement those things. The hard work still exists, but it is concentrated in a smaller layer of resource-owning primitives.

That leads to two questions. Should engineers adopt this style deliberately, or at least stop treating it as unfinished C++? And should language designers make the boundary between ordinary application code and resource-owning primitives easier to express and enforce?

One small example can take us through both questions.

## Building a Widget in C

Suppose a `Widget` owns three resources: a file, a heap buffer, and a mutex. In C, construction can fail partway through. When it does, the program must release exactly the resources that were acquired.

The naive implementation puts cleanup inside every failing branch. That duplicates teardown logic across several exit paths. The standard alternative is `goto cleanup`: one label and one shared teardown path.

Let us also allow destruction to report that closing the file failed:

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

    if (w->lock_initialized) mtx_destroy(&w->lock); // void: no failure to report

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

    if (mtx_init(&w->lock, mtx_plain) != thrd_success) {
        status = WIDGET_ERR_LOCK;
        goto cleanup;
    }
    w->lock_initialized = 1;

    *out = w;
    return WIDGET_OK;

cleanup:
    widget_destroy(w); // its return value is discarded
    return status;
}
```

This is disciplined C. `goto cleanup` is the conventional pattern for this problem. Even so, the code exposes three costs.

First, `lock_initialized` exists because `mtx_t` has no empty state. A pointer can be `NULL`, but a mutex cannot say "not initialized" on its own. The struct needs another field to record how far construction progressed. Other resource types need different sentinels, so the bookkeeping is not even uniform.

Second, the cleanup path can encounter two errors but has one return value. If `mtx_init` fails, `status` is already `WIDGET_ERR_LOCK`. Then `widget_destroy` runs and `fclose` may also fail. The code discards that second failure. This may be the right policy, but the language does not specify it. A caller cannot discover the policy from the function signature.

Third, `mtx_destroy` returns `void`. Some APIs do not let cleanup report failure at all. C therefore leaves each resource API and each caller to decide what cleanup failure means.

None of this is a criticism of the C author. Given what the language provides, this is close to the standard way to write the code. The interesting question is what changes when we write the same `Widget` in C++.

## The Same Widget in C++

Here is one C++ version:

```cpp
#include <fstream>
#include <mutex>
#include <optional>
#include <string>
#include <utility>
#include <vector>

struct Widget {
    std::ifstream file;
    std::vector<char> buffer;
    std::mutex lock;
};

std::optional<Widget> make_widget(const std::string& path) {
    std::ifstream file(path);
    if (!file) return std::nullopt;

    std::vector<char> buffer(4096);

    return std::optional<Widget>(
        std::in_place,
        std::move(file),
        std::move(buffer)
    );
}
```

`Widget` is an aggregate. It has no user-written constructor, destructor, copy constructor, move constructor, or assignment operator. This is the Rule of Zero in a particularly plain form.

Two C++ guarantees make the return statement work. First, C++20 allows an aggregate to be initialized from a parenthesized list. The in-place constructor of `optional` can therefore initialize `Widget` from the file and buffer. The omitted `lock` member is value-initialized, which default-constructs the mutex.

Second, the function returns a prvalue of its exact return type: `std::optional<Widget>`. Since C++17, that wrapper is constructed directly in the caller's storage. `std::mutex` is neither copyable nor movable, so `Widget` is not movable either. Constructing it in place inside the final `optional` means that no separate `Widget` ever needs to move.

If opening the file fails, the function returns an empty `optional`. If allocating the local vector fails, the local file is destroyed and `Widget` construction never begins. Once construction does begin, the file and buffer move into place and the mutex is default-constructed. No partially built `Widget` can escape to the caller.

The sentinel field and shared cleanup label are gone. Their work has not vanished. It moved into the resource-owning types.

## The Two-Tier Model

This suggests a two-tier model for C++.

**Tier 1** contains resource-owning primitives such as `std::mutex`, `std::ifstream`, `std::vector`, and types written in the same spirit. Their authors deal with ownership, moves, exception safety, and cleanup policy.

**Tier 2** is application code. It combines Tier-1 types into structs and functions. Its special member functions can usually be generated by the compiler, even though the generated destructor may do substantial work through its members.

That last distinction matters. A `Widget` destructor is not *trivial* in the formal C++ sense. Its members have non-trivial destructors. The useful property is that no one had to write `~Widget()` by hand.

This is the Rule of Zero as a consequence of layering, not merely a style preference. When every member already owns and releases its resource, the containing application type should not repeat that work.

It is also what makes "C with stdlib" a coherent style. The application layer can remain structurally simple: structs, enums, free functions, and direct control flow. C++-specific machinery is present, but application authors spend it rather than reimplement it.

If this model is accurate, difficult C++ should cluster at the seams between the tiers. Those are the places where someone must create a new resource-owning primitive or reconcile two primitives with different contracts.

The `Widget` example already contains such a seam.

## A Seam: Two Failure Modes, One Explicit Channel

`make_widget` returns `std::optional<Widget>`. Its return type tells the caller that producing no `Widget` is a normal outcome, but it does not say why. Only the implementation or documentation reveals that an empty `optional` means the file could not be opened.

The signature also does not reveal that constructing `buffer` can throw `std::bad_alloc`. C++ function signatures do not list the exceptions that may escape:

```cpp
if (auto w = make_widget(path)) {
    // use *w
} else {
    // the file could not be opened
}
```

This caller handles the empty result but may still receive an allocation exception. Discovering that possibility requires knowledge of `vector`, documentation for `make_widget`, or a look at the implementation.

The mismatch comes from the primitives being composed. `ifstream` reports an ordinary open failure through stream state, while `vector` reports allocation failure with an exception. `make_widget` inherits both conventions.

That may be a reasonable API. Failure to open a file and failure to allocate memory do not necessarily need the same reporting mechanism. If the application does want one explicit channel, C++23's `std::expected` can provide it:

```cpp
#include <expected>
#include <new>

enum class WidgetError { Open, Alloc };

std::expected<Widget, WidgetError>
make_widget(const std::string& path) noexcept {
    try {
        std::ifstream file(path);
        if (!file) return std::unexpected(WidgetError::Open);

        std::vector<char> buffer(4096);

        return std::expected<Widget, WidgetError>(
            std::in_place,
            std::move(file),
            std::move(buffer)
        );
    } catch (const std::bad_alloc&) {
        return std::unexpected(WidgetError::Alloc);
    }
}
```

`Widget` is again initialized as an aggregate inside the wrapper, so its mutex still does not need to move. The factory now translates two underlying conventions into one explicit error type.

The first benefit is documentation. `std::expected<Widget, WidgetError>` tells the caller that failure is part of the ordinary contract and names the type that describes it. `WidgetError` then lists `Open` and `Alloc`. An exception-based API cannot express its possible exception types in the function signature. The caller must get that information from documentation, implementation details, or convention.

The second benefit is that the two outcomes remain distinct, both semantically and operationally. A missing file may be an anticipated, recoverable result that means "use the default" or "try the next location," while real memory exhaustion is closer to an emergency. A caller can handle `WidgetError::Open` locally while escalating `WidgetError::Alloc`. In a hot path where "file missing" is routine, returning `WidgetError::Open` also keeps that outcome on an ordinary branch instead of paying for exception construction and stack unwinding each time.

The earlier `optional` version already kept open failure off the exception path, so `expected` is not a performance improvement over `optional` here. It preserves that useful property while making the reason for failure explicit. The performance contrast is between an explicit return channel and an API that throws a common, locally handled error.

The `noexcept` adds another promise: no exception will leave `make_widget`. It is not the error-handling mechanism. The `try` and `catch` translate allocation failure into `WidgetError::Alloc`. If a future edit introduces a different exception and the catch clauses are not updated, the program calls `std::terminate`.

That is a strong choice. It can make a broken contract fail quickly, but it can also turn an overlooked exception into process termination. A production API may reasonably omit `noexcept` unless it can maintain the guarantee.

Explicit errors also have a cost. When a failure should pass through several layers that have no reason to inspect it, each intermediate return type must carry the error. Exceptions can cross those layers without changing every signature. Their contracts are less visible, but their propagation is less intrusive.

Neither mechanism wins everywhere. The point is that a Tier-2 boundary such as `make_widget` must reconcile the contracts of the primitives it composes. Choosing `expected` makes that reconciliation visible, preserves the distinction between anticipated failure and emergency, and keeps routine failure out of exception unwinding.

## What Happens When Teardown Fails?

The C version exposed another seam: construction failed, cleanup also failed, and one `WidgetStatus` could not report both.

C++ does not make this combination impossible. Suppose an aggregate has already constructed a file-owning member when the construction of a later member throws. The file member is destroyed. The complete aggregate's destructor does not run because the aggregate never finished construction, but teardown of its completed members still happens.

The current `Widget` example does not take that exact path. Its buffer is allocated as a local before in-place construction begins. If that allocation fails, the local file is destroyed instead. In both cases, the relevant resource-owning type controls cleanup.

What C++ changes is where the cleanup policy lives.

A destructor with no explicit exception specification gets one based on the destructors of its bases and members. Resource-owning library types are normally designed with non-throwing destructors. If an exception tries to escape a non-throwing destructor, the program calls `std::terminate`.

A destructor can explicitly allow exceptions with `noexcept(false)`. Such an exception can propagate during ordinary execution. If it escapes while another exception is already unwinding the stack, the program still calls `std::terminate`. C++ has no general mechanism for propagating two active exceptions at once.

Suppose construction of a later member fails after a file member has been built, and closing that file also fails. The file wrapper has already chosen what happens next. It may contain or discard the close failure, allowing the original exception to continue. If its destructor throws during unwinding, the program terminates. The factory does not invent a merge policy because cleanup does not return to the factory through a second status value.

This is less magical than saying C++ solved destruction failure. It centralized the decision. Each Tier-1 type defines its cleanup behavior once, and every Tier-2 user inherits that policy.

Standard stream teardown illustrates the tradeoff. A stream destructor does not give calling code a way to observe a close failure. If an application must know that buffered output reached its destination, it should call an explicit operation such as `close` or `flush` and inspect the stream state. The destructor remains the fallback that prevents a leak; it is not always the place to report operational success.

The corrected comparison is therefore:

- C application code often performs cleanup itself and must choose how to combine failures at each call site.
- C++ resource-owning types perform cleanup and choose the failure policy once for all of their users.

RAII removes repeated bookkeeping. It does not make every cleanup failure recoverable, observable, or harmless.

## When There Is No Primitive to Use

The model depends on Tier 1 already containing the resource wrapper you need. What happens when an API gives you a raw file descriptor with no suitable wrapper?

Someone has to write one:

```cpp
#include <cstdlib>
#include <unistd.h>
#include <utility>

class FdGuard {
public:
    explicit FdGuard(int fd) noexcept : fd_(fd) {}

    FdGuard(const FdGuard&) = delete;
    FdGuard& operator=(const FdGuard&) = delete;

    FdGuard(FdGuard&& other) noexcept
        : fd_(std::exchange(other.fd_, -1)) {}

    FdGuard& operator=(FdGuard&& other) noexcept {
        if (this != &other) {
            reset();
            fd_ = std::exchange(other.fd_, -1);
        }
        return *this;
    }

    ~FdGuard() noexcept { reset(); }

    int get() const noexcept { return fd_; }

private:
    void reset() noexcept {
        if (fd_ >= 0 && close(fd_) != 0) {
            std::abort(); // this wrapper deliberately chooses fail-fast cleanup
        }
        fd_ = -1;
    }

    int fd_ = -1;
};
```

This is Tier-1 work. The author must define copy and move behavior, maintain an empty state, preserve `noexcept`, and choose what cleanup failure means.

The abort policy is deliberately severe. Another wrapper might log the error, expose an explicit checked-close operation, or store a diagnostic elsewhere. The point is not that aborting is universally correct. The point is that the wrapper makes the choice once.

Once `FdGuard` exists, an application type can hold it as a member and remain in Tier 2. Its compiler-generated destructor is non-trivial because it invokes `FdGuard::~FdGuard`, but the application author writes no cleanup code.

The two-tier model does not claim that everyone stays in Tier 2 forever. It identifies where the hard work belongs and lets most code avoid repeating it.

## A Question for Engineers

Should C with stdlib be adopted deliberately, or at least stop being treated as arrested development?

If the two-tier model holds, yes. Code review should not distrust a simple struct merely because it lacks custom special member functions. A `Widget` that composes `std::mutex`, `std::ifstream`, and `std::vector` is using C++ precisely by allowing those types to manage their resources.

A user-written destructor, copy constructor, or move constructor deserves closer attention. It may be necessary because the author is creating a new Tier-1 primitive. If so, review should ask the same questions raised by `FdGuard`: What owns the resource? What is the empty state? Is copying forbidden? Are moves safe and correctly marked `noexcept`? What happens when cleanup fails?

If the type is not a new primitive, hand-written resource management may be a sign that an existing wrapper should be used instead.

The model should also affect how C++ is taught. Courses often introduce inheritance, operator overloading, and hand-written move operations before showing how far someone can get by composing library types.

Most engineers spend more time in Tier 2. Early teaching should reflect that. Start with structs, value composition, standard containers, smart pointers, and the Rule of Zero. Introduce custom moves and destructors when students are ready to author a resource-owning primitive, not as an entrance exam for ordinary C++.

## A Question for Language Designers

Should C++ make this two-tier model easier to express?

Today, the boundary is mostly a convention. The language can generate special member functions, and tools can warn about some unsafe ownership patterns, but there is no general way to declare, "This is an application-layer type; reject user-written ownership machinery here."

Projects such as [cppfront](https://github.com/hsutter/cppfront) explore a safer surface that still uses the C++ ecosystem. [Carbon](https://github.com/carbon-language/carbon-lang) takes a different approach: an experimental successor language with C++ interoperability.

Tools such as [`clang-tidy`](https://clang.llvm.org/extra/clang-tidy/) enforce parts of the C++ Core Guidelines in existing code. None of these projects proves the two-tier model, but all reflect demand for safer defaults and more enforceable constraints.

The standard can also improve Tier 2 indirectly. In-place aggregate construction and guaranteed copy elision let the example construct a non-movable `Widget` directly inside the returned wrapper. `std::expected` gives primitive authors and application authors another way to state failure contracts. Every improvement that makes resource-owning types safer or easier to compose benefits the application layer without requiring it to manage resources directly.

There is a serious counterargument. C++ has an enormous existing surface and a large compatibility burden. A new enforcement mechanism could split the language further or reject established code with valid reasons for custom special members. It may be better for external tools and experimental languages to test these ideas before ISO C++ adopts them.

That is still a design choice worth naming. The current boundary exists whether the standard formalizes it or leaves it to libraries, tools, and style guides.

## Prove Me Wrong

The claim is simple: C++ code that resembles C is not necessarily unfinished C++. Structs, free functions, enums, `printf`, and an absence of hand-written destructors can be exactly what the application layer should look like.

The hard lifecycle problems have not disappeared. They have been solved somewhere else by the authors of resource-owning primitives. Application code gets simpler because it composes those solutions.

If the model is wrong, production code should reveal where. Find a Tier-2-shaped type that genuinely needs hand-written move semantics for reasons other than being a resource-owning primitive in disguise. Or find a seam that the two tiers explain badly.

I want to see those examples. The engineering and language-design questions only matter if the model underneath them survives contact with real code.
