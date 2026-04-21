# Hegel Rust Reference

## Table of Contents

- [Setup](#setup)
- [Test Structure](#test-structure) — `#[hegel::test]`, builder form, Settings, HealthCheck
- [TestCase Methods](#testcase-methods) — `draw`, `draw_silent`, `assume`, `note`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, binary, collections, tuples, optional, format, regex, random
- [Combinator Methods](#combinator-methods) — `.map()`, `.filter()`, `.flat_map()`, `.boxed()`
- [Macros](#macros) — `one_of!`, `#[hegel::composite]`, `compose!`, `#[derive(DefaultGenerator)]`, `derive_generator!`
- [Rust-Specific Examples](#rust-specific-examples) — Derived generators, randomness, dependent generation
- [Gotchas](#gotchas)
- [Stateful Testing](#stateful-testing) — `#[hegel::state_machine]`, rules, invariants, Variables

## Setup

```bash
cargo add --dev hegeltest```

If the code under test uses `rand` and you need hegel-controlled RNG instances, enable the `rand` feature:

```bash
cargo add --dev hegeltest --features rand
```

Run tests with `cargo test`. Hegel tests use `#[hegel::test]` in place of `#[test]` and integrate directly with the standard Rust test runner.

If something goes wrong with server installation, see https://hegel.dev/reference/installation.

## Test Structure

### `#[hegel::test]` (preferred)

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_addition_commutes(tc: hegel::TestCase) {
    let a = tc.draw(generators::integers::<i64>());
    let b = tc.draw(generators::integers::<i64>());
    assert_eq!(a.wrapping_add(b), b.wrapping_add(a));
}
```

With configuration:

```rust
#[hegel::test(test_cases = 500, verbosity = Verbosity::Verbose, seed = Some(42))]
fn test_with_config(tc: hegel::TestCase) {
    // ...
}
```

Attributes:
- `test_cases: u64` — Number of test cases (default: 100)
- `verbosity: Verbosity` — `Quiet`, `Normal`, `Verbose`, or `Debug`
- `seed: Option<u64>` — Fixed seed for reproducible runs
- `derandomize: bool` — Use a fixed seed derived from the test name (default: `true` in CI)
- `suppress_health_check: [HealthCheck; N]` — Suppress specific health checks (see below)

### `Hegel::new().run()` (builder form)

```rust
use hegel::{Hegel, Settings, Verbosity};

#[test]
fn test_with_builder() {
    Hegel::new(|tc| {
        let n = tc.draw(generators::integers::<i32>());
        assert!(n == n);
    })
    .settings(Settings::new()
        .test_cases(500)
        .verbosity(Verbosity::Verbose))
    .run();
}
```

### Settings and HealthCheck

`Settings` controls test execution. It can be passed to `#[hegel::test]` as named arguments or as a positional `Settings` object:

```rust
use hegel::{HealthCheck, Settings};

// Named arguments (most common)
#[hegel::test(test_cases = 500, derandomize = true)]
fn test_named(tc: hegel::TestCase) { /* ... */ }

// Suppress health checks
#[hegel::test(suppress_health_check = [HealthCheck::FilterTooMuch])]
fn test_filtered(tc: hegel::TestCase) { /* ... */ }

// Suppress all health checks
#[hegel::test(suppress_health_check = HealthCheck::all())]
fn test_all_suppressed(tc: hegel::TestCase) { /* ... */ }

// Positional Settings object
#[hegel::test(Settings::new().test_cases(500))]
fn test_positional(tc: hegel::TestCase) { /* ... */ }
```

`Settings` builder methods:
- `.test_cases(u64)` — Number of test cases (default: 100)
- `.verbosity(Verbosity)` — Output level
- `.seed(Option<u64>)` — Fixed seed for reproducibility
- `.derandomize(bool)` — Use deterministic seed from test name (default: `true` in CI)
- `.database(Option<String>)` — Path for failure database, or `None` to disable
- `.suppress_health_check(impl IntoIterator<Item = HealthCheck>)` — Suppress checks

`HealthCheck` variants:
- `FilterTooMuch` — Too many test cases rejected via `assume()`
- `TooSlow` — Test execution is too slow
- `TestCasesTooLarge` — Generated test cases are too large
- `LargeInitialTestCase` — The smallest natural input is very large

In CI environments (detected automatically), the database is disabled and tests are derandomized by default.

## TestCase Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `draw()` | `fn draw<T: Debug>(&self, gen: impl Generator<T>) -> T` | Draw a value; shown in counterexample output |
| `draw_silent()` | `fn draw_silent<T>(&self, gen: impl Generator<T>) -> T` | Draw without recording (no `Debug` bound) |
| `assume()` | `fn assume(&self, condition: bool)` | Reject this test case if condition is false |
| `note()` | `fn note(&self, message: &str)` | Print debug info (only on final counterexample replay) |

### Usage

```rust
#[hegel::test]
fn test_division(tc: hegel::TestCase) {
    let a = tc.draw(generators::integers::<i64>());
    let b = tc.draw(generators::integers::<i64>());
    tc.assume(b != 0);
    tc.note(&format!("dividing {} by {}", a, b));
    let q = a / b;
    let r = a % b;
    assert_eq!(a, q * b + r);
}
```

## Generator Reference

All generators are in the `hegel::generators` module. Import with:

```rust
use hegel::generators::{self, Generator};
```

The `Generator` trait import is needed for combinator methods (`.map()`, `.filter()`, `.flat_map()`, `.boxed()`).

### Numeric Generators

**`generators::integers::<T>()`** — Generate any integer type

Supported types: `i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`, `u128`, `isize`, `usize`.

```rust
let n: i32 = tc.draw(generators::integers::<i32>());
let bounded: u8 = tc.draw(generators::integers::<u8>()
    .min_value(1)
    .max_value(100));
```

Config methods:
- `.min_value(T)` — Inclusive lower bound
- `.max_value(T)` — Inclusive upper bound

**`generators::floats::<T>()`** — Generate `f32` or `f64`

```rust
let f: f64 = tc.draw(generators::floats::<f64>());
let bounded: f64 = tc.draw(generators::floats::<f64>()
    .min_value(0.0)
    .max_value(1.0));
```

Config methods:
- `.min_value(T)` — Inclusive lower bound
- `.max_value(T)` — Inclusive upper bound
- `.exclude_min(bool)` — Make lower bound exclusive
- `.exclude_max(bool)` — Make upper bound exclusive
- `.allow_nan(bool)` — Default: `true` if unbounded, `false` if bounded
- `.allow_infinity(bool)` — Default: `true` if unbounded on that side

### Boolean Generator

```rust
let b: bool = tc.draw(generators::booleans());
```

### Text and Binary Generators

**`generators::text()`** — Generate `String`

```rust
let s: String = tc.draw(generators::text());
let bounded: String = tc.draw(generators::text()
    .min_size(1).max_size(100));
```

**`generators::binary()`** — Generate `Vec<u8>`

```rust
let bytes: Vec<u8> = tc.draw(generators::binary());
let bounded: Vec<u8> = tc.draw(generators::binary()
    .min_size(10).max_size(50));
```

Config methods (both):
- `.min_size(usize)` — Minimum length (default: 0)
- `.max_size(usize)` — Maximum length

### Constant and Choice Generators

```rust
// Always returns the same value
let x: i32 = tc.draw(generators::just(42));

// Always returns ()
let u: () = tc.draw(generators::unit());

// Sample from a fixed set
let suit: &str = tc.draw(generators::sampled_from(
    vec!["hearts", "diamonds", "clubs", "spades"]));
```

### Collection Generators

**`generators::vecs(element_gen)`** — Generate `Vec<T>`

```rust
let v: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>()));
let bounded: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>())
    .min_size(1).max_size(10));
let unique: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>())
    .unique());
```

Config methods:
- `.min_size(usize)` — Minimum length (default: 0)
- `.max_size(usize)` — Maximum length
- `.unique()` — All elements distinct

**`generators::hashsets(element_gen)`** — Generate `HashSet<T>` where `T: Eq + Hash`

```rust
let s: HashSet<i32> = tc.draw(generators::hashsets(generators::integers::<i32>())
    .min_size(1).max_size(5));
```

**`generators::hashmaps(key_gen, value_gen)`** — Generate `HashMap<K, V>`

```rust
let m: HashMap<String, i32> = tc.draw(generators::hashmaps(
    generators::text().max_size(10),
    generators::integers::<i32>(),
).max_size(5));
```

**`generators::arrays::<T, N>(element_gen)`** — Generate `[T; N]`

```rust
let arr: [i32; 5] = tc.draw(generators::arrays::<i32, 5>(generators::integers()));
```

**`generators::fixed_dicts()`** — Generate CBOR maps with fixed keys

```rust
let map = tc.draw(generators::fixed_dicts()
    .field("name", generators::text())
    .field("age", generators::integers::<u32>())
    .build());
```

### Tuple Generators

Use the `tuples!` macro with 2–12 component generators:

```rust
let pair: (i32, String) = tc.draw(generators::tuples!(
    generators::integers::<i32>(),
    generators::text(),
));

let triple: (bool, i32, f64) = tc.draw(generators::tuples!(
    generators::booleans(),
    generators::integers::<i32>(),
    generators::floats::<f64>(),
));
```

### Optional Generator

```rust
let maybe: Option<i32> = tc.draw(
    generators::optional(generators::integers::<i32>()));
```

### Format Generators

```rust
let email: String = tc.draw(generators::emails());
let url: String = tc.draw(generators::urls());
let domain: String = tc.draw(generators::domains().with_max_length(50));
let date: String = tc.draw(generators::dates());        // YYYY-MM-DD
let time: String = tc.draw(generators::times());         // HH:MM:SS
let dt: String = tc.draw(generators::datetimes());
let ipv4: String = tc.draw(generators::ip_addresses().v4());
let ipv6: String = tc.draw(generators::ip_addresses().v6());
```

### Regex Generator

```rust
let code: String = tc.draw(
    generators::from_regex(r"[A-Z]{3}-[0-9]{3}").fullmatch(true));
```

- `.fullmatch(bool)` — Require the pattern matches the entire string

### Random Generator (requires `rand` feature)

```rust
// Cargo.toml: hegel = { ..., features = ["rand"] }

// Default: artificial randomness — every random decision is shrinkable
let mut rng = tc.draw(generators::randoms());

// True randomness — single shrinkable seed, real StdRng output
let mut rng = tc.draw(generators::randoms().use_true_random(true));
```

The returned `HegelRandom` implements `rand::RngCore` (rand 0.9).

**Default mode** routes every `next_u32`/`next_u64`/`fill_bytes` call through hegel, so the shrinker can minimize individual random decisions. Best for most code.

**`use_true_random()` mode** generates a single seed via hegel then creates a real `StdRng`. Use this when the code under test does rejection sampling or other algorithms that need statistically random-looking output — artificial randomness can cause these to loop indefinitely.

**Rand version compatibility:** hegel uses rand 0.9. If the project uses rand 0.8, the traits are incompatible. Ask the user to upgrade rand (main changes: `gen_range` -> `random_range`, `gen::<T>()` -> `random::<T>()`, `thread_rng()` -> `rng()`, `from_entropy` -> `from_os_rng`). Do not fall back to `ChaCha8Rng::seed_from_u64(hegel_seed)` — that defeats shrinking.

## Combinator Methods

These methods are on the `Generator` trait. You must import it:

```rust
use hegel::generators::Generator;
```

### `.map(f)`

Transform generated values:

```rust
let positive_str: String = tc.draw(
    generators::integers::<u32>()
        .min_value(1)
        .map(|n| n.to_string()));
```

### `.filter(predicate)`

Keep only values matching a predicate:

```rust
let even: i32 = tc.draw(
    generators::integers::<i32>()
        .filter(|x| x % 2 == 0));
```

Note: `.filter()` retries up to 3 times, then calls `tc.assume(false)`. Prefer bounds over filters when possible.

### `.flat_map(f)`

Dependent generation — use one value to choose the next generator:

```rust
let (n, v): (usize, Vec<i32>) = tc.draw(
    generators::integers::<usize>()
        .min_value(1)
        .max_value(5)
        .flat_map(|n| {
            generators::vecs(generators::integers::<i32>())
                .min_size(n).max_size(n)
                .map(move |v| (n, v))
        }));
assert_eq!(v.len(), n);
```

### `.boxed()`

Type-erase a generator for use in collections or polymorphic contexts:

```rust
let gen: BoxedGenerator<i32> = generators::integers::<i32>().boxed();
```

## Macros

### `one_of!`

Choose between multiple generators of the same type:

```rust
let n: i32 = tc.draw(hegel::one_of!(
    generators::just(0),
    generators::integers::<i32>().min_value(1).max_value(100),
    generators::integers::<i32>().min_value(-100).max_value(-1),
));
```

All branches must return the same type.

### `#[hegel::composite]`

Define a reusable generator as a function. The first parameter must be `TestCase`; additional parameters become arguments to the generator. The function must have an explicit return type.

```rust
#[hegel::composite]
fn points(tc: hegel::TestCase, max_coord: f64) -> (f64, f64) {
    let x = tc.draw(generators::floats::<f64>().min_value(-max_coord).max_value(max_coord));
    let y = tc.draw(generators::floats::<f64>().min_value(-max_coord).max_value(max_coord));
    (x, y)
}

#[hegel::test]
fn test_points(tc: hegel::TestCase) {
    let (x, y) = tc.draw(points(100.0));
    assert!(x.abs() <= 100.0);
}
```

This is generally preferred over `compose!` because it creates a named, reusable generator that can take parameters.

### `compose!`

Build an inline generator from imperative code (useful for one-off generators that don't need to be reused):

```rust
use hegel::compose;

let point_gen = compose!(|tc| {
    let x = tc.draw(generators::floats::<f64>().min_value(-100.0).max_value(100.0));
    let y = tc.draw(generators::floats::<f64>().min_value(-100.0).max_value(100.0));
    (x, y)
});

let (x, y): (f64, f64) = tc.draw(point_gen);
```

### `#[derive(DefaultGenerator)]`

Auto-derive a generator for structs you own:

```rust
use hegel::DefaultGenerator;
use hegel::generators::{self, DefaultGenerator as _, Generator};

#[derive(DefaultGenerator, Debug)]
struct User {
    name: String,
    age: u32,
    active: bool,
}

#[hegel::test]
fn test_user(tc: hegel::TestCase) {
    // Default generators for all fields:
    let user: User = tc.draw(generators::default::<User>());

    // Customize specific fields:
    let adult: User = tc.draw(User::default_generator()
        .age(generators::integers().min_value(18).max_value(120))
        .name(generators::from_regex(r"[A-Z][a-z]{2,15}").fullmatch(true)));
    assert!(adult.age >= 18);
}
```

The derive implements the `DefaultGenerator` trait and creates a generator with:
- `generators::default::<Type>()` or `Type::default_generator()` — Uses default generators for all fields
- `.<field>(gen)` — Override a specific field's generator

Works with enums too.

### `derive_generator!`

For types you don't own:

```rust
use hegel::derive_generator;
use hegel::generators::{self, DefaultGenerator, Generator};

struct Point { x: f64, y: f64 }

derive_generator!(Point { x: f64, y: f64 });

#[hegel::test]
fn test_point(tc: hegel::TestCase) {
    let p: Point = tc.draw(generators::default::<Point>()
        .x(generators::floats().min_value(-10.0).max_value(10.0))
        .y(generators::floats().min_value(-10.0).max_value(10.0)));
}
```

## Rust-Specific Examples

These examples show Rust-specific features. For general property patterns (round-trip, model-based, idempotence, etc.), see the main skill's Property Catalogue.

### Dependent generation with sequential draws

Hegel's imperative style means dependent generation is just sequential code — no `flat_map` needed:

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_valid_index(tc: hegel::TestCase) {
    let v: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>())
        .min_size(1));
    let idx = tc.draw(generators::integers::<usize>()
        .min_value(0)
        .max_value(v.len() - 1));
    // idx is always a valid index
    let _ = v[idx];
}
```

### Custom type with derived generator

```rust
use hegel::DefaultGenerator;
use hegel::generators::{self, DefaultGenerator as _, Generator};

#[derive(DefaultGenerator, Debug, Clone, PartialEq)]
struct Config {
    max_retries: u32,
    timeout_ms: u64,
    name: String,
}

#[hegel::test]
fn test_config_merge(tc: hegel::TestCase) {
    let base = tc.draw(generators::default::<Config>());
    let override_cfg = tc.draw(generators::default::<Config>());
    let merged = base.merge(&override_cfg);
    // Property: merged config should have override's values
    assert_eq!(merged.name, override_cfg.name);
}
```

### Testing code that uses randomness

```rust
use hegel::generators::{self, Generator};

// Code under test: fn sample(weights: &[f64], rng: &mut impl Rng) -> usize

#[hegel::test]
fn test_sample_returns_valid_index(tc: hegel::TestCase) {
    let weights: Vec<f64> = tc.draw(generators::vecs(
        generators::floats::<f64>().min_value(0.0).exclude_min(true)
    ).min_size(1));
    let mut rng = tc.draw(generators::randoms());
    let idx = sample(&weights, &mut rng);
    assert!(idx < weights.len());
}
```

If the code does rejection sampling and the test hangs with the default mode, switch to `use_true_random()`:

```rust
#[hegel::test]
fn test_rejection_sampler(tc: hegel::TestCase) {
    let weights: Vec<f64> = tc.draw(generators::vecs(
        generators::floats::<f64>().min_value(0.0).exclude_min(true)
    ).min_size(1));
    // use_true_random(true) avoids hangs from rejection sampling loops
    let mut rng = tc.draw(generators::randoms().use_true_random(true));
    let idx = rejection_sample(&weights, &mut rng);
    assert!(idx < weights.len());
}
```

### Wrapping arithmetic in test values

When computing test values from generated data, use wrapping operations to avoid panics in your *test* code:

```rust
// BAD — panics when k is near i32::MAX
map.insert(k, k * 10);

// GOOD — wrapping prevents test overflow
map.insert(k, k.wrapping_mul(10));

// ALSO GOOD — use smaller types for intermediate computation
let k = tc.draw(generators::integers::<i16>()) as i32;
let k_squared = k * k;  // can't overflow i32
```

## Gotchas

1. **Import `Generator` trait for combinators.** `.map()`, `.filter()`, `.flat_map()`, and `.boxed()` require `use hegel::generators::Generator`. Without the import, these methods won't be available.

2. **`#[hegel::test]` replaces `#[test]`, not both.** Don't write `#[test] #[hegel::test]` — the hegel macro already generates the test attribute.

3. **Add `.hegel/` to `.gitignore`.** Hegel creates a `.hegel/` directory for caching the server binary and storing the database of previous failures. Add it to `.gitignore`.

4. **Float defaults include NaN and infinity.** `generators::floats::<f64>()` with no bounds generates NaN and infinity by default. If your code doesn't handle these, use `.allow_nan(false)` and/or `.allow_infinity(false)` — but consider whether the code *should* handle them first.

5. **Type annotations are required for numeric generators.** `generators::integers()` won't compile — you must write `generators::integers::<i32>()` (or whatever type you need).

6. **Excessive assume/filter rejections fail the test.** If `tc.assume()` or `.filter()` rejects too many inputs, Hegel gives up. Restructure your generators to produce valid inputs directly.

7. **`note()` only prints on the final replay.** Don't rely on `tc.note()` for progress logging — it only appears when displaying the minimal counterexample.

8. **`target()` is not yet available** in Hegel-rust. It is planned for a future release.

9. **Default collection sizes are small.** `generators::vecs(gen)` with no bounds rarely produces 100+ elements. If you need large collections (e.g., to test tree traversal at depth), draw the size separately:
   ```rust
   let n = tc.draw(generators::integers::<usize>().max_value(300));
   let keys: Vec<i32> = tc.draw(generators::vecs(generators::integers()).min_size(n));
   ```

10. **Use `.unique()` for map/set key generation.** When testing ordered maps or sets, generate unique keys to avoid ambiguity about which value wins:
    ```rust
    let keys: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>())
        .max_size(50).unique());
    ```

## Stateful Testing

Hegel supports stateful (model-based) testing via `#[hegel::state_machine]`. Define rules (actions) and invariants (assertions checked after each rule), then run the state machine.

### Defining a State Machine

```rust
use hegel::TestCase;
use hegel::generators::integers;

struct IntegerStack {
    stack: Vec<i32>,
}

#[hegel::state_machine]
impl IntegerStack {
    #[rule]
    fn push(&mut self, tc: TestCase) {
        let element = tc.draw(integers::<i32>());
        self.stack.push(element);
    }

    #[rule]
    fn pop(&mut self, _: TestCase) {
        self.stack.pop();
    }

    #[rule]
    fn push_pop(&mut self, tc: TestCase) {
        let initial = self.stack.clone();
        let element = self.stack.pop();
        tc.assume(element.is_some());
        let element = element.unwrap();
        self.stack.push(element);
        assert_eq!(self.stack, initial);
    }

    #[invariant]
    fn length_nonnegative(&self, _: TestCase) {
        assert!(self.stack.len() < 100, "stack too large");
    }
}

#[hegel::test]
fn test_integer_stack(tc: TestCase) {
    let stack = IntegerStack { stack: Vec::new() };
    hegel::stateful::run(stack, tc);
}
```

- **`#[rule]`** methods are actions that can be applied. They take `&mut self` and `TestCase`. Use `tc.assume()` to skip a rule when it doesn't apply (e.g., can't pop from an empty stack).
- **`#[invariant]`** methods are checked after every successful rule. They take `&self` and `TestCase`.
- Call `hegel::stateful::run(machine, tc)` from a `#[hegel::test]` to execute.

### Variables (Pools)

For tests that need to track dynamically created resources (accounts, handles, keys), use `Variables`:

```rust
use hegel::stateful::{Variables, variables};

struct MyTest {
    accounts: Variables<String>,
    // ... other state
}

#[hegel::state_machine]
impl MyTest {
    #[rule]
    fn create_account(&mut self, tc: TestCase) {
        let name = tc.draw(generators::text().min_size(1));
        self.accounts.add(name);
    }

    #[rule]
    fn use_account(&mut self, tc: TestCase) {
        let account = self.accounts.draw().clone();  // borrows from pool
        // ... do something with account
    }

    #[rule]
    fn delete_account(&mut self, tc: TestCase) {
        let account = self.accounts.consume();  // removes from pool
        // ... clean up account
    }
}

#[hegel::test]
fn test_my_system(tc: TestCase) {
    let test = MyTest {
        accounts: variables(&tc),
    };
    hegel::stateful::run(test, tc);
}
```

`Variables<T>` methods:
- `.add(value)` — Add a value to the pool
- `.draw()` — Borrow a random value (calls `assume(false)` if empty)
- `.consume()` — Remove and return a random value (calls `assume(false)` if empty)
- `.empty()` — Check if the pool is empty
