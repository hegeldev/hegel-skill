# Hegel Rust SDK Reference

## Setup

Add to `Cargo.toml`:

```toml
[dev-dependencies]
hegel = { git = "https://github.com/hegeldev/hegel-rust" }
```

Requires [`uv`](https://github.com/astral-sh/uv) installed and on PATH.

Run tests with `cargo test`. Hegel tests use `#[hegel::test]` in place of
`#[test]` and integrate directly with the standard Rust test runner.

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

### `Hegel::new().run()` (builder form)

```rust
use hegel::{Hegel, Verbosity};

#[test]
fn test_with_builder() {
    Hegel::new(|tc| {
        let n = tc.draw(generators::integers::<i32>());
        assert!(n == n);
    })
    .test_cases(500)
    .verbosity(Verbosity::Verbose)
    .run();
}
```

Or the shorthand:

```rust
use hegel::hegel;

#[test]
fn test_shorthand() {
    hegel(|tc| {
        let n = tc.draw(generators::integers::<i32>());
        assert!(n == n);
    });
}
```

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

The `Generator` trait import is needed for combinator methods (`.map()`,
`.filter()`, `.flat_map()`, `.boxed()`).

### Numeric Generators

**`generators::integers::<T>()`** — Generate any integer type

Supported types: `i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`,
`u128`, `isize`, `usize`.

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
- `.exclude_min()` — Make lower bound exclusive
- `.exclude_max()` — Make upper bound exclusive
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

// Always returns None
let n: Option<i32> = tc.draw(generators::none::<i32>());

// Always returns ()
let u: () = tc.draw(generators::unit());

// Sample uniformly from a fixed set
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

Functions `tuples2()` through `tuples12()`:

```rust
let pair: (i32, String) = tc.draw(generators::tuples2(
    generators::integers::<i32>(),
    generators::text(),
));

let triple: (bool, i32, f64) = tc.draw(generators::tuples3(
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
    generators::from_regex(r"[A-Z]{3}-[0-9]{3}").fullmatch());
```

- `.fullmatch()` — Require the pattern matches the entire string

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

Note: `.filter()` retries up to 3 times, then calls `tc.assume(false)`. Prefer
bounds over filters when possible.

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

### `compose!`

Build a generator from imperative code:

```rust
use hegel::compose;

let point_gen = compose!(|tc| {
    let x = tc.draw(generators::floats::<f64>().min_value(-100.0).max_value(100.0));
    let y = tc.draw(generators::floats::<f64>().min_value(-100.0).max_value(100.0));
    (x, y)
});

let (x, y): (f64, f64) = tc.draw(point_gen);
```

### `#[derive(Generator)]`

Auto-derive a generator for structs you own:

```rust
use hegel::Generator;
use hegel::generators::{self, Generator as _};

#[derive(Generator, Debug)]
struct User {
    name: String,
    age: u32,
    active: bool,
}

#[hegel::test]
fn test_user(tc: hegel::TestCase) {
    // Default generators for all fields:
    let user: User = tc.draw(UserGenerator::new());

    // Customize specific fields:
    let adult: User = tc.draw(UserGenerator::new()
        .with_age(generators::integers().min_value(18).max_value(120))
        .with_name(generators::from_regex(r"[A-Z][a-z]{2,15}").fullmatch()));
    assert!(adult.age >= 18);
}
```

The derive creates a `<Type>Generator` struct with:
- `::new()` — Uses default generators for all fields
- `.with_<field>(gen)` — Override a specific field's generator

Works with enums too, creating `<Enum><Variant>Generator` for each variant.

### `derive_generator!`

For types you don't own:

```rust
use hegel::derive_generator;
use hegel::generators::{self, Generator};

struct Point { x: f64, y: f64 }

derive_generator!(Point { x: f64, y: f64 });

#[hegel::test]
fn test_point(tc: hegel::TestCase) {
    let p: Point = tc.draw(PointGenerator::new()
        .with_x(generators::floats().min_value(-10.0).max_value(10.0))
        .with_y(generators::floats().min_value(-10.0).max_value(10.0)));
}
```

## Idiomatic Patterns

### Round-trip (serialize/deserialize)

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_json_round_trip(tc: hegel::TestCase) {
    let value = tc.draw(UserGenerator::new());
    let json = serde_json::to_string(&value).unwrap();
    let recovered: User = serde_json::from_str(&json).unwrap();
    assert_eq!(value, recovered);
}
```

### Invariant preservation

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_sort_preserves_length(tc: hegel::TestCase) {
    let mut v: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    let original_len = v.len();
    v.sort();
    assert_eq!(v.len(), original_len);
}
```

### Oracle / reference implementation

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_my_sort_matches_std(tc: hegel::TestCase) {
    let v: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    let mut expected = v.clone();
    expected.sort();
    let actual = my_sort(&v);
    assert_eq!(actual, expected);
}
```

### No-crash / robustness

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_parse_doesnt_panic(tc: hegel::TestCase) {
    let input: String = tc.draw(generators::text());
    // Just verify it doesn't panic — any result is fine
    let _ = MyParser::parse(&input);
}
```

### Dependent generation

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
use hegel::Generator;
use hegel::generators::{self, Generator as _};

#[derive(Generator, Debug, Clone, PartialEq)]
struct Config {
    max_retries: u32,
    timeout_ms: u64,
    name: String,
}

#[hegel::test]
fn test_config_merge(tc: hegel::TestCase) {
    let base = tc.draw(ConfigGenerator::new());
    let override_cfg = tc.draw(ConfigGenerator::new());
    let merged = base.merge(&override_cfg);
    // Property: merged config should have override's values
    assert_eq!(merged.name, override_cfg.name);
}
```

### Evolving a unit test into a PBT

Before (unit test):

```rust
#[test]
fn test_reverse() {
    assert_eq!(reverse(&[1, 2, 3]), vec![3, 2, 1]);
    assert_eq!(reverse(&[]), Vec::<i32>::new());
    assert_eq!(reverse(&[42]), vec![42]);
}
```

After (property-based test):

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_reverse_involution(tc: hegel::TestCase) {
    let v: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    assert_eq!(reverse(&reverse(&v)), v);
}

#[hegel::test]
fn test_reverse_preserves_length(tc: hegel::TestCase) {
    let v: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    assert_eq!(reverse(&v).len(), v.len());
}

#[hegel::test]
fn test_reverse_preserves_elements(tc: hegel::TestCase) {
    let mut v: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    let mut reversed = reverse(&v);
    v.sort();
    reversed.sort();
    assert_eq!(v, reversed);
}
```

### Commutativity

```rust
use hegel::generators::{self, Generator};

#[hegel::test]
fn test_set_union_commutes(tc: hegel::TestCase) {
    let a: HashSet<i32> = tc.draw(generators::hashsets(generators::integers()));
    let b: HashSet<i32> = tc.draw(generators::hashsets(generators::integers()));
    assert_eq!(a.union(&b).collect::<HashSet<_>>(),
               b.union(&a).collect::<HashSet<_>>());
}
```

## Gotchas

1. **Import `Generator` trait for combinators.** `.map()`, `.filter()`,
   `.flat_map()`, and `.boxed()` require `use hegel::generators::Generator`.
   Without the import, these methods won't be available.

2. **`#[hegel::test]` replaces `#[test]`, not both.** Don't write
   `#[test] #[hegel::test]` — the hegel macro already generates the test
   attribute.

3. **Add `.hegel/` to `.gitignore`.** Hegel creates a `.hegel/` directory for
   caching the server binary and storing the database of previous failures. Add
   it to `.gitignore`.

4. **Float defaults include NaN and infinity.** `generators::floats::<f64>()`
   with no bounds generates NaN and infinity by default. If your code doesn't
   handle these, use `.allow_nan(false)` and/or `.allow_infinity(false)` — but
   consider whether the code *should* handle them first.

5. **Type annotations are required for numeric generators.**
   `generators::integers()` won't compile — you must write
   `generators::integers::<i32>()` (or whatever type you need).

6. **Excessive assume/filter rejections fail the test.** If `tc.assume()` or
   `.filter()` rejects too many inputs, Hegel gives up. Restructure your
   generators to produce valid inputs directly.

7. **`note()` only prints on the final replay.** Don't rely on `tc.note()` for
   progress logging — it only appears when displaying the minimal
   counterexample.

8. **`target()` is not yet available** in the Rust SDK. It is planned for a
   future release.
