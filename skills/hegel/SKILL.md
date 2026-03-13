---
name: hegel
description: >
  Write property-based tests using Hegel. Triggers on: "property-based tests",
  "PBT", "hegel tests", "fuzz this code", "test with random inputs",
  "generative tests", "find edge cases", "shrink counterexample",
  "test properties", "randomized testing"
---

# Hegel: Property-Based Testing

Hegel is a universal property-based testing framework. It provides SDKs for
Rust, Python, TypeScript, Go, C++, and OCaml, all powered by Hypothesis. Tests
integrate with standard test runners (`cargo test`, `pytest`, `jest`, etc.).
Hegel generates random inputs for your code and automatically shrinks failing
cases to minimal counterexamples.

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
- **Non-trivial** — they test real behavior, not tautologies
- **Falsifiable** — a buggy implementation could actually violate them

Write one test per property. Don't cram multiple properties into one test.

### 4. Check for Existing Tests to Evolve

Before writing tests from scratch, check if existing unit tests or example-based
tests can be evolved into property-based tests. Load `references/evolving-tests.md`
for guidance. This is often the fastest path to valuable PBTs.

### 5. Write the Tests

For each property:

1. Choose the **simplest possible generators** — start with no bounds
2. Draw values using `tc.draw()`
3. Run the code under test
4. Assert the property

### 6. Run and Reflect

Run the tests. When a test fails, ask:

- **Is this a real bug?** If the code violates its own contract, fix the code.
- **Is the property unsound?** If you asserted something the code never promised,
  fix the test.
- **Is the generator too broad?** Only if the failing input is genuinely outside
  the function's domain, add constraints. Investigate before constraining.

## Property Categories

Use this taxonomy to identify what to test. Not every category applies to every
function — pick the ones supported by evidence.

| Category | Description | Example |
|----------|-------------|---------|
| **Round-trip** | encode then decode recovers the original | `deserialize(serialize(x)) == x` |
| **Idempotence** | applying twice equals applying once | `sort(sort(xs)) == sort(xs)` |
| **Commutativity** | order of operations doesn't matter | `a + b == b + a` |
| **Invariant preservation** | an operation maintains a structural property | `insert into BST preserves ordering` |
| **Oracle / reference impl** | compare against a known-correct implementation | `my_sort(xs) == xs.sort()` |
| **Monotonicity** | more input means more (or equal) output | `len(xs ++ ys) >= len(xs)` |
| **Bounds / contracts** | output stays within documented limits | `clamp(x, lo, hi)` is in `[lo, hi]` |
| **No-crash / robustness** | function handles all valid inputs without panicking | `parse(arbitrary_string)` doesn't panic |
| **Equivalence** | two implementations produce the same result | `iterative_fib(n) == recursive_fib(n)` |
| **Model-based** | operations on real system match a simplified model | `HashMap ops match Vec<(K,V)> model` |

## Choosing Properties

Properties must be **evidence-based**. Find evidence in:

- **Type signatures**: A function `fn merge(a: Vec<T>, b: Vec<T>) -> Vec<T>` implies
  the output length might equal the sum of input lengths.
- **Docstrings and comments**: "Returns a sorted list" directly gives you an invariant.
- **Assertions and debug_asserts in the source**: These are properties the author
  already identified.
- **Usage patterns**: If callers always check `result.is_ok()`, that's evidence
  the function shouldn't panic on valid input.
- **Existing tests**: Unit tests often encode specific instances of general properties.

**Do not** invent properties from thin air. If you can't find evidence for a
property, it probably isn't one.

## Generator Discipline

This is the most important section. The single most common mistake agents make
when writing property-based tests is **over-constraining generators**. This
defeats the entire purpose of PBT.

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

Don't narrow ranges to "avoid edge cases." Edge cases are exactly what PBT is
for. If a function claims to work on all `i32` values, test it on all `i32`
values — including `i32::MIN`, `i32::MAX`, `0`, `-1`, and `1`.

### Don't Add `.min_size(1)` by Default

Unless the function's contract explicitly requires non-empty input, test with
empty collections too. If a function panics on an empty vec, that might be a bug
worth knowing about.

### When a Test Fails on Extreme Values

Your first reaction should be: **is this a real bug?**

- If the function's documentation says it handles all integers but it overflows
  on `i32::MAX`, that's a bug in the code, not in your test.
- Only add bounds after investigating and confirming the input is outside the
  function's documented domain.

### When to Add Constraints

Add generator bounds **only** when:

1. **The function's contract explicitly excludes some inputs.** For example,
   `fn sqrt(x: f64)` documents that `x >= 0` is required.
2. **You need to avoid undefined behavior.** For example, division by zero.
3. **A test failure has been investigated** and confirmed to be outside the
   function's domain.

### Prefer `tc.assume()` Over Generator Bounds for Multi-Value Constraints

When a constraint involves relationships between multiple generated values,
use `tc.assume()`:

```rust
let a = tc.draw(generators::integers::<i32>());
let b = tc.draw(generators::integers::<i32>());
tc.assume(a != b);  // constraint relates two values
```

### `.filter()` and `.allow_nan(false)` Are Fine When Justified

Use `.filter()` when it matches the function's documented preconditions:

```rust
// OK: the function documents that it requires non-empty strings
generators::text().filter(|s| !s.is_empty())
```

Use `.allow_nan(false)` when the function explicitly doesn't handle NaN:

```rust
// OK: comparison functions don't work with NaN by definition
generators::floats::<f64>().allow_nan(false)
```

## Common Mistakes

1. **Over-constraining generators** — Adding bounds "just in case." This hides
   bugs and makes tests less valuable. See Generator Discipline above.

2. **Testing trivial properties** — `assert!(x == x)` or `assert!(vec.len() >= 0)`
   test nothing. Every property should be falsifiable by a buggy implementation.

3. **Using the implementation as the oracle** — If your test calls the same
   function to compute the expected result, it can never fail. Use an independent
   reference implementation, a simpler algorithm, or a structural property.

4. **Generating too broadly then filtering almost everything** — If `.filter()`
   or `tc.assume()` rejects most inputs, Hegel will give up. Restructure your
   generators instead (e.g., use `.map()` or dependent generation).

5. **Testing too many things in one test** — Each test should check one property.
   If a test has five asserts testing different properties, split it into five tests.

## Quick Setup

### Rust

```toml
# Cargo.toml
[dev-dependencies]
hegel = { git = "https://github.com/hegeldev/hegel-rust" }
```

Requires [`uv`](https://github.com/astral-sh/uv) on PATH. Run with `cargo test`.
