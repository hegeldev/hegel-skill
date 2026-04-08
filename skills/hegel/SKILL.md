---
name: hegel
description: >
  Write property-based tests using Hegel across Rust and Go projects. Use this
  skill whenever the user asks to write tests, add test coverage, or improve
  testing for functions, modules, or libraries — especially when the code has
  properties like round-trips, invariants, or contracts that hold across many
  inputs. Also triggers on: "property-based tests", "PBT", "hegel", "fuzz",
  "generative tests", "randomized testing", "test with random inputs",
  "shrinking", or when existing tests use proptest, quickcheck, rapid, or gopter.
---

# Hegel: Property-Based Testing

Hegel is a family of property-based testing libraries supporting multiple languages, powered by Hypothesis. Tests integrate with standard language test runners. Hegel generates random inputs for your code and automatically shrinks failing cases to minimal counterexamples.

Even when PBTs add modest line coverage over unit tests, their value is in exercising combinations and boundary conditions that humans don't think to write by hand.

**Code examples in this file use Python-like pseudocode to illustrate concepts.** For exact API and syntax, load the language-specific reference (see step 1 of the workflow).

## Workflow

Follow these steps when writing property-based tests.

### 1. Load the Language Reference

Determine the project language and load the corresponding reference from `references/<language>/reference.md` for API details and idiomatic patterns.

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

See the **Property Catalogue** below for a taxonomy of what to look for.

### 4. Check for Existing Tests to Evolve or Port

Before writing tests from scratch, check what already exists.

**Existing PBTs in another framework** (proptest, quickcheck, rapid, gopter, etc.) should be ported to hegel. Load the language-specific porting reference (`references/<language>/porting.md`). Key things to know about hegel when porting:

- **Hegel is imperative.** Most PBT libraries declare what to generate in a function signature or strategy combinator. In hegel, your test receives a test case handle and calls `tc.draw()` whenever it needs a value — you can draw conditionally, in loops, and have later draws depend on earlier values without needing `flat_map`.
- **Shrinking is automatic.** Hegel's shrinking is handled server-side by Hypothesis. You don't implement shrink logic or define shrinking strategies.
- **Standard assertions.** Use the language's normal assertion mechanism. No special `prop_assert!` or return-a-bool pattern needed.
- **Broaden your generators.** Many existing PBTs use narrow input ranges because shrinking was slow or unreliable. Hegel's shrinking is more robust — try broader generators than the originals.

**Unit tests and example-based tests** can often be evolved into PBTs. Tests with hardcoded seeds, parameterized examples, or multiple similar test cases are prime candidates. Load `references/evolving-tests.md` for detailed guidance on recognizing what property a unit test is hiding. If you can't immediately see the right property, start by parameterizing the test — replace concrete values with generated ones and keep a simple oracle. You can refine the property later.

**Tests that use `rand` with fixed seeds** are especially good candidates — the randomness should come from hegel instead so failures produce shrinkable counterexamples.

When you evolve an existing test, **modify the existing test file** rather than creating a new one. Property-based tests are tests like any other and belong with the code they're testing. Do not create a separate file for hegel tests.

### 5. Write the Tests

For each property:

1. **Add tests to the appropriate existing test file.** Only create a new file if no relevant test file exists.
2. Choose the **simplest possible generators** — see Generator Discipline below.
3. Draw values, run the code under test, and assert the property.

### 6. Run and Reflect

Run the tests. When a test fails, ask:

- **Is this a real bug?** If the code violates its own contract, flag the bug to the user and ask what to do, or fix the code if instructed to do so.
- **Is the property unsound?** If you asserted something the code never promised, fix the test.
- **Is the generator too broad?** Only if the failing input is genuinely outside the function's domain, add constraints. Investigate before constraining.

### When NOT to Write PBTs

Property-based tests aren't always the right tool. Prefer unit tests when:

- **The test checks exact output.** `assert render(doc) == "<html>..."` depends on a specific output format — there's no general property to check.
- **Complex setup dominates.** Tests requiring database state, network mocks, or elaborate fixtures are hard to parameterize.
- **The test checks specific error messages.** Exact error string checks are a unit test concern. PBTs are better for testing that errors are *raised*, not what they *say*.
- **No property is apparent.** If you can't find a meaningful property after reading the code, don't force it. A good unit test beats a contrived PBT.

## Property Catalogue

Use this catalogue to identify what to test. Not every category applies to every function — pick the ones supported by evidence from the code.

The first five patterns are ordered by how often they've found real bugs in practice.

### Tier 1: High-Value Patterns

**Model tests** — For any data structure, the highest-value first test is a **stateful model test**: define rules for each operation (insert, remove, get, etc.), run them against both the library under test and a known-good reference (the "model"), and assert they agree after every operation. Use hegel's stateful testing support (see the language reference) rather than hand-rolling the operation loop.

The exact syntax varies significantly by language — check the language reference for the stateful testing API. Conceptually, a model test looks like:

```pseudocode
state_machine MyMapTest:
    subject = MyMap()
    model = HashMap()

    rule insert():
        k = tc.draw(integers())
        v = tc.draw(integers())
        subject.insert(k, v)
        model.insert(k, v)

    rule remove():
        k = tc.draw(integers())
        subject.remove(k)
        model.remove(k)

    rule get():
        k = tc.draw(integers())
        assert subject.get(k) == model.get(k)

    invariant agrees:
        assert subject == model
```

Choose the right model: `Vec` for sequential containers, `HashMap` for hash maps, `BTreeMap`/sorted map for ordered maps, `HashSet`/set for unordered sets.

**Idempotence tests** — Any normalization, case conversion, or formatting function should satisfy `f(f(x)) == f(x)`. Use full Unicode text generators (not ASCII-only) because Unicode edge cases like `ß` -> `SS` and combining characters are where bugs hide.

```pseudocode
s = tc.draw(text())
once = normalize(s)
twice = normalize(once)
assert once == twice
```

**Parse robustness** — Parsers (`from_str`, `parse`, `decode`) should handle all input without panicking. The property is simple: it should never crash, even on garbage input.

```pseudocode
s = tc.draw(text())
_ = MyType.parse(s)  # should return an error, never panic
```

**Roundtrip tests** — `parse(format(x)) == x` for any serialize/deserialize pair. Test with the full input domain. Bugs hide at zero (scientific notation edge cases), large integers (precision loss through f64 for values > 2^53), and unusual string content.

```pseudocode
n = tc.draw(integers())
s = format(n)
assert parse(s) == n
```

**Boundary value tests** — Integer boundary values (`MIN`, `MAX`, `0`) are where overflow bugs hide. Don't add bounds to avoid them — they ARE the test. Negating `MIN` overflows, intermediate products overflow, GCD/LCM computations overflow on boundary inputs.

```pseudocode
a = tc.draw(integers())  # includes MIN, MAX, 0
b = tc.draw(integers())
tc.assume(b != 0)
result = my_numeric_op(a, b)  # should not overflow/panic
```

### Tier 2: General Property Categories

| Category | Description | Example |
|----------|-------------|---------|
| **Commutativity** | order of operations doesn't matter | `a + b == b + a` or `f(g(x)) == g(f(x))` |
| **Invariant preservation** | an operation maintains a structural property | `insert into BST preserves ordering` |
| **Oracle / reference impl** | compare against a known-correct implementation | `my_sort(xs) == std_sort(xs)` |
| **Monotonicity** | more input means more (or equal) output | `len(xs ++ ys) >= len(xs)` |
| **Bounds / contracts** | output stays within documented limits | `clamp(x, lo, hi)` is in `[lo, hi]` |
| **No-crash / robustness** | function handles all valid inputs without panicking | `parse(arbitrary_string)` doesn't crash |
| **Equivalence** | two implementations produce the same result | `iterative_fib(n) == recursive_fib(n)` |
| **Consistency** | related APIs in the same library agree | `string_width(s) == sum(char_width(c) for c in s)` |
| **Large input sizes** | exercise deep structure paths that small inputs miss | draw size separately, force 50-200+ elements for trees/tries |
| **Feature flag testing** | non-default features are often less tested | enable SIMD, nightly, or experimental features and run tests |

### Bug Patterns by Category

| Category | What to look for |
|---|---|
| **Integer overflow** | Boundary values (MIN, MAX, 0) in arithmetic, GCD, negation, display |
| **Idempotence failure** | Case conversion / normalization with Unicode (ß -> SS), word splitting on case transitions |
| **Precision loss** | Numbers routed through f64 lose precision for integers > 2^53 |
| **Roundtrip failure** | Format/parse on edge cases: zero, empty strings, unusual path components |
| **Parse panic** | `from_str` delegates to a constructor that panics instead of returning Err |
| **Stale state** | Update operations that modify one index but don't clean up the old entry in another |
| **Unicode line breaks** | `\u{85}` (NEL), `\u{2028}` (LS), `\u{2029}` (PS) treated inconsistently as line breaks |
| **SIMD divergence** | SIMD code path produces different results than the scalar fallback |
| **Deep structure bugs** | Traversal that only fails when data structure has multiple internal levels (50-200+ elements) |

### Choosing Properties

Properties must be **evidence-based**. Find evidence in:

- **Names and type signatures**: A function `merge(a: List, b: List) -> List` implies the output length might equal the sum of input lengths.
- **Docstrings and comments**: "Returns a sorted list" directly gives you an invariant.
- **Assertions and debug checks in the source**: These are properties the author already identified — they may suggest other invariants.
- **Usage patterns**: If callers always assume a result is non-empty, assert that.
- **Existing tests**: Unit tests often encode specific instances of general properties.

Err on the side of creating more properties rather than fewer, and if they fail investigate whether the failure is legitimate behavior or not.

**Beware of properties that seem universal but aren't.** Read the docs carefully before asserting a property. Examples from real testing:
- Grapheme-based string reverse is NOT an involution (`reverse(reverse("\n\r")) != "\n\r"` because `\r\n` is one grapheme cluster while `\n\r` is two).
- A method called `difference` might mean symmetric difference (A triangle B), not set difference (A \ B) — check the docs.
- A function documented as "returns the largest key <= k" means <=, not <.

When a property fails, investigate whether it's a real bug or a genuine edge case in the domain. A weaker property often still holds.

## Generator Discipline

The most common mistake when writing property-based tests is **over-constraining generators**. Broad generators find more bugs because they explore inputs the developer didn't anticipate. Constrained generators give a false sense of safety.

### Start With No Bounds

If the function accepts any integer, generate any integer:

```pseudocode
n = tc.draw(integers())  # full range of the type, no min/max
```

Preemptively adding bounds like `.min(0).max(100)` means you'll never discover that the function overflows on large values, mishandles negatives, or breaks at the type's boundaries. Those are exactly the bugs PBT is designed to find.

### Edge Cases Are the Point

Don't narrow ranges to "avoid edge cases." If a function claims to work on all integers, test it on all integers — including `MIN`, `MAX`, `0`, `-1`, and `1`. If it breaks, that's valuable information.

### Don't Require Non-Empty by Default

Unless the function's contract explicitly requires non-empty input, test with empty collections too. If a function panics on an empty collection, that might be a bug worth knowing about.

### When a Test Fails on Extreme Values

Assume it's a real bug unless you have strong evidence otherwise. If in doubt, ask the user.

- If the function's documentation says it handles all integers but it overflows on `MAX`, that's a bug in the code, not in your test.
- Only add bounds after investigating and confirming the input is outside the function's documented domain.

### When to Add Constraints

Add generator bounds **only** when:

1. **The function's contract explicitly excludes some inputs.** For example, a square root function documents that input must be >= 0.
2. **You need to avoid undefined behavior.** For example, division by zero.
3. **A test failure has been investigated** and confirmed to be outside the function's domain.

### Avoid Rejection Sampling Where Possible

When a constraint involves relationships between multiple generated values, you might use `tc.assume()`:

```pseudocode
a = tc.draw(integers())
b = tc.draw(integers())
tc.assume(a != b)  # this is fine for simple constraints
```

But it's better to construct valid inputs directly when you can:

```pseudocode
# Instead of tc.assume(a <= b), generate in order:
a = tc.draw(integers())
b = tc.draw(integers())
if a > b:
    a, b = b, a
```

This is particularly important when the rejection rate would be high. For example, `integers().map(n -> n * 2)` is much better than `integers().filter(n -> n % 2 == 0)` — the latter throws away ~50% of test cases.

### Getting Large Collections

Hegel's default collection size is small. If you need large collections (e.g., to exercise deep tree paths or multi-level node structures), draw the size separately:

```pseudocode
# can generate large collections, and hegel can shrink n to find the minimal size
n = tc.draw(integers(min=0, max=300))
keys = tc.draw(lists(integers(), min_size=n))  # no max_size — let hegel go bigger

# BAD — hegel's default size distribution rarely produces 100+ elements
keys = tc.draw(lists(integers()))
```

### Use Unique Element Generation for Key Generation

When testing maps/sets that need unique keys, use the unique option on collection generators. This avoids confusion about which value wins for duplicate keys. See the language-specific reference for syntax.

## Handling Randomness in Code Under Test

When the code under test requires an RNG, **do not** create a seeded RNG with a hegel-generated seed. Hegel can only shrink the seed integer, not the actual random decisions the RNG makes — so when a test fails, you get a meaningless minimal seed rather than a meaningful minimal sequence of random choices.

Instead, use hegel's random generator, which gives you an RNG that routes random decisions through hegel's shrinking engine. See the language-specific reference for the exact API.

### Two modes: artificial vs true randomness

- **Default (artificial randomness):** Every random decision goes through hegel, enabling fine-grained shrinking of individual random values. Best for most code.
- **True randomness mode:** Generates a single seed via hegel, then creates a real RNG from it. Hegel can only shrink the seed, not individual random decisions. Use this when the code under test does **rejection sampling** or otherwise depends on the RNG producing statistically random-looking output — artificial randomness can cause rejection loops to hang.

**How to choose:** Start with the default. If tests hang or time out because the code does rejection sampling internally, switch to true randomness mode.

### Refactoring concrete RNG types

If the code under test takes a concrete RNG type rather than a trait/interface, consider whether it should be refactored to accept a generic RNG. This is both better API design and makes the code testable with hegel's random generator. Suggest this refactoring to the user.

## Common Mistakes

1. **Over-constraining generators** — Adding bounds "just in case" means the test will never find bugs at boundary values or with unexpected inputs. The whole value of PBT is exploring the input space the developer didn't think to test by hand. See Generator Discipline above.

2. **Testing trivial properties** — `assert x == x` or `assert len(vec) >= 0` test nothing useful. Every property should be falsifiable by a buggy implementation.

3. **Using the implementation as the oracle** — If your test calls the same function to compute the expected result, it can never fail. Use an independent reference implementation, a simpler algorithm, or a structural property.

4. **High rejection rates** — If `.filter()` or `tc.assume()` rejects most inputs, hegel will give up. Restructure generators to produce valid inputs directly (use `.map()` or dependent draws).

5. **Creating a separate test file for hegel tests** — Property-based tests belong alongside the existing tests for the same code. Add them to existing test files.

6. **Using manually seeded RNGs** — Use hegel's random generator so hegel controls the random decisions and can shrink them individually. See "Handling Randomness" above.

7. **Overflowing in test code** — When computing values from generated data (e.g., `map.insert(k, k * 10)`), your test code itself can overflow before the library has a chance to be buggy. Use wrapping arithmetic or draw a smaller type and widen it to prevent overflow in the test. Distinguish "this constraint protects the library's contract" (keep it) from "this constraint prevents my test from overflowing" (use wrapping arithmetic instead).

8. **Restricting collection size for performance** — If a test is slow with large collections, lower the test case count rather than restricting the input space. A slow test that finds bugs beats a fast test that can't. Many tree/trie bugs only manifest at 50-200+ elements.

