# Fun with Coroutines

A hands-on C++20 coroutines course repo. Five problems are solved twice — once with
callbacks and state machines (`no-coroutines/`), once with coroutines (`coroutines/`).
Google Test checks that both paths behave the same.

**Slides:** [`magic-with-coroutines.pptx`](magic-with-coroutines.pptx) — the accompanying course presentation.

## Repository layout

```
fun-with-coroutines/
├── Samples/
│   ├── common/              Shared infrastructure (reactor, pipes, parser, event loop, …)
│   ├── coroutines/          Coroutine solutions + return types
│   ├── no-coroutines/       Callback / state-machine equivalents
│   ├── tests/               Google Test suites (mirrors the demo folders)
│   ├── CMakeLists.txt       CMake build (CLion, ctest)
│   └── Makefile             Command-line build (Linux)
└── magic-with-coroutines.pptx   Course slides
```

### `Samples/common/`

Code shared by both trees: line-protocol helpers, `InputBuffer`, `Reactor`, `Pipe`,
`EventLoop`, tree data for generator demos, and test utilities.

### `Samples/coroutines/return_types/`

Custom coroutine return types used across the demos. Each defines a nested
`promise_type` with a different suspend policy. See
[`Samples/coroutines/return_types/README.md`](Samples/coroutines/return_types/README.md)
for the hook comparison table.

| Return type | Used in |
|-------------|---------|
| `Generator` | Demo 1 — lazy sequences with `co_yield` |
| `ResumableCoroutine` | Demo 2 — caller-driven parser (`resume()` per chunk) |
| `NeverSuspendCoroutine` | Demo 3 — fire-and-forget async work |
| `FinalSuspendCoroutine` | Demo 4 — eager start, frame kept alive across I/O |
| `Task<T>` | Demo 5 — lazy task with `co_return`; also an awaitable for chaining |

## Requirements

- **C++20** compiler with coroutine support (`g++` 13+ or recent `clang++`)
- **Google Test** — fetched automatically by CMake, or install system-wide for the Makefile (`libgtest-dev` on Debian/Ubuntu)
- **Linux or macOS** for the full demo set — demos 4 and 5 use POSIX pipes and a reactor (`select`/`epoll`); they are excluded on Windows in the CMake build

## Installation and run

All build commands run from the `Samples/` directory.

### CLion (CMake)

1. Open **`Samples/`** as the project root in CLion (it picks up `CMakeLists.txt`).
2. Let CMake configure — Google Test is downloaded automatically via `FetchContent`.
3. Build the **`fun_with_coroutines_test`** target (output binary: `fun_with_coroutines`).
4. Run tests from the gutter icons next to each `TEST_F`, or run **ctest** from the CMake tool window.

Interactive calculator REPL (Linux/macOS only):

```bash
./cmake-build-debug/bin/calc_repl
```

### Command line (Makefile)

The [`Samples/Makefile`](Samples/Makefile) builds on **Linux** with `g++` and a system Google Test install.

Install dependencies (Debian/Ubuntu example):

```bash
sudo apt install g++ libgtest-dev
```

Build and run:

```bash
cd Samples
make          # builds bin/fun_with_coroutines and bin/calc_repl
make test     # runs all Google Tests
make clean    # removes build/ and bin/
```

Run the calculator REPL:

```bash
./bin/calc_repl
```

### Command line (CMake)

```bash
cd Samples
cmake -B cmake-build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build cmake-build-debug
ctest --test-dir cmake-build-debug
```

## The five demos

Each demo lives in matching folders under `coroutines/`, `no-coroutines/`, and
`tests/`. Side-by-side diffs are the intended way to teach the material.

### 1. Generator — `01-generator/`

**Problem:** Produce a lazy sequence of values while preserving state between steps.

**Highlights:**
- Fibonacci without a hand-written phase enum and switch
- Tree DFS (preorder/inorder) with recursion and `co_yield` instead of an explicit stack
- Return type: `Generator` (`initial_suspend` / `final_suspend`: always / always)

**Keywords:** `co_yield`, `co_return`

### 2. Parser — `02-parser/`

**Problem:** Parse a line-oriented key/value protocol when data arrives in arbitrary
chunk sizes (push model).

**Protocol:** `SET <key> <value>` and `GET <key>` (Redis-like, minimal framing).

**Highlights:**
- Driver pushes chunks; parser pulls complete words via `co_await`
- `InputBuffer` handles word boundaries across partial reads
- Return type: `ResumableCoroutine` — lazy start, caller calls `resume()` per chunk

**Keywords:** `co_await`, `co_return` (void)

### 3. Async work — `03-async-work/`

**Problem:** Offload slow work to a background thread and continue on the calling thread.

**Highlights:**
- `AsyncWork` — resumes on the worker thread (intentionally wrong affinity)
- `AsyncWorkMarshalled` — posts resume to an `EventLoop` (correct pattern)
- Linear `co_await` flow replaces nested completion callbacks
- Return type: `NeverSuspendCoroutine` (never / never)

**Keywords:** `co_await`

### 4. Async I/O — `04-async-io/`

**Problem:** Non-blocking read/write over pipes with partial transfers and a
multi-step request/response session, all on one thread via a reactor.

**Highlights:**
- Calculator client and server over a pair of pipes
- Line-delimited expressions (`2+2\n`) evaluated by the server
- `AsyncBuffer` awaitable hides `EAGAIN` retry and offset tracking
- Client/server coroutines read like synchronous code inside `while` loops
- Return type: `FinalSuspendCoroutine` (never / always)

**Keywords:** `co_await`, `co_return` (void)

### 5. Task composition — `05-task-composition/`

**Problem:** Chain async steps and return a final value to synchronous code.

**Highlights:**
- `TaskChain` — three `Task<int>` coroutines chained with `co_await`; symmetric
  transfer via `FinalAwaiter`
- `SolveQuiz` / `CalculatorBatch` — builds on Demo 4; sends a fixed quiz sequentially,
  accumulates integer answers, returns `Task<std::vector<int>>`
- `sync_wait` / `start()` + `result()` for top-level entry from non-coroutine code
- Return type: `Task<T>` (always / always + `return_value`)

**Keywords:** `co_await`, `co_return`

## Suggested reading order

1. Pick a demo in `no-coroutines/` — feel the callback/state-machine cost
2. Read the matching `coroutines/` solution
3. Run the tests under `tests/<demo>/`
4. Inspect the return type in `coroutines/return_types/`
