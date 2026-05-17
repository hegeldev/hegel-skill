# Hegel Rust Extras Integrations

Hegel ships generators for several third-party crates under `hegel::extras`. Each integration lives behind its own Cargo feature flag and is **off by default**.

**Load this file when** the code under test produces or consumes values from `chrono`, `jiff`, `serde_json`, or `rand`. If none of those crates appear in the project, you don't need any of this.

## Enabling an integration

Add hegel with the relevant features:

```bash
cargo add --dev hegeltest --features chrono,jiff,serde_json,rand
```

Or in `Cargo.toml`:

```toml
[dev-dependencies]
hegeltest = { version = "...", features = ["chrono", "jiff", "serde_json", "rand"] }
```

Only enable the features you actually need — there's no benefit to enabling `chrono` if the project uses `jiff`, and vice versa.

The idiomatic import for each integration is a module alias:

```rust
use hegel::extras::chrono as chrono_gs;
use hegel::extras::jiff as jiff_gs;
use hegel::extras::serde_json as json_gs;
use hegel::extras::rand as rand_gs;
```

## `chrono` (feature `chrono`)

Generators for [`chrono`](https://docs.rs/chrono):

| Generator | Yields |
|-----------|--------|
| `chrono_gs::naive_dates()` | `chrono::NaiveDate` |
| `chrono_gs::naive_times()` | `chrono::NaiveTime` |
| `chrono_gs::naive_datetimes()` | `chrono::NaiveDateTime` |
| `chrono_gs::naive_weeks()` | `chrono::NaiveWeek` |
| `chrono_gs::time_deltas()` | `chrono::TimeDelta` |
| `chrono_gs::fixed_offsets()` | `chrono::FixedOffset` |
| `chrono_gs::weekday_sets()` | `chrono::WeekdaySet` |
| `chrono_gs::datetimes()` | `chrono::DateTime<FixedOffset>` |

```rust
use hegel::extras::chrono as chrono_gs;

#[hegel::test]
fn my_test(tc: hegel::TestCase) {
    let d: chrono::NaiveDate = tc.draw(chrono_gs::naive_dates());
    // ...
}
```

## `jiff` (feature `jiff`)

Generators for [`jiff`](https://docs.rs/jiff):

| Generator | Yields |
|-----------|--------|
| `jiff_gs::dates()` | `jiff::civil::Date` |
| `jiff_gs::times()` | `jiff::civil::Time` |
| `jiff_gs::datetimes()` | `jiff::civil::DateTime` |
| `jiff_gs::timestamps()` | `jiff::Timestamp` |
| `jiff_gs::spans()` | `jiff::Span` |
| `jiff_gs::signed_durations()` | `jiff::SignedDuration` |
| `jiff_gs::offsets()` | `jiff::tz::Offset` |
| `jiff_gs::zoneds()` | `jiff::Zoned` |

```rust
use hegel::extras::jiff as jiff_gs;

#[hegel::test]
fn my_test(tc: hegel::TestCase) {
    let z: jiff::Zoned = tc.draw(jiff_gs::zoneds());
    // ...
}
```

## `serde_json` (feature `serde_json`)

Generators for [`serde_json`](https://docs.rs/serde_json):

| Generator | Yields |
|-----------|--------|
| `json_gs::numbers()` | `serde_json::Number` |
| `json_gs::values()` | `serde_json::Value` (arbitrary JSON values, recursive) |
| `json_gs::raw_values()` | `Box<serde_json::value::RawValue>` (requires extra feature `serde_json_raw_value`) |

The integration also implements `DefaultGenerator` for `serde_json::Number`, `serde_json::Value`, and `serde_json::Map<String, Value>`, so `gs::default::<serde_json::Value>()` works directly.

Particularly useful for testing serializers, parsers, and any code that round-trips through JSON:

```rust
use hegel::extras::serde_json as json_gs;

#[hegel::test]
fn json_roundtrip(tc: hegel::TestCase) {
    let v: serde_json::Value = tc.draw(json_gs::values());
    let s = serde_json::to_string(&v).unwrap();
    let parsed: serde_json::Value = serde_json::from_str(&s).unwrap();
    assert_eq!(v, parsed);
}
```

## `rand` (feature `rand`)

`rand_gs::randoms()` is the hegel-controlled RNG generator — use it when the code under test takes an `impl Rng` (or similar) and you want hegel to shrink the random decisions, not just a seed.

```rust
use hegel::extras::rand as rand_gs;

// Default: artificial randomness — every random decision is shrinkable
let mut rng = tc.draw(rand_gs::randoms());

// True randomness — single shrinkable seed, real StdRng output
let mut rng = tc.draw(rand_gs::randoms().use_true_random(true));
```

The returned `HegelRandom` implements `rand::RngCore` (rand 0.9).

**Default mode** routes every `next_u32`/`next_u64`/`fill_bytes` call through hegel, so the shrinker can minimize individual random decisions. Best for most code.

**`use_true_random()` mode** generates a single seed via hegel then creates a real `StdRng`. Use this when the code under test does rejection sampling or other algorithms that need statistically random-looking output — artificial randomness can cause these to loop indefinitely.

**Rand version compatibility:** hegel uses rand 0.9. If the project uses rand 0.8, the traits are incompatible. Ask the user to upgrade rand (main changes: `gen_range` -> `random_range`, `gen::<T>()` -> `random::<T>()`, `thread_rng()` -> `rng()`, `from_entropy` -> `from_os_rng`). Do not fall back to `ChaCha8Rng::seed_from_u64(hegel_seed)` — that defeats shrinking.
