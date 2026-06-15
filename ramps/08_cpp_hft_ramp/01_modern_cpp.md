# Stage 01 — Modern C++

**Status:** not started
**Estimated time:** 5–7 days
**Type:** additive start — this codebase grows through all 7 stages
**Depends on:** basic C++ syntax (classes, loops, functions)

The job description says "strong proficiency in C++ with 5+ years." That means C++17 idioms — not C-style C++. RAII, move semantics, smart pointers, templates, and the standard library. This stage gets you to the level where you write the language the way a senior engineer at an HFT firm would read it and respect it.

The project folder is `trading_engine/`. It starts here and every subsequent stage adds to it.

---

## What You're Building

A compiled project with:
- A proper CMake build setup
- A `core/` module with RAII-safe resource wrappers
- A `types/` module with the primitive types used throughout the engine (Price, Quantity, OrderId, Side)
- A `utils/` module with logging and timing utilities
- A main entry point that runs and proves the project compiles

---

## Concepts This Stage Teaches

```yaml
concepts:
  RAII:
    what: >
      Resource Acquisition Is Initialization. Objects acquire resources in constructors
      and release them in destructors. When an object goes out of scope, cleanup happens
      automatically — no manual delete, no cleanup in a catch block.
    why: >
      HFT code has tight resource budgets — file handles, sockets, shared memory.
      RAII is how you ensure nothing leaks even when exceptions fire. It's the C++
      answer to C#'s using/IDisposable, but baked into the language itself.
    pattern: >
      class FileHandle {
        FILE* f;
      public:
        FileHandle(const char* path) : f(fopen(path, "r")) {
          if (!f) throw std::runtime_error("cannot open file");
        }
        ~FileHandle() { if (f) fclose(f); }
        // delete copy, allow move
      };

  Move semantics:
    what: >
      Move constructor and move assignment operator transfer ownership of resources
      rather than copying them. std::move() marks an object as "safe to steal from."
    why: >
      Copies are expensive. Moving a vector of a million orders is O(1). Copying is O(n).
      HFT code passes large data structures constantly — you must understand move to avoid
      accidental copies on hot paths.
    gotcha: >
      After std::move(), the source object is in a valid but unspecified state.
      Never use a moved-from object except to reassign or destroy it.

  Smart pointers:
    what: >
      std::unique_ptr — sole ownership, zero overhead, cannot be copied.
      std::shared_ptr — shared ownership, reference counted, small overhead.
      std::weak_ptr — non-owning observer, avoids cycles with shared_ptr.
    why: >
      Raw pointers in HFT code are not absent — they appear in hot paths for performance.
      But ownership semantics are expressed with smart pointers. You need both.
    rule: >
      Use unique_ptr by default. Reach for shared_ptr only when shared ownership is
      genuinely necessary. Never use raw new/delete except in performance-critical allocators.

  Templates and type safety:
    what: >
      Templates generate code at compile time for each type. Function templates,
      class templates, and template specializations.
    why: >
      HFT type systems are deeply templated. A Price<Instrument> is different from
      a Price<Currency>. Catching these mismatches at compile time, not runtime,
      is a core design principle.
    pattern: >
      template<typename T>
      T clamp(T val, T lo, T hi) {
        return val < lo ? lo : (val > hi ? hi : val);
      }

  constexpr and compile-time computation:
    what: >
      constexpr tells the compiler to evaluate an expression at compile time.
      constexpr functions can be used in both compile-time and runtime contexts.
    why: >
      Constants used in HFT (tick sizes, lot sizes, instrument IDs) should be
      compile-time values — they're free at runtime and prevent accidental mutation.

  std::optional, std::variant, std::span:
    what: >
      optional<T> — a value that may or may not exist. Better than nullable pointers.
      variant<A, B, C> — a type-safe union. One of A, B, or C at any time.
      span<T> — a non-owning view over a contiguous range. No copy, no allocation.
    why: >
      These are the building blocks of expressive modern C++ interfaces — exactly what
      the JD means by "user-friendly, expressive and efficient trading interfaces."

  The Rule of Five:
    what: >
      If you define any of destructor / copy constructor / copy assignment /
      move constructor / move assignment — you probably need to define all five.
    why: >
      Getting this wrong causes double-frees, shallow copies, and silent resource leaks.
      Any class that manages a resource (memory, file handle, socket) must follow this rule.
```

---

## Project Setup

```bash
mkdir trading_engine && cd trading_engine

# Structure
trading_engine/
├── CMakeLists.txt
├── src/
│   ├── main.cpp
│   ├── core/
│   │   ├── CMakeLists.txt
│   │   └── timer.hpp          ← high-resolution timing utility
│   ├── types/
│   │   ├── CMakeLists.txt
│   │   └── primitives.hpp     ← Price, Quantity, OrderId, Side
│   └── utils/
│       ├── CMakeLists.txt
│       └── logger.hpp         ← simple timestamped logger
└── build/                     ← git-ignored

# Build
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
./trading_engine
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Set up CMake project"
    detail: >
      Root CMakeLists.txt:
        cmake_minimum_required(VERSION 3.20)
        project(trading_engine CXX)
        set(CMAKE_CXX_STANDARD 17)
        set(CMAKE_CXX_STANDARD_REQUIRED ON)
        add_subdirectory(src/core)
        add_subdirectory(src/types)
        add_subdirectory(src/utils)
        add_executable(trading_engine src/main.cpp)
        target_link_libraries(trading_engine PRIVATE core types utils)

      Every stage will add modules to this root CMakeLists.

  2:
    task: "Define trading primitives in types/primitives.hpp"
    detail: >
      // Strong typedef pattern — prevents Price being passed where Quantity is expected
      struct Price    { int64_t value; };   // fixed-point, e.g. price * 1e6
      struct Quantity { int64_t value; };
      struct OrderId  { uint64_t value; };

      enum class Side { Buy, Sell };
      enum class OrderType { Limit, Market, Cancel };

      // Comparison operators for Price so it works in maps and priority queues
      inline bool operator<(Price a, Price b)  { return a.value < b.value; }
      inline bool operator==(Price a, Price b) { return a.value == b.value; }
      // ... etc

      Strong typedefs catch bugs at compile time. Passing a Quantity where an OrderId
      is expected becomes a compile error, not a runtime disaster.

  3:
    task: "Write a high-resolution timer in core/timer.hpp"
    detail: >
      #include <chrono>

      class Timer {
        using Clock = std::chrono::high_resolution_clock;
        Clock::time_point start_;
      public:
        Timer() : start_(Clock::now()) {}
        void reset() { start_ = Clock::now(); }
        int64_t elapsed_ns() const {
          return std::chrono::duration_cast<std::chrono::nanoseconds>(
            Clock::now() - start_).count();
        }
      };

      This is your performance measurement tool for every subsequent stage.

  4:
    task: "Write a simple logger in utils/logger.hpp"
    detail: >
      A struct Logger with a log(std::string_view) method that prefixes a timestamp.
      Use std::chrono for the timestamp, std::cout for output.
      Make it constructible with a name: Logger("OrderBook"), Logger("Strategy").
      This is for development visibility — not for hot paths.

  5:
    task: "Write an RAII resource wrapper"
    detail: >
      In core/, write a class MappedBuffer that wraps a raw byte buffer.
      Constructor: allocates N bytes with new[].
      Destructor: deallocates with delete[].
      Move constructor: steals the pointer, nulls the source.
      Copy constructor: deleted.

      This is the RAII + Rule of Five exercise. You'll use a similar pattern
      for lock-free ring buffers in Stage 02.

  6:
    task: "Write a template utility and use constexpr"
    detail: >
      In utils/, write:
        template<typename T> T clamp(T val, T lo, T hi)
        constexpr int64_t to_fixed(double price, int decimals)  // e.g. to_fixed(100.25, 2) = 10025

      Use to_fixed in primitives.hpp to convert human-readable prices to the fixed-point
      integer representation used internally.

  7:
    task: "Use std::optional in a real function"
    detail: >
      In types/, write:
        std::optional<Price> parse_price(const std::string& s)

      Returns std::nullopt if the string isn't a valid price.
      Call it from main.cpp and handle both the present and absent cases.
      This is the pattern used everywhere order parsing can fail.

  8:
    task: "main.cpp integration check"
    detail: >
      main.cpp should:
        1. Construct a Logger and log "engine started"
        2. Construct a Timer
        3. Parse a price string using parse_price, log the result
        4. Construct a MappedBuffer of 1024 bytes
        5. Log elapsed nanoseconds from the timer
        6. Print "all checks passed"

      If this compiles and runs, Stage 01 is done.
```

---

## Done When

```yaml
done_when:
  - CMake project compiles with -Wall -Wextra and zero warnings
  - You can explain RAII to someone in 30 seconds using your MappedBuffer as the example
  - You can explain move semantics and show one place they prevent a copy
  - You understand the Rule of Five and can apply it without looking it up
  - strong typedef pattern (Price, Quantity, OrderId) is in place and used correctly
  - std::optional is used in at least one function
  - Timer gives nanosecond resolution and you've measured something with it
```

---

## Interview Signal

```yaml
what_this_demonstrates:
  - You write modern C++, not C-with-classes
  - You use the type system to prevent bugs, not just describe data
  - You know RAII — the most important C++ idiom in systems programming
  - You can set up a CMake project from scratch — baseline expectation at any HFT firm
```

---

## Open Questions

```yaml
open_questions:
  - What is the actual overhead of shared_ptr vs unique_ptr in a hot loop?
    (Benchmark this in Stage 02 — make a note now)
  - What is std::pmr (polymorphic memory resources)? Why do HFT systems care?
  - What does __attribute__((packed)) do to a struct? When would you use it?
  - What is the difference between inline and constexpr functions?
```
