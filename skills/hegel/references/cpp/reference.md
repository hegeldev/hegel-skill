# Hegel C++ Reference

## Table of Contents

- [Setup](#setup) — CMake integration, test runner, C++20 requirement
- [Test Structure](#test-structure) — `hegel::test`, `Settings`, `HealthCheck`
- [TestCase Methods](#testcase-methods) — `draw`, `assume`, `note`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, binary, collections, tuples, optional, format, regex, random
- [Combinator Methods](#combinator-methods) — `.map()`, `.filter()`, `.flat_map()`
- [Building Custom Generators](#building-custom-generators) — `compose`, `builds`, `builds_agg`, `field`, `default_generator`, `.override()`
- [C++-Specific Examples](#c-specific-examples) — Dependent generation, randomness, derived generators
- [Gotchas](#gotchas)
- [Stateful Testing](#stateful-testing)

## Setup

hegel-cpp is a static library (not header-only). It requires **C++20** and **CMake 3.14+**.

Add it to your `CMakeLists.txt`:

```cmake
include(FetchContent)
FetchContent_Declare(
    hegel
    GIT_REPOSITORY https://github.com/hegeldev/hegel-cpp.git
    GIT_TAG v0.3.3
)
FetchContent_MakeAvailable(hegel)

target_link_libraries(your_test_target PRIVATE hegel)
```

hegel-cpp is **test-runner agnostic**. `hegel::test(...)` is a regular function — it runs the property, shrinks on failure, and throws a `std::runtime_error` with the minimal counterexample. Any runner that surfaces uncaught exceptions as test failures will work: Google Test, Catch2, doctest, Boost.Test, or a hand-written `main`. Use the existing project's runner; don't introduce GTest just for hegel.

Write whatever test block the runner uses and call `hegel::test(...)` inside it (see the examples below).

If something goes wrong with server installation, see https://hegel.dev/reference/installation.

## Test Structure

### `hegel::test`

Unlike Rust (`#[hegel::test]`) or Go (`hegel.Case`), hegel-cpp does **not** define a custom test macro. Use your project's existing test framework and call `hegel::test` inside the test body with a lambda that takes a `hegel::TestCase&`:

```cpp
#include <hegel/hegel.h>

namespace gs = hegel::generators;

// With Google Test:
TEST(Arithmetic, AdditionCommutes) {
    hegel::test([](hegel::TestCase& tc) {
        auto a = tc.draw(gs::integers<int64_t>());
        auto b = tc.draw(gs::integers<int64_t>());
        assert(a + b == b + a);
    });
}

// With Catch2 / doctest (same structure):
TEST_CASE("addition commutes") {
    hegel::test([](hegel::TestCase& tc) {
        auto a = tc.draw(gs::integers<int64_t>());
        auto b = tc.draw(gs::integers<int64_t>());
        REQUIRE(a + b == b + a);
    });
}
```

Inside the lambda, use whatever assertion mechanism the surrounding framework provides — `ASSERT_*`/`EXPECT_*` for GTest, `REQUIRE`/`CHECK` for Catch2 and doctest, `BOOST_TEST` for Boost.Test, `assert` or a `throw` for a hand-written main. On failure, hegel catches the exception, shrinks, and rethrows the minimal counterexample; the outer runner reports it as a failing test.

With configuration:

```cpp
hegel::test([](hegel::TestCase& tc) {
    // ...
}, hegel::Settings{
    .test_cases = 500,
    .verbosity = hegel::Verbosity::Verbose,
    .seed = 42,
});
```

### Settings

`hegel::Settings` controls test execution. All fields are optional; pass it to `hegel::test` as the second argument.

| Field | Type | Purpose |
|-------|------|---------|
| `test_cases` | `std::optional<uint64_t>` | Number of test cases (default: 100) |
| `verbosity` | `Verbosity` | `Quiet`, `Normal`, `Verbose`, or `Debug` |
| `seed` | `std::optional<uint64_t>` | Fixed seed for reproducibility |
| `derandomize` | `bool` | Use a deterministic seed (default: `false`) |
| `database` | `Database` | `Database::unset()` (default), `Database::disabled()`, or `Database::from_path(path)` |
| `suppress_health_check` | `std::vector<HealthCheck>` | Suppress specific health checks |

### HealthCheck

`HealthCheck` variants (pass to `suppress_health_check`):

- `HealthCheck::FilterTooMuch` — Too many test cases rejected via `assume()`
- `HealthCheck::TooSlow` — Test execution is too slow
- `HealthCheck::TestCasesTooLarge` — Generated test cases are too large
- `HealthCheck::LargeInitialTestCase` — The smallest natural input is very large

```cpp
hegel::test([](hegel::TestCase& tc) {
    // ...
}, hegel::Settings{
    .suppress_health_check = {hegel::HealthCheck::FilterTooMuch},
});
```

## TestCase Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `draw` | `template<class T> T draw(const Generator<T>& gen)` | Draw a value from a generator |
| `assume` | `void assume(bool condition)` | Reject this test case if condition is false |
| `note` | `void note(std::string_view message)` | Record debug info (printed on final counterexample replay) |

### Usage

```cpp
hegel::test([](hegel::TestCase& tc) {
    auto a = tc.draw(gs::integers<int64_t>());
    auto b = tc.draw(gs::integers<int64_t>());
    tc.assume(b != 0);
    tc.note("dividing " + std::to_string(a) + " by " + std::to_string(b));
    auto q = a / b;
    auto r = a % b;
    assert(a == q * b + r);  // or ASSERT_EQ / REQUIRE, depending on runner
});
```

`TestCase` is a non-owning handle. It is not copyable or movable and must not outlive the test callback.

## Generator Reference

All generators live in `hegel::generators`. Idiomatic import:

```cpp
namespace gs = hegel::generators;
```

Generators take their configuration as a designated-initializer struct (one struct per generator). Leave fields unset for defaults.

### Numeric Generators

**`gs::integers<T>(IntegersParams<T> = {})`** — Generate any integer type

Supported types: `int8_t`, `int16_t`, `int32_t`, `int64_t`, `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`, `int`, `long`, `size_t`, etc. The template parameter is required.

```cpp
auto n = tc.draw(gs::integers<int>());
auto bounded = tc.draw(gs::integers<int>({.min_value = 1, .max_value = 100}));
auto nonnegative = tc.draw(gs::integers<int>({.min_value = 0}));
```

Fields:
- `min_value: std::optional<T>` — Inclusive lower bound (default: type min)
- `max_value: std::optional<T>` — Inclusive upper bound (default: type max)

**`gs::floats<T>(FloatsParams<T> = {})`** — Generate `float` or `double`

```cpp
auto f = tc.draw(gs::floats<double>());
auto unit = tc.draw(gs::floats<double>({.min_value = 0.0, .max_value = 1.0}));
auto open_interval = tc.draw(gs::floats<double>({
    .min_value = 0.0, .max_value = 1.0,
    .exclude_min = true, .exclude_max = true,
}));
```

Fields:
- `min_value: std::optional<T>`
- `max_value: std::optional<T>`
- `exclude_min: bool` — Default: `false`
- `exclude_max: bool` — Default: `false`
- `allow_nan: std::optional<bool>` — Default: `true` if unbounded
- `allow_infinity: std::optional<bool>` — Default: `true` if unbounded on that side

### Boolean and Constant Generators

```cpp
auto b = tc.draw(gs::booleans());

auto answer = tc.draw(gs::just(42));
auto greeting = tc.draw(gs::just("hello"));  // Generator<std::string>
```

### Text and Binary Generators

**`gs::text(TextParams = {})`** — Generate `std::string`

```cpp
auto s = tc.draw(gs::text());
auto short_s = tc.draw(gs::text({.min_size = 1, .max_size = 10}));
auto ascii = tc.draw(gs::text({.alphabet = "abcdefghijklmnopqrstuvwxyz"}));
```

Fields:
- `min_size: size_t` (default: 0)
- `max_size: std::optional<size_t>`
- `alphabet: std::optional<std::string>` — Fixed allowed characters
- `codec: std::optional<std::string>` — e.g. `"ascii"`, `"utf-8"`
- `min_codepoint: std::optional<uint32_t>` / `max_codepoint: std::optional<uint32_t>`
- `categories: std::optional<std::vector<std::string>>` — Unicode categories (e.g. `{"Ll", "Lu"}`)
- `exclude_categories: std::optional<std::vector<std::string>>`
- `include_characters: std::optional<std::string>` / `exclude_characters: std::optional<std::string>`

**`gs::characters(CharactersParams = {})`** — Generate a single-character string. Same options as `text` except no size or `alphabet` fields.

**`gs::binary(BinaryParams = {})`** — Generate `std::vector<uint8_t>`

```cpp
auto bytes = tc.draw(gs::binary());
auto sized = tc.draw(gs::binary({.min_size = 1, .max_size = 256}));
```

Fields: `min_size`, `max_size`.

**`gs::from_regex(pattern, fullmatch = false)`** — Generate strings matching a regex

```cpp
auto id = tc.draw(gs::from_regex(R"([A-Z]{3}-[0-9]{3})", /*fullmatch=*/true));
```

### Collection Generators

**`gs::vectors(element_gen, VectorsParams = {})`** — Generate `std::vector<T>`

```cpp
auto v = tc.draw(gs::vectors(gs::integers<int>()));
auto bounded = tc.draw(gs::vectors(gs::integers<int>(),
    {.min_size = 1, .max_size = 10}));
auto distinct = tc.draw(gs::vectors(gs::integers<int>(), {.unique = true}));
```

Fields:
- `min_size: size_t` (default: 0)
- `max_size: std::optional<size_t>` (default: 100)
- `unique: bool` — All elements distinct (default: `false`)

**`gs::sets(element_gen, SetsParams = {})`** — Generate `std::set<T>`. Fields: `min_size`, `max_size`.

**`gs::maps(key_gen, value_gen, MapsParams = {})`** — Generate `std::map<K, V>`. Fields: `min_size`, `max_size`.

```cpp
auto m = tc.draw(gs::maps(gs::text(), gs::integers<int>()));
```

### Tuple Generator

**`gs::tuples(gen_a, gen_b, ...)`** — Generate `std::tuple<A, B, ...>` from 2 or more generators

```cpp
auto pair = tc.draw(gs::tuples(gs::integers<int>(), gs::text()));
auto triple = tc.draw(gs::tuples(
    gs::booleans(), gs::integers<int>(), gs::floats<double>()));
```

### Optional and Variant Generators

```cpp
auto maybe = tc.draw(gs::optional(gs::integers<int>()));  // std::optional<int>

// std::variant<int, std::string, bool>
auto v = tc.draw(gs::variant(gs::integers<int>(), gs::text(), gs::booleans()));
```

### Sampling and Choice Generators

**`gs::sampled_from(elements)`** — Pick one element from a vector or initializer list

```cpp
auto color = tc.draw(gs::sampled_from({"red", "green", "blue"}));
auto digit = tc.draw(gs::sampled_from(std::vector<int>{1, 2, 3, 4, 5}));
```

**`gs::one_of({gen_a, gen_b, ...})`** — Choose between generators of the same type

```cpp
auto n = tc.draw(gs::one_of({
    gs::integers<int>({.min_value = 0, .max_value = 10}),
    gs::integers<int>({.min_value = 100, .max_value = 110}),
}));
```

### Format Generators

```cpp
auto email = tc.draw(gs::emails());
auto url   = tc.draw(gs::urls());
auto host  = tc.draw(gs::domains());                      // DomainsParams{.max_length}
auto ipv4  = tc.draw(gs::ip_addresses({.v = 4}));
auto ipv6  = tc.draw(gs::ip_addresses({.v = 6}));
auto date  = tc.draw(gs::dates());                        // YYYY-MM-DD
auto time  = tc.draw(gs::times());                        // HH:MM:SS
auto dt    = tc.draw(gs::datetimes());
```

### Random Generator

**`gs::randoms(RandomsParams = {})`** — A hegel-controlled RNG compatible with `<random>`

```cpp
#include <random>

hegel::test([](hegel::TestCase& tc) {
    auto rng = tc.draw(gs::randoms());
    std::uniform_real_distribution<double> dist(0.0, 10.0);
    double x = dist(rng);
    assert(x >= 0.0 && x <= 10.0);
});
```

The returned `HegelRandom` satisfies the `UniformRandomBitGenerator` concept and works with every distribution in `<random>`.

Fields:
- `use_true_random: bool` — Default: `false` (artificial mode)

**Default mode (artificial randomness)** routes every random decision through hegel so the shrinker can minimize individual random choices. Best for most code.

**`use_true_random = true`** generates a single shrinkable seed and uses `std::mt19937` locally. Use this when the code under test does rejection sampling or otherwise needs statistically random-looking output — artificial randomness can cause rejection loops to hang.

## Combinator Methods

Every `Generator<T>` has these chainable methods. They return a new generator (value semantics; cheap to copy).

### `.map(f)`

Transform generated values:

```cpp
auto positive_str = gs::integers<uint32_t>({.min_value = 1})
    .map([](uint32_t n) { return std::to_string(n); });
```

`.map()` preserves the underlying schema when the source is schema-backed, so mapped primitives are still generated efficiently server-side.

### `.filter(predicate)`

Keep only values matching a predicate:

```cpp
auto even = gs::integers<int>()
    .filter([](int x) { return x % 2 == 0; });
```

`.filter()` retries up to 3 times, then calls `tc.assume(false)`. Prefer bounds or constructing valid inputs directly.

### `.flat_map(f)`

Dependent generation — use one value to choose the next generator:

```cpp
auto sized_string = gs::integers<size_t>({.min_value = 1, .max_value = 10})
    .flat_map([](size_t len) {
        return gs::text({.min_size = len, .max_size = len});
    });
```

Rarely needed in hegel-cpp — sequential `tc.draw()` calls inside the test body are usually clearer (see C++-Specific Examples).

## Building Custom Generators

### `gs::compose(f)` — Inline custom generator

Build a generator from imperative code. The return type is deduced from the lambda:

```cpp
auto point_gen = gs::compose([](const hegel::TestCase& tc) {
    auto x = tc.draw(gs::floats<double>({.min_value = -100.0, .max_value = 100.0}));
    auto y = tc.draw(gs::floats<double>({.min_value = -100.0, .max_value = 100.0}));
    return std::pair{x, y};
});

auto [x, y] = tc.draw(point_gen);
```

Use a trailing return type if the deduction picks the wrong type: `[](const hegel::TestCase& tc) -> long { ... }`.

### `gs::builds<T>(gen_a, gen_b, ...)` — Construct via constructor

Calls `T(arg1, arg2, ...)` with values drawn from each generator, in order:

```cpp
struct Point {
    Point(double x, double y) : x(x), y(y) {}
    double x, y;
};

auto point = gs::builds<Point>(
    gs::floats<double>({.min_value = 0.0, .max_value = 100.0}),
    gs::floats<double>({.min_value = 0.0, .max_value = 100.0}));
```

### `gs::builds_agg<T>(fields...)` — Construct aggregates by member

Use `gs::field<&T::member>(gen)` to bind a generator to a named field:

```cpp
struct Rectangle { int width; int height; };

auto rect = gs::builds_agg<Rectangle>(
    gs::field<&Rectangle::width>(gs::integers<int>({.min_value = 1, .max_value = 100})),
    gs::field<&Rectangle::height>(gs::integers<int>({.min_value = 1, .max_value = 100})));
```

### `gs::default_generator<T>()` — Auto-derive for structs

Uses [reflect-cpp](https://github.com/getml/reflect-cpp) to inspect struct fields and pick default generators for each field type. Works with primitives, strings, containers, `std::optional`, `std::variant`, `std::tuple`, and nested structs.

```cpp
struct Person {
    std::string name;
    int age;
};

TEST(Person, HasValidAge) {
    hegel::test([](hegel::TestCase& tc) {
        auto p = tc.draw(gs::default_generator<Person>());
        // ...
    });
}
```

### `.override()` — Customize derived fields

`default_generator<T>()` returns a `DerivedGenerator<T>` with an `.override()` method that replaces the default generator for specific fields:

```cpp
auto adult_gen = gs::default_generator<Person>()
    .override(
        gs::field<&Person::age>(gs::integers<int>({.min_value = 18, .max_value = 120})),
        gs::field<&Person::name>(gs::text({.min_size = 1, .max_size = 50})));
```

Unspecified fields keep their defaults. Multiple `.override()` calls can be chained.

## C++-Specific Examples

These examples show C++-specific features. For general property patterns (round-trip, model-based, idempotence, etc.), see the main skill's Property Catalogue.

These examples show the `hegel::test` body only. Wrap it in whatever test block your runner uses.

### Dependent generation with sequential draws

Hegel's imperative style means dependent generation is just sequential code — no `flat_map` needed:

```cpp
hegel::test([](hegel::TestCase& tc) {
    auto v = tc.draw(gs::vectors(gs::integers<int>(), {.min_size = 1}));
    auto idx = tc.draw(gs::integers<size_t>({
        .min_value = 0,
        .max_value = v.size() - 1,
    }));
    (void)v[idx];  // always a valid index
});
```

### Custom type with derived generator

```cpp
struct Config {
    uint32_t max_retries;
    uint64_t timeout_ms;
    std::string name;
};

hegel::test([](hegel::TestCase& tc) {
    auto base = tc.draw(gs::default_generator<Config>());
    auto override_cfg = tc.draw(gs::default_generator<Config>());
    auto merged = merge(base, override_cfg);
    assert(merged.name == override_cfg.name);
});
```

### Testing code that uses randomness

```cpp
// Under test: size_t sample(const std::vector<double>& weights, auto& rng);

hegel::test([](hegel::TestCase& tc) {
    auto weights = tc.draw(gs::vectors(
        gs::floats<double>({.min_value = 0.0, .exclude_min = true}),
        {.min_size = 1}));
    auto rng = tc.draw(gs::randoms());
    auto idx = sample(weights, rng);
    assert(idx < weights.size());
});
```

If the code does rejection sampling and the test hangs with the default mode, switch to `.use_true_random = true`:

```cpp
auto rng = tc.draw(gs::randoms({.use_true_random = true}));
```

### Wrapping arithmetic in test values

When computing test values from generated data, avoid overflow in your *test* code:

```cpp
// BAD — signed overflow is undefined behavior
map.insert({k, k * 10});

// GOOD — compute via unsigned then cast back
auto uk = static_cast<uint64_t>(k);
map.insert({k, static_cast<int64_t>(uk * 10)});

// ALSO GOOD — draw a smaller type and widen
auto k = static_cast<int64_t>(tc.draw(gs::integers<int16_t>()));
auto k_squared = k * k;  // can't overflow int64_t
```

## Gotchas

1. **No `#[hegel::test]`-style macro.** Call `hegel::test(...)` inside whatever test block your project's runner uses (`TEST(...)` for GTest, `TEST_CASE(...)` for Catch2/doctest, `BOOST_AUTO_TEST_CASE` for Boost.Test, etc.). Hegel is runner-agnostic — on failure it throws, and the surrounding runner reports the failure. Use that runner's own assertions inside the lambda.

2. **Designated-initializer params, not builders.** Unlike Rust (`.min_value(1).max_value(100)`) or Go (`hegel.Integers[int](1, 100)`), hegel-cpp generators take a params struct: `gs::integers<int>({.min_value = 1, .max_value = 100})`. Omit fields to get defaults.

3. **Template parameter is required for numeric generators.** `gs::integers()` won't compile — you must write `gs::integers<int>()` or similar. Same for `gs::floats<double>()`.

4. **Add `.hegel/` to `.gitignore`.** Hegel creates a `.hegel/` directory for caching the server binary and storing the failure database.

5. **Float defaults include NaN and infinity** when unbounded. If your code doesn't handle them, pass `.allow_nan = false` and/or `.allow_infinity = false` — but first consider whether the code *should* handle them.

6. **Excessive `assume`/`filter` rejections fail the test.** If `tc.assume()` or `.filter()` rejects too many inputs, hegel gives up (via `HealthCheck::FilterTooMuch`). Restructure generators to produce valid inputs directly.

7. **`note()` only prints on the final replay.** Don't rely on it for progress logging — it only appears when the minimal counterexample is displayed.

8. **Default collection sizes are small.** `gs::vectors(gen)` with no bounds rarely produces 100+ elements. If you need large collections (e.g., to exercise tree traversal at depth), draw the size separately:
   ```cpp
   auto n = tc.draw(gs::integers<size_t>({.max_value = 300}));
   auto keys = tc.draw(gs::vectors(gs::integers<int>(), {.min_size = n}));
   ```

9. **Use `.unique = true` for key generation.** When testing ordered maps or sets, generate unique keys to avoid ambiguity about which value wins:
   ```cpp
   auto keys = tc.draw(gs::vectors(gs::integers<int>(),
       {.max_size = 50, .unique = true}));
   ```

10. **C++20 is required.** The library makes heavy use of concepts, designated initializers, and template pack features. Older standards will not compile.

11. **`TestCase&` is not copyable or movable.** Capture it by reference in nested lambdas or pass it explicitly — don't try to store it.

12. **`target()` is not yet available** in hegel-cpp. It is planned for a future release.

## Stateful Testing

**Stateful (model-based) testing is not yet available in hegel-cpp.** It is planned for a future release. Until then, if you need a state-machine test, either:

- Write the rule loop by hand inside `hegel::test(...)`, drawing a rule choice with `gs::sampled_from(...)` and dispatching, or
- Use hegel in one of the languages where stateful testing is supported (Rust, Python/Hypothesis) for that specific test.

When stateful testing lands in hegel-cpp, this section will be updated with the rule/invariant API.
