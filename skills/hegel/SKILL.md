---
name: hegel
description: >
  Write property-based tests using Hegel across Rust, Go, C++, TypeScript,
  Java, and OCaml projects. Use this skill whenever the user asks to write
  tests, add test coverage, or improve testing for functions, modules, or
  libraries — especially when the code has properties like round-trips,
  invariants, or contracts that hold across many inputs. Also triggers on:
  "property-based tests", "PBT", "hegel", "fuzz", "generative tests",
  "randomized testing", "test with random inputs", "shrinking", or when
  existing tests use proptest, quickcheck, rapid, gopter, rapidcheck,
  fast-check, jqwik, junit-quickcheck, qcheck, or crowbar.
---

# Hegel: Property-Based Testing

Hegel is a family of property-based testing libraries supporting multiple languages, powered by a shared native engine based on Hypothesis. Everything runs in-process, and tests integrate with standard language test runners. Hegel generates random inputs for your code and automatically shrinks failing cases to minimal counterexamples.

Even when PBTs add modest line coverage over unit tests, their value is in exercising combinations and boundary conditions that humans don't think to write by hand.

**Code examples in this file use Python-like pseudocode to illustrate concepts.** For exact API and syntax, load the language-specific reference (see step 1 of the workflow).

## Workflow

Follow these steps when writing property-based tests.

**Before touching a third-party project, check whether its maintainers have opted out of AI contributions.** Look for a governance file gating AI work (`HUMAN_VS_MACHINE.md`, an `.ai-ack`/acknowledgement requirement), or an anti-AI clause in `CONTRIBUTING.md`/`README`/`LICENSE`. If you find one, **surface it to the user and stop** — do not silently create the acknowledgement file, bypass the gate, or upstream anything. Testing a local copy for your own understanding may still be fine, but the decision to proceed and especially to contribute back is the user's to make, not yours to assume.

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
- **Shrinking is automatic.** Hegel's engine handles shrinking. You don't implement shrink logic or define shrinking strategies.
- **Standard assertions.** Use the language's normal assertion mechanism. No special `prop_assert!` or return-a-bool pattern needed.
- **Broaden your generators.** Many existing PBTs use narrow input ranges because shrinking was slow or unreliable. Hegel's shrinking is more robust — try broader generators than the originals.

**Unit tests and example-based tests** can often be evolved into PBTs. Tests with hardcoded seeds, parameterized examples, or multiple similar test cases are prime candidates. Load `references/evolving-tests.md` for detailed guidance on recognizing what property a unit test is hiding. If you can't immediately see the right property, start by parameterizing the test — replace concrete values with generated ones and keep a simple oracle. You can refine the property later.

**Tests that use `rand` with fixed seeds** are especially good candidates — the randomness should come from hegel instead so failures produce shrinkable counterexamples. Default to *supplementing* rather than deleting: keep the original seeded test (it pins known-good behavior cheaply) and add the hegel version beside it, unless the original is fully subsumed and the maintainers would clearly prefer one test.

When you evolve an existing test, **modify the existing test file** rather than creating a new one. Property-based tests are tests like any other and belong with the code they're testing. Do not create a separate file for hegel tests.

### 5. Write the Tests

For each property:

1. **Add tests to the appropriate existing test file.** Only create a new file if no relevant test file exists.
2. Choose the **simplest possible generators** — see Generator Discipline below.
3. Draw values, run the code under test, and assert the property.

### 6. Run and Reflect

Run the tests. Two checks before trusting a green run:

- **Verify the tests are live — and that guarded branches actually fire.** Passing hegel tests print nothing extra, so temporarily break one property (invert an assertion), confirm hegel reports a shrunk counterexample, and revert. A suite that can't fail is worthless. Note that mutation detection is itself stochastic: a mutation that *should* be caught can survive a single default-count run by bad luck (the triggering input just wasn't drawn), which looks identical to a dead test. If a mutation unexpectedly survives, re-run it (and/or at a higher case count) before concluding the property is vacuous — and don't wipe the `.hegel` failure database between the broken and reverted runs, since its replay is what preserves a lucky-draw counterexample. Separately, for a property whose interesting work is behind a guard (`if let Some(c) = compose(a,b) { assert … }`, `if parses { check roundtrip }`), the plain liveness check is not enough — the property can pass *vacuously* because the guard never held. Confirm the guarded branch is reached (invert the assertion *inside* the guard and check it still fails, or count how often the branch runs), and if it fires rarely, add a generator that constructs inputs satisfying the guard directly.
- **Do one exploratory high-count run.** The default case count is tuned for CI, not for discovery — real bugs are regularly invisible at the default but found at 10x (boundary coincidences need the extra trials). Run the suite once with `test_cases` raised ~10x before concluding the code is clean, then drop back to the default for the committed tests. Don't trust wall-clock time to confirm the high-count run actually ran more cases — a fast property finishes near-instantly at both counts, so a run that silently did nothing extra looks identical. Confirm it with a case counter (increment a counter each iteration, print or assert the total) rather than by how long it took.

When a test fails, ask:

- **Is this a real bug?** If the code violates its own contract, flag the bug to the user and ask what to do, or fix the code if instructed to do so. Weight your confidence by *how* the property fails. **Oracle-independent** failures — a panic, a crash/abort, a hang, a returned value that violates a self-evident invariant (a duplicate in a set, a CDF outside [0,1], a round-trip that doesn't restore the input you still hold) — are almost always real, because no external judgment is involved. **Oracle-dependent** failures — a differential disagreement with a hand-written reimplementation or a sibling library — are only as trustworthy as the oracle: in a mature, heavily-fuzzed crate a semantic disagreement is more likely a bug in *your* oracle, or a genuinely under-specified corner (empty patterns, degenerate/extreme inputs) the crate never promised behavior for, than a real defect. Before pinning an oracle-dependent disagreement in a hardened crate, confirm it isn't oracle-independent-reproducible (reduce it to a panic/invariant-violation if you can) and check whether the docs actually promise the behavior you expected — if the contract is silent, report it upstream as a question, not as a confirmed bug. Special case: if the failure *aborts or hangs the process* (stack overflow, OOM-kill, or an infinite loop — there is no per-test timeout, so a hang looks exactly like slow shrinking) rather than panicking, a permanently failing test would take the whole suite down with it — keep a minimal reproducer marked ignored/skipped with a comment explaining why, and document the bug prominently instead. To tell a genuine hang from slow generation/shrinking, look at the stuck process: 0% CPU with all threads parked (e.g. in `futex_wait`) is a deadlock, whereas steady CPU is just slow work. A deadlock in the code under test is especially insidious under a shared/parallel test pool — the wedged tasks occupy workers and stall *unrelated* tests too; build the code under test on a single-threaded pool in the affected tests so one bug can't hang the rest of the suite. Once a bug is pinned by one canonical failing test, it is fine to exclude that known-buggy input from *sibling* properties so they stay green and keep testing everything else — add a comment on each exclusion referencing the canonical test.
- **Is the property unsound?** If you asserted something the code never promised, fix the test.
- **Is the generator too broad?** Only if the failing input is genuinely outside the function's domain, add constraints. Investigate before constraining.

### If Hegel Itself Won't Work

If you cannot get hegel installed or compiling in the project, **stop and report the blocker with the exact error** — do not fall back to writing example-based tests and presenting them as property-based tests. That silently substitutes a different deliverable, and the blocker is usually one of the documented setup issues (crate naming, MSRV, old editions — check the language reference's Setup and Gotchas sections first).

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

**Parse robustness** — Parsers (`from_str`, `parse`, `decode`) should handle all input without panicking. The property is simple: it should never crash, even on garbage input. Pair it with the converse **validation/rejection** check: take a *valid* input, apply a mutation that makes it invalid per the grammar (inject a forbidden character, corrupt a length field, add leading zeros where banned), and assert the parser returns an error — no-panic and valid-roundtrip properties both miss parsers that silently accept bad input.

```pseudocode
s = tc.draw(text())
discard(MyType.parse(s))  # should return an error, never panic
```

(Use your language's idiomatic way to discard the result — in lint-strict Rust projects `drop(...)` avoids `let_underscore_drop` warnings, though for `Copy` results it trips `dropping_copy_types` and `let _ =` is right; pick per the type.)

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
| **Feature flag testing** | non-default features are often less tested | enable SIMD, nightly, or experimental features and run tests; re-run the *finished* suite once per feature/backend combination — a bug on a non-default code path (e.g. a SIMD wrapper missing an empty-input guard the scalar path has) is invisible in the default build at any case count |

### Generate Configurations, Not Just Payloads

When a library exposes a *builder or specification* for its core object — an encoding spec, parser options, layout style, compression level, schema — generate the configuration too, not just payloads for a handful of canned configurations. Running one round-trip property against every predefined configuration mostly re-tests the same code path; generating arbitrary *valid* configurations explores the interactions (padding × wrapping × translation, option × option) where bugs actually live. Build a composite generator that constructs valid configurations directly, encoding the documented validity rules as construction logic rather than filters:

```pseudocode
composite arbitrary_config(tc):
    width = tc.draw(sampled_from(valid_widths))
    symbols = tc.draw(lists(symbol_chars, size=width, unique=true))
    config = Config(symbols)
    if tc.draw(booleans()):
        config.padding = tc.draw(chars_not_in(symbols))
    ...
    return config
```

The library's own fuzz targets (a `fuzz/` directory) are strong evidence for which object the maintainers consider worth generating — read them.

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

### Oracle Sourcing

A property is only as good as its oracle — the independent source of truth it checks against. Beyond a hand-written reference implementation, these oracle patterns have repeatedly found real bugs:

- **The crate's own dependency tree.** Exact or reference implementations often ship as dependencies (a geometry crate on exact predicates, a datetime crate on a tz database). They can't be fooled by the same precision/logic error as the code under test.
- **A dispatcher's generic fallback vs its specialized fast path.** When a library has both a fast specialized routine and a general algorithm for the same query (a cuboid-cuboid distance shortcut and a GJK fallback; an ASCII fast path and a Unicode path), run both and assert they agree — one is usually the oracle for the other. When you *cannot select* which path runs (the library dispatches internally on the input), force the slower/general path with a **semantics-neutral wrapper** on the input — e.g. a regex engine that delegates simple patterns to a fast crate can be forced onto its own backtracking VM by prefixing a no-op `(?=[\s\S]?)`; the differential then exercises the path you actually want to test instead of tautologically re-running the delegate.
- **A sibling API that must agree.** `distance(a,b) == 0` iff `intersects(a,b)`; `find(x).is_some()` iff `contains(x)`; a streaming API vs a one-shot API. But **beware the converse**: a sibling that shares the implementation (or is *defined* in terms of the code under test) can share the bug — a broken convex hull was accepted by the crate's own `contains`, which trusted the same broken geometry.
- **Exhaustive enumeration over a small universe.** For "for all subsets / for all quorums / for all orderings" safety properties, generate the structure over a tiny fixed universe (say 5–7 elements) and check against brute-force enumeration of every subset (a bitmask loop). This turns an un-checkable universal claim into a decidable one and is how a consensus library's joint-quorum coherence was validated.
- **The spec, transcribed.** For a wire format with a written spec, transcribe the spec's algorithm (a LEB128 encoder, an RFC length-header rule) as the oracle rather than trusting the crate's own encoder as its own reference.

**Audit a floating-point oracle before trusting a tiny disagreement.** If your oracle is itself an external library (a JSON parser, another codec) and it disagrees by ~1 ULP, suspect the *oracle* first — the reference may round differently than the code under test (a JSON crate's default float parser vs its opt-in round-trip mode). Confirm the oracle is exact for the domain before reporting a 1-ULP mismatch as a bug.

### Choosing Properties

Properties must be **evidence-based**. Find evidence in:

- **Names and type signatures**: A function `merge(a: List, b: List) -> List` implies the output length might equal the sum of input lengths.
- **Docstrings and comments**: "Returns a sorted list" directly gives you an invariant.
- **Assertions and debug checks in the source**: These are properties the author already identified — they may suggest other invariants.
- **Oracles**: see the Oracle Sourcing section above for where independent sources of truth come from.
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

**Serializer round-trips: generate at the encode-set boundary.** When a round-trip property involves escaping or percent-encoding, read the writer's encode/escape set and generate characters *just outside* it — bugs live in the characters the writer forgot to escape, and uniform generation almost never lands on the one missing character (a credential-corrupting missing `:` survived a 10x run until the encode set was read and targeted).

**Probe recursion depth directly.** For code that consumes recursive input (parsers, interpreters, decoders), a stack overflow or missing depth limit lives at nesting depths of hundreds to thousands — depths uniform random generation almost never reaches. Draw a depth explicitly and build input nested to it (`"(" * n + ")" * n`, deeply nested arrays/objects), sweeping n upward, rather than hoping a recursive generator wanders deep. This class (unbounded recursion aborting the process) recurred across every interpreter and several parsers tested. It is highest-value against a *checked/validated entry point that makes an explicit safety promise* (a `validate`/`access`/`from_untrusted_bytes` that documents protection against malicious input): a stack overflow there is a security-contract violation, not a robustness nit.

**Callback/streaming APIs: cap emitted output.** For an API that streams results through a callback or into an unbounded buffer (a flattener, a shaper, an iterator that materializes), a bug that emits unboundedly OOM-kills the test — an abort, not a shrinkable failure. Cap the emission count in the test harness and assert the cap isn't hit, converting the OOM into a normal falsifiable property.

**Construct degenerate cases exactly, don't boost toward them.** When the interesting input is exactly degenerate (a zero-length segment, coincident points, an all-equal collection), generate it by construction — boosting a broad generator "toward" degeneracy often lands *near* it, and rounding/normalization then de-degenerates it before the code under test sees the exact case.

**Boundary conjunctions need help.** Some bugs require several drawn values to hit boundaries *simultaneously* (e.g. scale is `MIN` *and* the mantissa has trailing zeros). Uniform generation rarely produces such coincidences in a default run. When a property combines multiple drawn values, either boost boundary values explicitly (a `one_of` between the full range and a sample of `MIN`/`MAX`/`0`), or read the code for suspicious arithmetic on the drawn quantities and target the conjunction it implies.

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

7. **Overflowing in test code** — When computing values from generated data (e.g., `map.insert(k, k * 10)`), your test code itself can overflow before the library has a chance to be buggy. Use wrapping arithmetic or draw a smaller type and widen it to prevent overflow in the test. Distinguish "this constraint protects the library's contract" (keep it) from "this constraint prevents my test from overflowing" (use wrapping arithmetic instead). The same distinction applies to *memory*: an oracle or model that materializes what the library represents lazily (e.g. aligning two numbers whose exponents differ by 2^60) can OOM the test on inputs the library handles fine. Bound such inputs, and say in a comment that the bound protects the test's resources, not the library's contract — ideally keep a separate unbounded no-panic test for the extremes. It applies to *runtime* too: when one implementation under test is super-polynomial by design (an exhaustive/IDA\*-style search that is exponential in input size), a single large drawn input hangs the whole property — lowering the case count doesn't help because it's one case that's slow, not many. Bound that algorithm's input size specifically (comment that the bound protects the test's runtime, not any contract), and let the polynomial siblings run on larger inputs.

8. **Restricting collection size for performance** — If a test is slow with large collections, lower the test case count rather than restricting the input space. A slow test that finds bugs beats a fast test that can't. Many tree/trie bugs only manifest at 50-200+ elements.

