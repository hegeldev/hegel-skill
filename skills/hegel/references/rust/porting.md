# Porting Rust PBT Libraries to Hegel

## From Proptest

Proptest is the most common Rust PBT library. The main differences:

- Proptest is declarative (strategies in function signatures or the `proptest!` macro); hegel is imperative (`tc.draw()` calls).
- Proptest shrinking is defined per-Strategy; hegel's engine shrinks the underlying choice sequence automatically, so there are no shrinkers to write or preserve.
- Proptest uses `prop_assert!`; hegel uses standard `assert!`.

### Test Structure

Proptest:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_addition(a in 0..100i32, b in 0..100i32) {
        prop_assert!(a + b >= a);
        prop_assert!(a + b >= b);
    }
}
```

Hegel:

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_addition(tc: hegel::TestCase) {
    let a = tc.draw(generators::integers::<i32>().min_value(0).max_value(99));
    let b = tc.draw(generators::integers::<i32>().min_value(0).max_value(99));
    assert!(a + b >= a);
    assert!(a + b >= b);
}
```

But consider: should those bounds be there at all? If the property is about addition, test the full range unless there's a reason not to.

### Strategy → Generator Mapping

| Proptest | Hegel |
|----------|-------|
| `any::<i32>()` | `generators::integers::<i32>()` |
| `0..100i32` | `generators::integers::<i32>().min_value(0).max_value(99)` |
| `any::<bool>()` | `generators::booleans()` |
| `any::<f64>()` | `generators::floats::<f64>()` — but note: proptest's `any::<f64>()` excludes NaN by default, hegel's `floats()` includes it. A mechanical port silently broadens the domain and breaks `==`-based roundtrips; decide explicitly whether NaN belongs to the property |
| `"[a-z]{1,10}"` | `generators::from_regex(r"[a-z]{1,10}")` |
| `any::<String>()` | `generators::text()` |
| `prop::collection::vec(strat, 0..10)` | `generators::vecs(gen).max_size(9)` |
| `prop::collection::hash_set(strat, 0..5)` | `generators::hashsets(gen).max_size(4)` |
| `prop::collection::hash_map(k, v, 0..5)` | `generators::hashmaps(k, v).max_size(4)` |
| `prop::option::of(strat)` | `generators::optional(gen)` |
| `(strat_a, strat_b)` | `generators::tuples!(gen_a, gen_b)` |
| `Just(value)` | `generators::just(value)` |
| `prop_oneof![s1, s2]` | `hegel::one_of!(g1, g2)` |
| `strat.prop_map(f)` | `gen.map(f)` |
| `strat.prop_flat_map(f)` | `gen.flat_map(f)` |
| `strat.prop_filter(msg, f)` | `gen.filter(f)` |
| `strat.boxed()` | `gen.boxed()` |

### Assertions

| Proptest | Hegel |
|----------|-------|
| `prop_assert!(cond)` | `assert!(cond)` |
| `prop_assert_eq!(a, b)` | `assert_eq!(a, b)` |
| `prop_assert_ne!(a, b)` | `assert_ne!(a, b)` |
| `prop_assume!(cond)` | `tc.assume(cond)` |

### Configuration

| Proptest | Hegel |
|----------|-------|
| `ProptestConfig::with_cases(500)` | `#[hegel::test(test_cases = 500)]` |
| `ProptestConfig { max_shrink_iters: 0, .. }` | `Settings::phases` without `Phase::Shrink` |
| `PROPTEST_CASES=500` env var | No equivalent |

### Derive

Proptest:

```rust
use proptest_derive::Arbitrary;

#[derive(Debug, Arbitrary)]
struct Point { x: f64, y: f64 }

proptest! {
    #[test]
    fn test_point(p: Point) { /* ... */ }
}
```

Hegel:

```rust
use hegel::DefaultGenerator;
use hegel::generators::{self, DefaultGenerator as _, Generator};

#[derive(Debug, DefaultGenerator)]
struct Point { x: f64, y: f64 }

#[hegel::test]
fn test_point(tc: hegel::TestCase) {
    let p: Point = tc.draw(generators::default::<Point>());
    // Or customize: tc.draw(Point::default_generator().x(generators::floats().min_value(0.0)))
}
```

### Dependent Generation

Proptest (requires `flat_map`):

```rust
proptest! {
    #[test]
    fn test_valid_index(
        (v, i) in prop::collection::vec(any::<i32>(), 1..100)
            .prop_flat_map(|v| {
                let len = v.len();
                (Just(v), 0..len)
            })
    ) {
        prop_assert!(i < v.len());
    }
}
```

Hegel (just use sequential draws):

```rust
#[hegel::test]
fn test_valid_index(tc: hegel::TestCase) {
    let v: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>()).min_size(1));
    let i = tc.draw(generators::integers::<usize>().min_value(0).max_value(v.len() - 1));
    assert!(i < v.len());
}
```

This is one of hegel's main ergonomic advantages — dependent generation is just sequential code, no combinator gymnastics needed.

## From Quickcheck

Quickcheck is simpler than proptest but more limited.

### Test Structure

Quickcheck:

```rust
#[quickcheck]
fn test_reverse_involution(xs: Vec<i32>) -> bool {
    reverse(&reverse(&xs)) == xs
}
```

Hegel:

```rust
#[hegel::test]
fn test_reverse_involution(tc: hegel::TestCase) {
    let xs: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    assert_eq!(reverse(&reverse(&xs)), xs);
}
```

Key differences:
- Quickcheck infers generators from the function signature via `Arbitrary`; hegel uses explicit `tc.draw()` calls.
- Quickcheck tests return `bool` (or `TestResult`); hegel tests use `assert!`.
- Quickcheck has an 8-parameter limit on the macro; hegel has no limit.

### Arbitrary → Generator

| Quickcheck | Hegel |
|-----------|-------|
| `Arbitrary for T` (trait impl) | `Generator<T>` (trait impl) or `#[derive(DefaultGenerator)]` |
| `fn arbitrary(g: &mut Gen) -> Self` | `fn do_draw(&self, tc: &TestCase) -> T` |
| `fn shrink(&self) -> Box<dyn Iterator>` | Automatic — no shrink implementation needed |
| `g.size()` for size control | Implicit — the engine controls size distribution (draw sizes explicitly if you need large values) |

### Common Patterns

Quickcheck `TestResult` for conditional properties:

```rust
#[quickcheck]
fn test_division(a: i64, b: i64) -> TestResult {
    if b == 0 { return TestResult::discard(); }
    TestResult::from_bool(a == (a / b) * b + (a % b))
}
```

Hegel:

```rust
#[hegel::test]
fn test_division(tc: hegel::TestCase) {
    let a = tc.draw(generators::integers::<i64>());
    let b = tc.draw(generators::integers::<i64>());
    tc.assume(b != 0);
    assert_eq!(a, (a / b) * b + (a % b));
}
```

## Porting Checklist

When porting tests from proptest or quickcheck:

1. **Enumerate the existing properties first.** List every property the old suite tests (expand macros mentally — one `macro_rules!` invocation per type is one property per type). Ported coverage must be a superset of the original: map each old test to a hegel test, and explicitly note any property you intentionally drop and why. Do not rewrite the suite from scratch and assume you covered everything. Exception: when a macro instantiates the same property over a large type family (e.g. 20 storage types), porting one or two representative instantiations and saying so beats mechanically porting all of them — the property, not the instantiation count, is the coverage.
2. **Check how the old suite is wired in before removing anything.** If the existing PBT suite is gated behind a non-default cfg flag or feature (e.g. `#[cfg(property_tests)]`), find out why — often MSRV or dependency weight. Removing the gate or deleting the suite is a behavior change for the maintainer's CI. Prefer adding hegel tests in a location that runs under plain `cargo test`, treating the gated suite as evidence, and leaving it untouched unless the maintainer asks.
3. **Remove the old dependency** from `Cargo.toml` only if nothing else uses it and you've fully ported its tests; otherwise keep both.
4. **Replace the test macro/attribute** with `#[hegel::test]`.
5. **Convert strategies/Arbitrary to `tc.draw()` calls.** Start with the broadest generators — don't carry over narrow bounds from the old framework unless they're justified by the function's contract.
6. **Replace framework-specific assertions** (`prop_assert!`, bool returns) with standard `assert!`.
7. **Replace `prop_assume!` / `TestResult::discard()`** with `tc.assume()`.
8. **Simplify dependent generation.** If the old test used `flat_map` chains just to make later values depend on earlier ones, rewrite as sequential `tc.draw()` calls.
9. **Remove custom `Shrink` implementations.** Hegel handles shrinking automatically.
10. **Strengthen where the original was weak.** Ports are allowed (encouraged) to assert more than the original: a proptest that only checked "parses without error" can become a full value-equality round-trip. Similarly, when porting a fuzz target of the form "generate arbitrary text, check the property only if it happens to parse", note that hegel's default 100 cases will rarely hit the parse-success branch — pair the port with a construction-based generator that builds valid inputs directly, and keep the arbitrary-text version as a no-panic test.
11. **Run the tests.** If they fail on inputs the old framework didn't find, investigate — that's the point. One caution: if a test only fails *after you broadened* a ported generator, check whether the original narrow range was protecting a documented limit of the operation (precision, domain, resource bounds) rather than being historical timidity — see "Beware of properties that seem universal but aren't" in the main skill. Investigate before reporting a bug.
