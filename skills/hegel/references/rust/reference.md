# Hegel Rust Reference

## Table of Contents

- [Setup](#setup)
- [Test Structure](#test-structure) — `#[hegel::test]`, builder form, Settings, HealthCheck
- [TestCase Methods](#testcase-methods) — `draw`, `draw_silent`, `assume`, `note`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, binary, collections, tuples, optional, format, regex
- [Combinator Methods](#combinator-methods) — `.map()`, `.filter()`, `.flat_map()`, `.boxed()`
- [Macros](#macros) — `one_of!`, `#[hegel::composite]`, `compose!`, `#[derive(DefaultGenerator)]`, `derive_generator!`
- [Rust-Specific Examples](#rust-specific-examples) — Derived generators, randomness, dependent generation
- [Gotchas](#gotchas)
- [Stateful Testing](#stateful-testing) — `#[hegel::state_machine]`, rules, invariants, Pools

For generators that integrate with third-party crates (`chrono`, `jiff`, `serde_json`, `rand`), see `references/rust/extras.md`. Load that file only when the code under test uses one of those crates.

## Setup

```bash
cargo add --dev hegeltest
```

The *package* is named `hegeltest`, but its *library crate* is named `hegel`: all imports and attributes use the `hegel` name (`use hegel::generators;`, `#[hegel::test]`). In a Cargo workspace, add `-p <crate>` so the dependency lands in the right member's `Cargo.toml`. If the crate under test also appears inside hegeltest's own dependency tree (e.g. testing `miniz_oxide`, which hegeltest depends on), `cargo test -p <name>` is ambiguous — disambiguate with `-p <name>@<version>`.

If the target crate pins a `rust-version` older than hegeltest's MSRV, `cargo add` fails with an MSRV resolution error; retry with `--ignore-rust-version`. When upstreaming such tests, call out the dev-dependency MSRV in the PR — whether to accept it is the maintainer's decision.

**Edition 2015 crates**: `use hegel::...` will not resolve on its own. Add `extern crate hegel;` at the top of the test file (integration tests) or `#[cfg(test)] extern crate hegel;` at the crate root (unit test modules). Everything else, including `#[hegel::test]`, works unchanged on edition 2015.

**`no_std` crates**: `#[hegel::test]`'s generated code assumes the std prelude (it calls `.to_string()` and `format!`), so unit-test modules in `#![no_std]` crates fail with E0599/E0425 pointing at the attribute. Fix: add `use std::{format, string::ToString};` (or `use alloc::{format, string::{String, ToString}};` in `no_std + alloc` crates) to each test module — composites returning `String` need the `String` import too, and `one_of!` expands to the `vec!` macro, so `no_std + alloc` modules using it also need `use alloc::vec;`. In crates that don't declare an `std` dependency at all (core-style `no_std`), `use std::...` itself fails E0433 — put `extern crate std;` inside the test module first. The tests themselves still run under std — this is only about name resolution inside the crate. Related: on edition ≤2018 crates, `panic!`/`assert!` single-argument messages are not format strings — `assert!(cond, "{x:?}")` prints the literal braces; pass format arguments explicitly.

If the code under test uses `rand` and you need hegel-controlled RNG instances, enable the `rand` feature:

```bash
cargo add --dev hegeltest --features rand
```

Run tests with `cargo test`. Hegel tests use `#[hegel::test]` in place of `#[test]` and integrate directly with the standard Rust test runner.

Hegel runs entirely in-process: the engine (the `hegeltest-c` crate) is a normal Cargo dependency compiled from source.

Note that adding hegeltest substantially inflates the test binary. This is normally harmless, but pre-existing tests that read *their own binary* as sample data (a trick some codec crates use) suddenly process tens of megabytes and can look like a hung hegel property — check what a slow pre-existing test is actually reading before diagnosing.

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
#[hegel::test(test_cases = 500, verbosity = hegel::Verbosity::Verbose, seed = Some(42))]
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

The closure is `FnMut(TestCase)` with no `'static` bound, so the builder form can borrow expensive shared fixtures (a test repo, a loaded font) that `#[hegel::test]`'s free function cannot.

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
- `.database(Option<String>)` — Path for failure database (default: `.hegel/examples`), or `None` to disable
- `.suppress_health_check(impl IntoIterator<Item = HealthCheck>)` — Suppress checks; *replaces* any previously configured suppressions, so pass all checks in one call
- `.phases(impl IntoIterator<Item = Phase>)` — Restrict which lifecycle phases run (`Phase::Explicit`, `Reuse`, `Generate`, `Target`, `Shrink`); e.g. omitting `Phase::Shrink` disables shrinking
- `.print_blob(bool)` — On failure, print a base64 blob replayable via `#[hegel::reproduce_failure("…")]` stacked below `#[hegel::test]` (default: `false`)
- `.report_multiple_failures(bool)` — Report every distinct failure found instead of only the first (default: `false`)
- `.backend(Backend)` — Randomness source; `Backend::Urandom` exists for running under Antithesis (auto-selected there)

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
| `target()` | `fn target(&self, score: f64)` | Feed an observation back to the engine to guide generation toward higher-scoring inputs |
| `target_labelled()` | `fn target_labelled(&self, score: f64, label: impl Into<String>)` | Like `target()`, but with an explicit label so multiple targets are optimised independently |
| `reject()` | `fn reject(&self) -> !` | Like `assume(false)`, but returns `!` so the compiler knows code after the call is unreachable |

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

### Targeted Testing

`tc.target(score)` feeds an observation back to hegel's engine. The engine uses the score to bias generation toward higher-scoring inputs, which is useful for stress-testing properties that fail at extremes:

```rust
#[hegel::test]
fn my_test(tc: hegel::TestCase) {
    let n: u64 = tc.draw(generators::integers::<u64>().max_value(1000));
    let m: u64 = tc.draw(generators::integers::<u64>().max_value(1000));
    tc.target((n + m) as f64);
    assert!(n + m < 2000);
}
```

Inside `#[hegel::test]`, `#[hegel::main]`, or `#[hegel::standalone_function]`, `tc.target(expr)` is rewritten to `tc.target_labelled(expr, "expr")` so separate targeting expressions get separate labels by default. Call `tc.target_labelled(score, "...")` directly to control the label yourself.

## Generator Reference

All generators are in the `hegel::generators` module. Import with:

```rust
use hegel::generators;
```

Add the `Generator` trait — `use hegel::generators::{self, Generator};` — only when you use the combinator methods (`.map()`, `.filter()`, `.flat_map()`, `.boxed()`); importing it unused triggers an `unused_imports` warning.

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

`text()` also takes character constraints: `.alphabet("abc")`, `.codec("ascii")`, `.min_codepoint(u32)` / `.max_codepoint(u32)`, `.categories(&["Lu"])` / `.exclude_categories(&[...])` (Unicode general categories), `.include_characters("é")` / `.exclude_characters("\0")`. `generators::characters()` generates single `char`s with the same constraint methods (minus `.alphabet` — use `sampled_from` for that).

### Constant and Choice Generators

```rust
// Always returns the same value (T must be Clone + Send + Sync)
let x: i32 = tc.draw(generators::just(42));

// Always returns ()
let u: () = tc.draw(generators::unit());

// Sample from a fixed set
let suit: &str = tc.draw(generators::sampled_from(
    vec!["hearts", "diamonds", "clubs", "spades"]));
```

`sampled_from` takes any `Vec` of owned values, not just `&str` — sampling which API object a property runs against is a workhorse pattern:

```rust
// e.g. run one property across every codec/encoding/config the crate ships
let encoding: Encoding = tc.draw(generators::sampled_from(all_encodings()));
```

Note `just` and `sampled_from` require `T: Send + Sync` (plus `Clone` for `just`). For non-`Send` values (tagged pointers, `Rc`-based types), draw a plain discriminant (`integers`/`sampled_from` over an enum of your own) and construct the value imperatively after the draw.

### Collection Generators

**`generators::vecs(element_gen)`** — Generate `Vec<T>`

```rust
let v: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>()));
let bounded: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>())
    .min_size(1).max_size(10));
let unique: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>())
    .unique(true));
```

Config methods:
- `.min_size(usize)` — Minimum length (default: 0)
- `.max_size(usize)` — Maximum length
- `.unique(bool)` — All elements distinct

For an exact-size `Vec`, set both: `.min_size(n).max_size(n)`.

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

**`generators::arrays(element_gen)`** — Generate `[T; N]` (annotate the binding to pin `N`)

```rust
let arr: [i32; 5] = tc.draw(generators::arrays(generators::integers::<i32>()));
```

Let inference pick the size from the annotation — the function has three generic parameters (`arrays<G, T, const N>`), so the turbofish form `arrays::<i32, 5>(...)` is an E0107 error.

### Tuple Generators

Use the `tuples!` macro with up to 12 component generators:

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
let domain: String = tc.draw(generators::domains().max_length(50));
let uuid: String = tc.draw(generators::uuids());              // hyphenated; .version(4) to pin
let date: String = tc.draw(generators::date_strings());       // YYYY-MM-DD
let time: String = tc.draw(generators::time_strings());       // HH:MM:SS[.ffffff]
let dt: String = tc.draw(generators::datetime_strings());     // ISO 8601
let ip: std::net::IpAddr = tc.draw(generators::ip_addresses());
let ipv4: std::net::Ipv4Addr = tc.draw(generators::ip_addresses().v4());
let ipv6: std::net::Ipv6Addr = tc.draw(generators::ip_addresses().v6());
let d: std::time::Duration = tc.draw(generators::durations()
    .max_value(std::time::Duration::from_secs(60)));
```

The `*_strings()` date/time generators are not configurable; for typed, boundable date/time values use the `chrono` or `jiff` extras (see `references/rust/extras.md`).

### Characters and Codepoints

There is no dedicated `char` generator. The idiom is drawing a codepoint and converting, with `one_of!` to weight interesting planes:

```rust
let c: char = tc.draw(hegel::one_of!(
    generators::integers::<u32>().max_value(0x7F),                        // ASCII
    generators::integers::<u32>().min_value(0x80).max_value(0xFFFF),      // BMP
    generators::integers::<u32>().min_value(0x1_0000).max_value(0x10_FFFF), // supplementary
).map(|n| char::from_u32(n).unwrap_or('\u{FFFD}')));
```

`generators::text()`'s character distribution is not documented/configurable — if a property lives on specific characters (line breaks, controls, combining marks), build the string from drawn codepoints or mix `text()` with targeted insertions rather than hoping `text()` produces them.

### Regex Generator

```rust
let code: String = tc.draw(generators::from_regex(r"[A-Z]{3}-[0-9]{3}"));
```

- `.fullmatch(bool)` — Whether the entire string must match the pattern (default: `true`); `false` generates strings that merely contain a match
- `.alphabet(generators::characters()...)` — Constrain which characters may appear

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

All branches must yield the same *value* type; the generator types may differ (box the branches with `.boxed()` only if the compiler demands unification).

### `#[hegel::composite]`

Define a reusable generator as a function. The first parameter must be `TestCase`; additional parameters become arguments to the generator. The function must have an explicit return type.

Note: the macro expands the body into a `Fn` closure, so a non-`Copy` extra parameter (e.g. `Vec<usize>`, `String`) captured across draws hits E0507 ("cannot move out of captured variable in an `Fn` closure"). Bind a `.clone()` of it inside the function before using it, or take it by a `Copy`/reference-friendly form.

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

The body must return a plain *value*, drawing from any inner generators with `tc.draw(...)`. Returning a generator expression (e.g. a bare `one_of!(...)`) is a type error — to combine generators inside a composite, draw from the combination:

```rust
#[hegel::composite]
fn small_or_boundary(tc: hegel::TestCase) -> i64 {
    tc.draw(hegel::one_of!(
        generators::integers::<i64>().min_value(-100).max_value(100),
        generators::sampled_from(vec![i64::MIN, i64::MAX]),
    ))
}
```

Composites can draw from other composites — `tc.draw(points(100.0))` inside another composite works fine.

**Recursive composites don't compile.** A `#[hegel::composite]` that (directly or mutually) draws from itself fails with an opaque `E0283`-style error about `impl Fn(TestCase) -> T` and `Send` — the opaque return type becomes self-referential. For recursive data (trees, nested documents), write a plain function that takes depth and returns a boxed generator via `compose!`:

```rust
fn toml_values(depth: u32) -> hegel::generators::BoxedGenerator<'static, Value> {
    hegel::compose!(|tc| {
        if depth == 0 || tc.draw(generators::integers::<u8>().max_value(3)) > 0 {
            Value::from(tc.draw(generators::integers::<i64>()))
        } else {
            let inner = tc.draw(generators::vecs(toml_values(depth - 1)));
            Value::from_iter(inner)
        }
    })
    .boxed()
}
```

Two details of that pattern: the return type needs the `'static` lifetime parameter spelled out, and `compose!` takes exactly `|tc| { ... }` — it inserts `move` itself, so writing `compose!(move |tc| ...)` is rejected.

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
        .name(generators::from_regex(r"[A-Z][a-z]{2,15}")));
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

### Helper functions taking `&TestCase`

Not every reusable draw needs a composite. A plain function taking `&hegel::TestCase` works and is often simpler, especially for effectful op-sequence generation inside state machines (`TestCase` methods take `&self`, so borrowing is all you need):

```rust
fn draw_key(tc: &hegel::TestCase) -> Vec<u8> {
    tc.draw(generators::binary().max_size(64))
}
```

Two differences from `#[hegel::composite]`: draws inside plain helpers print as unnamed `draw_N` in counterexamples (composites group them under the composite's span), and helpers can't be passed where a `Generator` is expected. For draw-heavy helpers whose individual values would clutter counterexample output, use `tc.draw_silent(...)` for the internals and `tc.note(...)` to record the assembled value.

For sharing generators between `src/` unit tests and `tests/` integration tests, Rust visibility forces a choice: a `#[cfg(test)]` helpers module for unit tests plus a `tests/common/mod.rs` for integration tests (some duplication), or putting all hegel tests on one side.

### Drawing schedules for streaming APIs

When driving a streaming/chunked API, don't draw inside an unbounded feed loop (each draw consumes engine choices, and an input-dependent number of draws shrinks poorly). Draw a bounded *schedule* up front and cycle it:

```rust
let chunk_sizes: Vec<usize> = tc.draw(generators::vecs(
    generators::integers::<usize>().min_value(1).max_value(64),
).min_size(1).max_size(8));
let mut i = 0;
while has_more_input() {
    feed(next_chunk(chunk_sizes[i % chunk_sizes.len()]));
    i += 1;
}
```

### APIs whose returned handle borrows the input

Some eval/parse APIs return a handle that *borrows* the source you passed (e.g. `compile_expression(src) -> Expression<'a>`). Calling one on a freshly-formatted string — `engine.compile_expression(&format!("{a} + {b}"))` — fails with E0716 (temporary dropped while borrowed), because the `format!` temporary dies at the end of the statement while the returned handle still borrows it. Bind the source to a `let` first, then pass a reference:

```rust
let src = format!("{a} + {b}");
let expr = engine.compile_expression(&src)?;   // src outlives expr
```

### Drawing a shrinkable random subset / permutation

When a property needs "any k of these n items" (which shards to erase, which indices to drop), don't reach for `rand` — that gives an unshrinkable choice. Draw the selection through `tc` so it shrinks. A partial Fisher-Yates over drawn swap targets yields a uniform, fully-shrinkable subset:

```rust
fn draw_subset(tc: &hegel::TestCase, n: usize, k: usize) -> Vec<usize> {
    let mut pool: Vec<usize> = (0..n).collect();
    let mut chosen = Vec::with_capacity(k);
    for i in 0..k {
        // draw an index into the remaining pool; shrinks toward earlier items
        let j = tc.draw(generators::integers::<usize>().min_value(i).max_value(n - 1));
        pool.swap(i, j);
        chosen.push(pool[i]);
    }
    chosen
}
```

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

This uses the `rand` extra — see `references/rust/extras.md` for the feature flag and the full `randoms()` API (including the artificial-vs-true-random modes).

```rust
use hegel::generators::{self, Generator};
use hegel::extras::rand as rand_gs;

// Code under test: fn sample(weights: &[f64], rng: &mut impl Rng) -> usize

#[hegel::test]
fn test_sample_returns_valid_index(tc: hegel::TestCase) {
    let weights: Vec<f64> = tc.draw(generators::vecs(
        generators::floats::<f64>().min_value(0.0).exclude_min(true)
    ).min_size(1));
    let mut rng = tc.draw(rand_gs::randoms());
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
    let mut rng = tc.draw(rand_gs::randoms().use_true_random(true));
    let idx = rejection_sample(&weights, &mut rng);
    assert!(idx < weights.len());
}
```

### Floating-point tolerances at extremes

When comparing floats with a magnitude-scaled tolerance, two failure modes bite at the extremes of the input range and produce false counterexamples:
- **Underflow to zero.** An `eps * magnitude` *absolute* tolerance underflows to `0.0` for subnormal-scale inputs, so a legitimate 1-denormal-ULP residual reads as a failure. Floor the tolerance at `f64::MIN_POSITIVE`. Relatedly, a purely *relative* tolerance is unsound when subnormal intermediates appear — include an absolute term.
- **The test's own oracle overflows first.** A `hypot`/sum-of-squares/product you compute to *check* the result can overflow to `inf`/`NaN` before the library misbehaves. Use a max-norm (or otherwise overflow-safe) formulation in the test, and suspect the test's arithmetic before reporting a boundary "bug".
- **The floor may come from library constants, not ULPs.** A crate that ships lower-precision baked constants (e.g. 7-significant-digit color-conversion matrices) has a precision floor set by *those constants*, not by `f64::EPSILON` — a correct roundtrip can be off by ~1e-7. Read the constants to set the tolerance rather than deriving it from ulp arithmetic.
- **For iterative numerics, the source's stop criterion IS the accuracy contract.** A function computed by Newton iteration, a series, or a root-find (many `inverse_cdf`/special-function implementations) is only as accurate as its own convergence test — e.g. a `while |Δ| > 1e-9` loop cannot deliver a relative-accurate answer deep in a tail. Read the stop criterion in the implementation and set your tolerance from *it*, not from the theoretical precision; a disagreement tighter than the code's own criterion is expected behavior, not a bug (a disagreement much *looser* than it — 138 orders off — is a real iteration-exhaustion bug).
- **When pinning a hang/non-termination zone, exclude it with decades-wide margins.** If a property must skip a region because the code hangs or blows up there (e.g. an iterative routine that diverges past some parameter magnitude), the shrinker will walk a naive boundary exclusion right up to the edge and re-trigger the hang. Exclude generously (orders of magnitude inside the safe region) and pin the hang itself with a separate `#[ignore]`d reproducer.

### Arbitrary-precision integers (num-bigint etc.)

There is no built-in bignum generator. For crates built on `num-bigint`, combine a machine-integer generator (covers `MIN`/`MAX`/`0` boundaries) with digit-strings for values wider than any machine type:

```rust
#[hegel::composite]
fn big_ints(tc: hegel::TestCase) -> BigInt {
    tc.draw(hegel::one_of!(
        generators::integers::<i128>().map(BigInt::from),
        generators::from_regex(r"-?[1-9][0-9]{0,59}")
            .map(|s: String| BigInt::from_str(&s).unwrap()),
    ))
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

3. **Add `.hegel/` to `.gitignore`.** Hegel stores its database of previous failures in `.hegel/examples` by default, created on the first recorded failure — don’t be surprised if it doesn’t appear until a test fails. Add `.hegel/` to `.gitignore` up front.

4. **Float defaults include NaN and infinity.** `generators::floats::<f64>()` with no bounds generates NaN and infinity by default. If your code doesn't handle these, use `.allow_nan(false)` and/or `.allow_infinity(false)` — but consider whether the code *should* handle them first.

5. **Type annotations are required for numeric generators.** `generators::integers()` won't compile — you must write `generators::integers::<i32>()` (or whatever type you need).

6. **Excessive assume/filter rejections fail the test.** If `tc.assume()` or `.filter()` rejects too many inputs, Hegel gives up. Restructure your generators to produce valid inputs directly.

7. **`note()` only prints on the final replay.** Don't rely on `tc.note()` for progress logging — at the default verbosity it only appears when displaying the minimal counterexample.

8. **Default collection sizes are small.** `generators::vecs(gen)` with no bounds rarely produces 100+ elements. If you need large collections (e.g., to test tree traversal at depth), draw the size separately:
   ```rust
   let n = tc.draw(generators::integers::<usize>().max_value(300));
   let keys: Vec<i32> = tc.draw(generators::vecs(generators::integers()).min_size(n));
   ```
   The same applies to `text()`: for multi-kilobyte strings (rope/tree structures need them to grow internal levels), draw a target length and build the string (repeat a drawn seed chunk, or collect drawn codepoints) — `text().min_size(n)` with large drawn `n` works but shrinks less gracefully.

9. **Use `.unique(true)` for map/set key generation.** When testing ordered maps or sets, generate unique keys to avoid ambiguity about which value wins:
   ```rust
   let keys: Vec<i32> = tc.draw(generators::vecs(generators::integers::<i32>())
       .max_size(50).unique(true));
   ```

10. **Generators are single-use values.** `tc.draw(gen)` takes the generator by value, so drawing twice from the same variable is a move error. Rebuild the generator per draw (they're cheap), or use a `BoxedGenerator` (`.boxed()`), which implements `Clone`.

11. **Passing tests print nothing extra.** A passing hegel test looks exactly like a passing unit test; there is no per-case output. To confirm cases are actually being generated, run once with `#[hegel::test(verbosity = hegel::Verbosity::Verbose)]` (the qualified path — bare `Verbosity` is E0433 without an import), or temporarily break the property and check that hegel reports a shrunk counterexample. (A deliberate liveness-check failure leaves an entry in the `.hegel/` failure database — harmless, but delete `.hegel/` afterwards to avoid replaying it. The same applies while iterating on generators: the DB replays counterexamples recorded under the *old* generator, which can look like your new generator producing impossible values — `rm -rf .hegel` when a replayed failure makes no sense.)

12. **Adding tests to existing test files can collide with existing names.** Two recurring cases: E0255 when a test-function name matches an existing one (alias your imports or rename), and E0659 ambiguous `assert_eq!` when the surrounding file glob-imports `pretty_assertions` (add `use pretty_assertions::assert_eq;` inside your new module, or use fully-qualified `core::assert_eq!`).

13. **Lint-strict crates may need `#[allow(...)]` on hegel tests.** `#[hegel::test]`'s generated code can trip a crate's own strict lints (`disallowed_methods`, `doc_markdown`, `clippy::pedantic`); add the needed `#[allow(...)]` to the test module rather than assuming your test is wrong.

14. **There is no global case-count override** (no equivalent of `PROPTEST_CASES`). To run the exploratory 10x pass, temporarily edit `test_cases` in the attributes (or add `Settings` positionally), then revert.

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

- **`#[rule]`** methods are actions that can be applied. They take `&mut self` (or `&self`) and `TestCase`. Use `tc.assume()` to skip a rule when it doesn't apply (e.g., can't pop from an empty stack).
- **`#[invariant]`** methods are checked after every successful rule. They take `&self` (or `&mut self`) and `TestCase`. Invariants are optional — a machine with only rules is valid.
- Call `hegel::stateful::run(machine, tc)` from a `#[hegel::test]` to execute. `run` consumes the machine, so any end-of-test assertions must live in an invariant (or in data the machine writes elsewhere) — you cannot inspect the machine after `run` returns.

Practical notes:

- **Invariants also run once on the initial state**, before any rule is applied — don't assert things that only become true after the first rule.
- **Subjects that borrow.** If the type under test borrows other state (e.g. an incremental `Encoder<'a>` that mutably borrows its output buffer), it can't be stored in the machine alongside what it borrows. Store *owned inputs* (e.g. the fragments fed so far) in the machine instead, and reconstruct/finalize the borrowing object inside the rule or invariant that checks it.
- **Compounding rules are slow.** Rules that multiply state size (e.g. `mul` on an arbitrary-precision number, appending to a document every step) make late steps expensive. Prefer moderate `test_cases` for such machines rather than shrinking the generated values.
- **For incremental/lazily-validated subjects, prefer a check *rule* over an invariant.** Invariants run after every step; if the check itself perturbs the subject (forcing lazy revalidation, warming caches), per-step checking masks exactly the paths you want to test. A drawn `check` rule exercises both checked and unchecked interleavings.
- **Shrinking can take minutes** — for machines whose rules do I/O, but also for plain tests drawing large collections — this is normal, not a hang. To iterate faster on a known failure, temporarily reduce `test_cases`. `verbosity = Verbosity::Verbose` shows `Step N: <rule>` lines and assumption-skips, which is the way to see what a machine is doing.
- **Attributes compose.** `#[hegel::test]` works together with extra attributes like `#[cfg_attr(not(feature = "foo"), ignore)]`.
- **Distinguish slow shrinking from memory blowup.** A model that *materializes elements* (e.g. a `BTreeSet<u64>` mirroring ranges) can be OOM-killed by inputs the subject handles symbolically — a SIGKILL mid-run looks like the slow-shrink hang but needs the opposite fix: bound the model-facing input sizes (documented as protecting the model, per Generator Discipline), don't reduce `test_cases`.
- **Resource-owning machines and Drop order.** If the machine's fields hold resources with interdependent teardown (e.g. a write transaction and the database it came from), a wrong field order can deadlock in the generated drop — which looks exactly like hegel hanging mid-run. Order fields so dependents drop first, or wrap in `Option` and tear down explicitly in a rule.

### Pools

For tests that need to track dynamically created resources (accounts, handles, keys), use `Pool`. A pool hands out *generators* over its contents, which you draw from with `tc.draw` like any other generator:

```rust
use hegel::stateful::{Pool, pool};

struct MyTest {
    accounts: Pool<String>,
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
        let account = tc.draw(self.accounts.values_reusable()).clone();
        // ... do something with account; it stays in the pool
    }

    #[rule]
    fn delete_account(&mut self, tc: TestCase) {
        let account = tc.draw(self.accounts.values_consumed());
        // ... clean up account; it has been removed from the pool
    }
}

#[hegel::test]
fn test_my_system(tc: TestCase) {
    let test = MyTest {
        accounts: pool(&tc),
    };
    hegel::stateful::run(test, tc);
}
```

`Pool<T>` API:
- Drawing pool values goes through `tc.draw`, which requires `T: Debug` — the compile error appears at the draw site, not on `Pool<T>`. For non-`Debug` resources (file handles, savepoints), use `tc.draw_silent(...)` instead.
- `.add(value)` — Add a value to the pool
- `.values_reusable()` — Generator over `&T`: drawing yields a reference to a pool value without removing it (rejects the test case like `assume(false)` if the pool is empty)
- `.values_consumed()` — Generator over `T`: drawing removes a value from the pool and yields it by value (rejects if empty)
- `.is_empty()` / `.len()` — Inspect the pool

Because pool choices go through `tc.draw`, they are recorded in failure replays and shrink like any other draw.

## Keeping a Failing Test That Pins a Real Bug

When you keep a test failing to document a genuine bug (per the main skill), make it fail *deterministically*: boost the generator toward the counterexample's region (a `one_of!` branch with the exact shape) and raise that one test's `test_cases` so the failure reproduces on every run — a probabilistically-failing test reads as flaky. Add a KNOWN FAILURE comment naming the bug.

Two failure modes need special handling:
- **Process-aborting bugs** (stack overflow, OOM): no hegel report or failure-database entry is written. To identify the aborting input, run with `--nocapture` and stream the drawn values via `eprintln!` (not `tc.note()`, which only prints on replay); keep the reproducer `#[ignore]`d.
- **Engine panics** (a panic pointing inside hegeltest itself rather than your test or the library): re-run the test — the failure database replays and shrinking usually completes cleanly on the second run. Report the engine panic to hegel-rust with the test source.
