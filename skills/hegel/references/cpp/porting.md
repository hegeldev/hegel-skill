# Porting C++ PBT Libraries to Hegel

## From RapidCheck

[RapidCheck](https://github.com/emil-e/rapidcheck) is the most common C++ PBT library. The main differences:

- RapidCheck uses macros (`RC_GTEST_PROP`, `RC_BOOST_PROP`, `rc::check`) that infer generators from function parameter types or from declared `rc::gen::*` values. Hegel is runner-agnostic: you call `hegel::test(...)` with a lambda inside whatever test block your project already uses, and draws happen imperatively via `tc.draw(...)`.
- RapidCheck shrinks in-process; hegel delegates shrinking to a server.
- RapidCheck has `RC_ASSERT`, `RC_PRE`, `RC_CLASSIFY`; hegel uses your framework's native assertions, `tc.assume`, and `tc.note`.
- RapidCheck expresses custom arbitraries by specializing `rc::Arbitrary<T>`; hegel uses `gs::default_generator<T>()` (via reflect-cpp) or `gs::builds<T>`/`gs::builds_agg<T>` for custom construction.

### Test Structure

RapidCheck with the GTest integration:

```cpp
#include <rapidcheck.h>
#include <rapidcheck/gtest.h>

RC_GTEST_PROP(Arithmetic, AdditionCommutes, (int a, int b)) {
    RC_ASSERT(a + b == b + a);
}
```

Hegel (example shown with GTest, but any runner works — see the reference's Setup section):

```cpp
#include <hegel/hegel.h>
#include <gtest/gtest.h>

namespace gs = hegel::generators;

TEST(Arithmetic, AdditionCommutes) {
    hegel::test([](hegel::TestCase& tc) {
        auto a = tc.draw(gs::integers<int>());
        auto b = tc.draw(gs::integers<int>());
        ASSERT_EQ(a + b, b + a);
    });
}
```

Keep the project's existing test runner — if you were using `RC_BOOST_PROP` with Boost.Test, keep Boost.Test and call `hegel::test(...)` from inside a `BOOST_AUTO_TEST_CASE`. Consider whether any bounds in the RapidCheck test are justified. If the property is about addition on `int`, test the full `int` range unless the function's contract says otherwise.

### Generator Mapping

| RapidCheck | Hegel |
|------------|-------|
| `rc::gen::arbitrary<int>()` | `gs::integers<int>()` |
| `rc::gen::inRange(lo, hi)` | `gs::integers<int>({.min_value = lo, .max_value = hi - 1})` *(inRange is half-open)* |
| `rc::gen::inRange<int64_t>(lo, hi)` | `gs::integers<int64_t>({.min_value = lo, .max_value = hi - 1})` |
| `rc::gen::positive<int>()` | `gs::integers<int>({.min_value = 1})` |
| `rc::gen::nonNegative<int>()` | `gs::integers<int>({.min_value = 0})` |
| `rc::gen::arbitrary<double>()` | `gs::floats<double>()` |
| `rc::gen::arbitrary<bool>()` | `gs::booleans()` |
| `rc::gen::arbitrary<std::string>()` | `gs::text()` |
| `rc::gen::character<char>()` | `gs::characters()` *(returns a one-codepoint `std::string`, not `char`; use `.map([](std::string s) { return s[0]; })` with `.max_codepoint = 127` if you need an ASCII `char`)* |
| `rc::gen::string<std::string>()` | `gs::text()` |
| `rc::gen::container<std::vector<T>>(gen)` | `gs::vectors(gen)` |
| `rc::gen::container<std::vector<T>>(n, gen)` | `gs::vectors(gen, {.min_size = n, .max_size = n})` |
| `rc::gen::container<std::set<T>>(gen)` | `gs::sets(gen)` |
| `rc::gen::container<std::map<K,V>>(kgen, vgen)` | `gs::maps(kgen, vgen)` |
| `rc::gen::unique<std::vector<T>>(gen)` | `gs::vectors(gen, {.unique = true})` |
| `rc::gen::tuple(g1, g2)` | `gs::tuples(g1, g2)` |
| `rc::gen::pair(g1, g2)` | `gs::tuples(g1, g2)` *(returns `std::tuple`, not `std::pair`)* |
| `rc::gen::just(x)` | `gs::just(x)` |
| `rc::gen::element(a, b, c)` | `gs::sampled_from({a, b, c})` |
| `rc::gen::elementOf(container)` | `gs::sampled_from(container)` |
| `rc::gen::oneOf(g1, g2)` | `gs::one_of({g1, g2})` |
| `rc::gen::maybe(gen)` | `gs::optional(gen)` |
| `rc::gen::map(gen, fn)` / `gen.map(fn)` | `gen.map(fn)` |
| `rc::gen::mapcat(gen, fn)` | `gen.flat_map(fn)` |
| `rc::gen::suchThat(gen, pred)` | `gen.filter(pred)` |
| `rc::gen::exec(lambda)` | `gs::compose(lambda)` |
| Custom `rc::Arbitrary<T>` specialization | `gs::default_generator<T>()` or `gs::builds<T>` / `gs::builds_agg<T>` |

RapidCheck has no built-in regex, format (emails/URLs/dates), or RNG-controlled generator — you had to build those manually. Use hegel's `gs::from_regex`, `gs::emails`, `gs::urls`, `gs::dates`, `gs::randoms`, etc. directly.

### Assertions and Preconditions

Replace `RC_*` macros with your runner's native assertions — `ASSERT_*`/`EXPECT_*` for GTest, `REQUIRE`/`CHECK` for Catch2 and doctest, `BOOST_TEST` for Boost.Test.

| RapidCheck | Hegel |
|------------|-------|
| `RC_ASSERT(cond)` | Runner's assert (e.g. `ASSERT_TRUE(cond)`, `REQUIRE(cond)`) |
| `RC_ASSERT_FALSE(cond)` | Runner's negated assert |
| `RC_ASSERT_THROWS(expr)` | Runner's throws macro (e.g. `ASSERT_THROW`, `REQUIRE_THROWS`) |
| `RC_PRE(cond)` | `tc.assume(cond)` |
| `RC_DISCARD()` | `tc.assume(false)` |
| `RC_TAG("label")` / `RC_CLASSIFY(...)` | No equivalent — omit |
| `RC_LOG() << "x = " << x` | `tc.note("x = " + std::to_string(x))` |

### Configuration

| RapidCheck | Hegel |
|------------|-------|
| `rc::Configuration{.maxSuccess = 500}` | `hegel::Settings{.test_cases = 500}` |
| `RC_PARAMS="max_success=500"` env var | No equivalent — set via `Settings` |
| Fixed seed via `RC_PARAMS="seed=N"` | `hegel::Settings{.seed = N}` |

### Custom Types

RapidCheck:

```cpp
struct Point { double x, y; };

namespace rc {
template<>
struct Arbitrary<Point> {
    static Gen<Point> arbitrary() {
        return gen::build<Point>(
            gen::set(&Point::x, gen::arbitrary<double>()),
            gen::set(&Point::y, gen::arbitrary<double>()));
    }
};
}

RC_GTEST_PROP(Geometry, DistanceNonNeg, (const Point& a, const Point& b)) {
    RC_ASSERT(distance(a, b) >= 0.0);
}
```

Hegel:

```cpp
struct Point { double x, y; };

TEST(Geometry, DistanceNonNeg) {
    hegel::test([](hegel::TestCase& tc) {
        auto a = tc.draw(gs::default_generator<Point>());
        auto b = tc.draw(gs::default_generator<Point>());
        assert(distance(a, b) >= 0.0);  // or your runner's assertion
    });
}
```

Or, for explicit control over each field's generator:

```cpp
auto point_gen = gs::builds_agg<Point>(
    gs::field<&Point::x>(gs::floats<double>({.min_value = -100.0, .max_value = 100.0})),
    gs::field<&Point::y>(gs::floats<double>({.min_value = -100.0, .max_value = 100.0})));

auto p = tc.draw(point_gen);
```

### Dependent Generation

RapidCheck (requires `mapcat`):

```cpp
auto indexed = rc::gen::mapcat(
    rc::gen::container<std::vector<int>>(rc::gen::arbitrary<int>()),
    [](std::vector<int> v) {
        if (v.empty()) return rc::gen::just(std::make_pair(std::vector<int>{}, size_t{0}));
        auto n = v.size();
        return rc::gen::map(
            rc::gen::inRange<size_t>(0, n),
            [v = std::move(v)](size_t i) { return std::make_pair(v, i); });
    });
```

Hegel (just use sequential draws):

```cpp
hegel::test([](hegel::TestCase& tc) {
    auto v = tc.draw(gs::vectors(gs::integers<int>(), {.min_size = 1}));
    auto i = tc.draw(gs::integers<size_t>({.min_value = 0, .max_value = v.size() - 1}));
    (void)v[i];  // always a valid index
});
```

This is one of hegel's main ergonomic advantages — dependent generation is just sequential code, no combinator gymnastics needed.

### Stateful Testing

RapidCheck has `rc::state::check(initialState, generationFunc, sut)` for state-machine tests. **Hegel-cpp does not yet support stateful testing.** If you are porting a RapidCheck state test, either keep that specific test on RapidCheck for now, or write the state machine loop by hand inside a `hegel::test(...)` block (draw a rule choice with `gs::sampled_from`, dispatch, assert invariants).

## From Catch2 Generators

Catch2 ships "generators" for `GENERATE(...)` blocks, but these are **example enumerators**, not a PBT library — there is no random exploration or shrinking. Any test using `GENERATE(random(lo, hi))` or `GENERATE(take(N, random(...)))` is better reframed as a hegel property test:

Catch2:

```cpp
TEST_CASE("addition is commutative") {
    int a = GENERATE(take(100, random(-1000, 1000)));
    int b = GENERATE(take(100, random(-1000, 1000)));
    REQUIRE(a + b == b + a);
}
```

Hegel (keep Catch2 as the runner — hegel is runner-agnostic):

```cpp
TEST_CASE("addition commutes") {
    hegel::test([](hegel::TestCase& tc) {
        auto a = tc.draw(gs::integers<int>());  // full range, not -1000..1000
        auto b = tc.draw(gs::integers<int>());
        REQUIRE(a + b == b + a);
    });
}
```

## From Boost.Test Data-Driven Tests

Boost.Test's `BOOST_DATA_TEST_CASE(..., random())` similarly enumerates random examples without shrinking. Port the same way — keep Boost.Test as the runner and call `hegel::test(...)` from inside a `BOOST_AUTO_TEST_CASE`, using `BOOST_TEST` for assertions.

## Porting Checklist

When porting tests from RapidCheck (or random-example data-driven tests):

1. **Remove the old dependency** from `CMakeLists.txt` (if no other tests use it) and add hegel via `FetchContent` (see `reference.md`'s Setup section).
2. **Keep the project's test runner.** Replace the `RC_*_PROP` macro with whatever test block the runner already uses (`TEST`, `TEST_CASE`, `BOOST_AUTO_TEST_CASE`, etc.) and call `hegel::test([](hegel::TestCase& tc) { ... })` inside it.
3. **Convert `rc::gen::*` expressions to `tc.draw(gs::*)` calls.** Start with the broadest generators — don't carry over narrow bounds unless they're justified by the function's contract.
4. **Remember `rc::gen::inRange(lo, hi)` is half-open.** Hegel bounds are inclusive, so subtract 1 from the high end.
5. **Replace `RC_ASSERT` with the runner's native assertion** (`ASSERT_EQ`, `REQUIRE`, `BOOST_TEST`, etc.). Replace `RC_PRE` with `tc.assume`.
6. **Replace `RC_LOG` with `tc.note`** — but remember `note` only prints on the final counterexample replay.
7. **Simplify dependent generation.** If the old test used `mapcat` chains just to make later values depend on earlier ones, rewrite as sequential `tc.draw` calls.
8. **Remove custom `rc::Arbitrary<T>` specializations.** Replace with `gs::default_generator<T>()` or `gs::builds_agg<T>` + `gs::field<&T::member>(...)`.
9. **Drop `RC_TAG` / `RC_CLASSIFY`.** Hegel has no equivalent; if the classification was load-bearing, the user should decide what to do (most of the time it's informational and can be removed).
10. **Run the tests.** If they fail on inputs the old framework didn't find, investigate — that's the point.
