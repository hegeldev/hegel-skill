---
name: hegel
description: >
  Write property-based tests using Hegel. Triggers on: "property-based tests",
  "PBT", "hegel tests", "test with random inputs",
  "generative tests",  "test properties", "randomized testing"
---

# Hegel: Property-Based Testing

Hegel is a universal family of property-based testing, supporting a variety of
languages all powered by Hypothesis. Tests integrate with standard test runners
(e.g. `cargo test`, `pytest`, `jest`, etc.). Hegel generates random inputs
for your code and automatically shrinks failing cases to minimal counterexamples.

## Workflow

Follow these steps when writing property-based tests.

### 1. Detect Language and Load References

Identify the project language from build files:

| File | Language | Reference |
|------|----------|-----------|
| `Cargo.toml` | Rust | `references/rust.md` |

Load the corresponding reference file for API details and idiomatic patterns.

### 2. Explore the Code Under Test

Before writing any test, understand what you're testing:

- **Read the source code** of the function/module under test
- **Read existing tests** to understand expected behavior and edge cases
- **Read docstrings, comments, and type signatures** for documented contracts
- **Read usage sites** to see how callers use the code and what they expect

The goal is to find *evidence* for properties, not to invent them.

### 3. Identify Valuable Properties

Look for properties that are:

- **Grounded in evidence** from the code, docs, or usage patterns
- **Non-trivial** — they test real behavior, not tautologies, and do not duplicate the code being tested
- **Falsifiable** — a buggy implementation could actually violate them

Write one test per property. Don't cram multiple properties into one test.

### 4. Check for Existing Tests to Evolve or Port

Before writing tests from scratch, **always** check existing tests:

- **Existing PBTs in another framework** (proptest, quickcheck, etc.) should be
  ported to hegel. Load `references/porting.md` for general guidance and the
  language-specific porting reference (e.g., `references/porting-rust.md`).
  Don't carry over narrow generator bounds from the old framework — use broader
  generators unless bounds are justified by the function's contract.
- **Unit tests and example-based tests** can often be evolved into PBTs. Load
  `references/evolving-tests.md` for guidance. Tests with hardcoded seeds,
  parameterized examples, or multiple similar test cases are prime candidates.
- **Tests that use `rand` with fixed seeds** are especially good candidates —
  the randomness should come from hegel instead so failures produce shrinkable
  counterexamples.

When you evolve an existing test, **modify the existing test file** rather than
creating a new one. Add hegel tests alongside (or replacing) the existing tests
in the same file where the original tests live. Do not create a separate
`test_hegel.rs` or similar — property-based tests are tests like any other and
belong with the code they're testing.

### 5. Write the Tests

For each property:

1. **Add tests to the appropriate existing test file.** If there's already a
   `test_foo.rs` covering the module, add hegel tests there. Only create a new
   file if no relevant test file exists.
2. Choose the **simplest possible generators** — start with no bounds, unless
   bounds are logically necessary (e.g. if a number has to be non-zero it's
   fine to force it to be, but lists should not have `max_size` set unless there
   is a compelling correctness reason to set them or poor performance has been
   observed when actually running the test)
3. Draw values using `tc.draw()`
4. Run the code under test
5. Assert the property

### 6. Run and Reflect

Run the tests. When a test fails, ask:

- **Is this a real bug?** If the code violates its own contract, flag the bug to the user and ask what to do, or fix the code if instructed to do so.
- **Is the property unsound?** If you asserted something the code never promised, fix the test.
- **Is the generator too broad?** Only if the failing input is genuinely outside the function's domain, add constraints. Investigate before constraining.

## Property Categories

Use this taxonomy to identify what to test. Not every category applies to every
function — pick the ones supported by evidence.

| Category | Description | Example |
|----------|-------------|---------|
| **Round-trip** | encode then decode recovers the original | `deserialize(serialize(x)) == x` |
| **Idempotence** | applying twice equals applying once | `sort(sort(xs)) == sort(xs)` |
| **Commutativity** | order of operations doesn't matter | `a + b == b + a` or `f(g(x)) == g(f(x))` |
| **Invariant preservation** | an operation maintains a structural property | `insert into BST preserves ordering` |
| **Oracle / reference impl** | compare against a known-correct implementation | `my_sort(xs) == xs.sort()`, or comparing against an unoptimised implementation |
| **Monotonicity** | more input means more (or equal) output | `len(xs ++ ys) >= len(xs)` |
| **Bounds / contracts** | output stays within documented limits | `clamp(x, lo, hi)` is in `[lo, hi]` |
| **No-crash / robustness** | function handles all valid inputs without panicking | `parse(arbitrary_string)` doesn't panic |
| **Equivalence** | two implementations produce the same result | `iterative_fib(n) == recursive_fib(n)` |
| **Model-based** | operations on real system match a simplified model | `HashMap ops match Vec<(K,V)> model` |

## Choosing Properties

Properties must be **evidence-based**. Find evidence in:

- **Names and Type signatures**: A function `fn merge(a: Vec<T>, b: Vec<T>) -> Vec<T>` implies the output length might equal the sum of input lengths.
- **Docstrings and comments**: "Returns a sorted list" directly gives you an invariant.
- **Assertions and debug_asserts in the source**: These are properties the author already identified, and do not need to be duplicated in the tests, but may suggest other invariants.
- **Usage patterns**: If callers always assume a result is non-empty, assert that the result is always non-empty.
- **Existing tests**: Unit tests often encode specific instances of general properties.

Err on the side of creating more properties rather than fewer, and if they fail investigate whether the failure is legitimate behaviour or not.

## Generator Discipline

A common mistake agents make when writing property-based tests is **over-constraining generators**.
This leads to tests that are weaker than they need to be.

### Start With No Bounds

If the function accepts any `i32`, use:

```rust
generators::integers::<i32>()  // no min_value, no max_value
```

Do NOT preemptively write:

```rust
generators::integers::<i32>().min_value(0).max_value(100)  // WRONG unless justified
```

### Edge Cases Are the Point

Don't narrow ranges to "avoid edge cases." Edge cases are exactly what PBT is for. If a function claims to work on all `i32` values, test it on all `i32` values — including `i32::MIN`, `i32::MAX`, `0`, `-1`, and `1`.

### Don't Add `.min_size(1)` by Default

Unless the function's contract explicitly requires non-empty input, test with empty collections too. If a function panics on an empty vec, that might be a bug worth knowing about.

### When a Test Fails on Extreme Values

Your first reaction should be: **is this a real bug?**

You should assume that it is unless you have strong evidence that it is not. If in doubt, ask the user.

- If the function's documentation says it handles all integers but it overflows on `i32::MAX`, that's a bug in the code, not in your test.
- Only add bounds after investigating and confirming the input is outside the function's documented domain.

### When to Add Constraints

Add generator bounds **only** when:

1. **The function's contract explicitly excludes some inputs.** For example, `fn sqrt(x: f64)` documents that `x >= 0` is required.
2. **You need to avoid undefined behavior.** For example, division by zero.
3. **A test failure has been investigated** and confirmed to be outside the function's domain.

### Prefer `tc.assume()` Over Generator Bounds for Multi-Value Constraints

When a constraint involves relationships between multiple generated values, use `tc.assume()`:

```rust
let a = tc.draw(generators::integers::<i32>());
let b = tc.draw(generators::integers::<i32>());
tc.assume(a != b);  // constraint relates two values
```

However it is better to avoid these altogether when you can. 

e.g.

```rust
let a = tc.draw(generators::integers::<i32>());
let b = tc.draw(generators::integers::<i32>().min_value(a));
```

is better than

```rust
let a = tc.draw(generators::integers::<i32>());
let b = tc.draw(generators::integers::<i32>());
tc.assume(a <= b)
```

Even better is:

```rust
let mut a = tc.draw(generators::integers::<i32>());
let mut b = tc.draw(generators::integers::<i32>());
if (a > b) {
    (a, b) = (b, a);
}
```

## Handling Randomness in Code Under Test

When the code under test requires an RNG (e.g., `fn sample(&self, rng: &mut impl Rng)`),
**do not** create a seeded RNG like `ChaCha8Rng::seed_from_u64(seed)` with a
hegel-generated seed. This defeats shrinking — hegel can only shrink the seed
integer, not the actual random decisions the RNG makes.

Instead, use hegel's `rand` feature to get a hegel-controlled RNG. See the
language-specific reference for API details.

### Rand version mismatch

Hegel's `rand` feature uses rand 0.9. If the project uses an older version of
rand (e.g., 0.8), the RNG traits will be incompatible. In this case, ask the
user whether they'd like to upgrade the project's rand dependency to 0.9. This
is usually straightforward (the main API changes are `gen_range` -> `random_range`,
`gen::<T>()` -> `random::<T>()`, `thread_rng()` -> `rng()`,
`from_entropy` -> `from_os_rng`). Do not silently fall back to seeded ChaCha —
that defeats the purpose.

### Two modes: artificial vs true randomness

`generators::randoms()` has two modes:

- **Default (artificial randomness):** Every random decision goes through hegel,
  enabling fine-grained shrinking of individual random values. This is the best
  option for most code.

- **`generators::randoms().use_true_random()`:** Generates a single seed via
  hegel, then creates a real `StdRng` from it. Hegel can only shrink the seed,
  not individual random decisions. Use this when the code under test does
  **rejection sampling** or otherwise depends on the RNG producing
  statistically random-looking output. Artificial randomness can cause rejection
  loops to hang because the controlled byte sequences don't look random enough.

**How to choose:** Start with the default. If tests hang or time out because the
code does rejection sampling internally, switch to `.use_true_random()`.

### Refactoring concrete RNG types

If the code under test takes a concrete RNG type (e.g., `rng: &mut ChaCha8Rng`)
rather than a trait bound, consider whether it should be refactored to accept
`impl Rng` or `&mut dyn RngCore` instead. This is both better API design and
makes the code testable with hegel's random generator. Suggest this refactoring
to the user.

## Common Mistakes

1. **Over-constraining generators** — Adding bounds "just in case." This hides bugs and makes tests less valuable. See Generator Discipline above.
2. **Testing trivial properties** — `assert!(x == x)` or `assert!(vec.len() >= 0)` test nothing. Every property should be falsifiable by a buggy implementation.
3. **Using the implementation as the oracle** — If your test calls the same function to compute the expected result, it can never fail. Use an independent reference implementation (do not just copy the code to write this!), a simpler algorithm, or a structural property.
4. **Generating too broadly then filtering almost everything** — If `.filter()` or `tc.assume()` rejects most inputs, Hegel will give up. Restructure your generators instead (e.g., use `.map()` or dependent generation).
5. **Creating a separate test file for hegel tests** — Property-based tests belong alongside the existing tests for the same code. Don't put them in `test_hegel.rs` or `test_properties.rs` — add them to the existing test files.
6. **Using manually seeded RNGs** — Don't generate a seed with hegel then create `ChaCha8Rng::seed_from_u64(seed)`. Use `generators::randoms()` with the `rand` feature so hegel controls the random decisions and can shrink them. See "Handling Randomness" above.

## Quick Setup

### Rust

```toml
# Cargo.toml
[dev-dependencies]
hegel = { git = "https://github.com/hegeldev/hegel-rust" }
```

If the code under test uses `rand`:

```toml
[dev-dependencies]
hegel = { git = "https://github.com/hegeldev/hegel-rust", features = ["rand"] }
```

Requires [`uv`](https://github.com/astral-sh/uv) on PATH. Run with `cargo test`.
