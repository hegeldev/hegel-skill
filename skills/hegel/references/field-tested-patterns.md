# Field-Tested Property Patterns

These patterns are drawn from extensive property-based testing of popular Rust
crate libraries with hegel. They are ordered by effectiveness — patterns that
found more bugs are listed first.

## Pattern 1: Model Tests for Data Structures

Compare every operation on the data structure under test against a known-good
std reference. Assert agreement after **every** operation, not just at the end.

```rust
#[hegel::test(test_cases = 1000)]
fn test_model(tc: hegel::TestCase) {
    let mut subject = MyMap::new();
    let mut model = std::collections::HashMap::new();

    let num_ops = tc.draw(generators::integers::<usize>().max_value(100));
    for _ in 0..num_ops {
        let op = tc.draw(generators::integers::<u8>().max_value(4));
        match op {
            0 => {
                let k = tc.draw(generators::integers::<i32>());
                let v = tc.draw(generators::integers::<i32>());
                assert_eq!(subject.insert(k, v), model.insert(k, v), "insert mismatch");
            }
            1 => {
                let k = tc.draw(generators::integers::<i32>());
                assert_eq!(subject.remove(&k), model.remove(&k), "remove mismatch");
            }
            2 => {
                let k = tc.draw(generators::integers::<i32>());
                assert_eq!(subject.get(&k), model.get(&k), "get mismatch");
            }
            3 => {
                let k = tc.draw(generators::integers::<i32>());
                assert_eq!(subject.contains_key(&k), model.contains_key(&k));
            }
            _ => {
                assert_eq!(subject.len(), model.len(), "len mismatch");
            }
        }
        assert_eq!(subject.len(), model.len(), "len mismatch after op");
    }
}
```

**Key points:**
- Assert **return values** of mutating operations (insert, remove), not just
  final state. A common bug pattern is `insert` returning the wrong boolean
  (e.g. claiming a value was already present when it wasn't).
- Include `len()` checks after every operation to catch subtle state corruption
  like stale index entries that aren't cleaned up on update.
- Use unconstrained key generators — some bugs only manifest with many unique
  keys (e.g. 50-200+) because they require deep tree structures with multiple
  node levels.

**Oracle selection:**

| Data structure type | Oracle |
|---|---|
| Sequential containers (fixed-capacity vecs, small vecs) | `Vec` |
| Deque-like containers (ring buffers, persistent vectors) | `VecDeque` |
| Hash maps (alternative hash maps, concurrent maps) | `HashMap` |
| Ordered maps (tree maps, persistent ordered maps) | `BTreeMap` |
| Ordered sets / bitmaps (compressed bitmaps, tree sets) | `BTreeSet` |
| Unordered sets (indexed sets, bit sets) | `HashSet` |

## Pattern 2: Idempotence Tests for String Processing

Any normalization, case conversion, or formatting function should be idempotent.
The critical ingredient is `generators::text()` — ASCII-only inputs miss the bugs.

```rust
#[hegel::test(test_cases = 1000)]
fn test_normalize_idempotent(tc: hegel::TestCase) {
    let s: String = tc.draw(generators::text());
    let once = normalize(&s);
    let twice = normalize(&once);
    assert_eq!(once, twice,
        "not idempotent for {:?}: {:?} -> {:?}", s, once, twice);
}
```

**Why Unicode matters:** Some Unicode characters change length when case-mapped.
For example, the German sharp-s (`ß`) uppercases to `SS` (two characters). A
case conversion function that splits words on case transitions will see different
word boundaries on the first and second pass, breaking idempotence. This is
completely invisible with ASCII-only generators.

**Apply to:** case conversion, URL normalization, path canonicalization, HTML
escaping, string slugification, Unicode normalization, any function that
transforms text into a "canonical" form.

## Pattern 3: Parse Robustness

Every `from_str`, `parse`, or `decode` function should handle all input without
panicking — even invalid input. The property is simple:

```rust
#[hegel::test(test_cases = 1000)]
fn test_parse_robustness(tc: hegel::TestCase) {
    let s: String = tc.draw(generators::text());
    let _ = MyType::from_str(&s);  // Should never panic
}
```

**Why this finds bugs:** Parsers often delegate to internal constructors that
panic on invalid values. For example, a fraction parser might successfully parse
a numerator and denominator from the string, then call a constructor like
`Ratio::new(0, 0)` which panics with "denominator == 0" instead of returning
an error. The parser validated the syntax but not the semantics.

**Apply to:** any `FromStr` impl, any `parse()` method, any `decode()` function,
XML/JSON/YAML/TOML parsers, URL parsers, date/time parsers.

## Pattern 4: Roundtrip Tests

Test `parse(format(x)) == x` for any serialize/deserialize pair.

```rust
#[hegel::test(test_cases = 1000)]
fn test_display_parse_roundtrip(tc: hegel::TestCase) {
    let v = tc.draw(generators::integers::<i64>());
    let s = format!("{}", v);
    let parsed: i64 = s.parse().unwrap();
    assert_eq!(v, parsed);
}
```

**Where roundtrips break:**
- **Zero as a special case:** Formatters that produce scientific notation may
  emit `"e0"` instead of `"0e0"` for zero — missing the coefficient entirely.
  The parser then rightfully rejects the output.
- **Large integers through f64:** Some parsers route all numeric types through
  f64 internally, silently losing precision for integers > 2^53. The value
  `9007199254740993` gets roundtripped as `9007199254740992`.
- **Unusual path components:** URL and path operations may break roundtrips on
  edge cases like double slashes, empty segments, or relative path resolution.

## Pattern 5: Boundary Value Tests for Numeric Code

Integer boundary values (`MIN`, `MAX`, `0`) are where overflow bugs hide. Don't
add bounds to avoid them — they ARE the test.

```rust
#[hegel::test(test_cases = 1000)]
fn test_numeric_operation(tc: hegel::TestCase) {
    let a = tc.draw(generators::integers::<i64>());  // includes i64::MIN
    let b = tc.draw(generators::integers::<i64>());
    tc.assume(b != 0);
    // Operations that internally negate, multiply, or compute GCD/LCM
    // often overflow on boundary values
    let _result = my_numeric_op(a, b);
}
```

**Common overflow patterns:**
- **Negating `MIN`:** `-i32::MIN` overflows because `|i32::MIN| > i32::MAX`.
  Any code path that negates an integer (conjugate, absolute value, GCD) is
  vulnerable.
- **Intermediate products:** Computing `a * b + c` where the multiplication
  overflows even though the final result would fit.
- **GCD/LCM computations:** These often internally negate values or multiply
  denominators, triggering overflow on boundary inputs.
- **Display/formatting:** Implementations that check `if value < 0` then negate
  to format the absolute value will panic on `MIN`.

## Pattern 6: API Consistency Tests

When a library provides multiple ways to compute the same thing, they should
agree:

```rust
#[hegel::test(test_cases = 1000)]
fn test_batch_vs_individual(tc: hegel::TestCase) {
    let s: String = tc.draw(generators::text());
    let batch_result = compute_for_string(&s);
    let individual_sum: usize = s.chars().map(|c| compute_for_char(c)).sum();
    assert_eq!(batch_result, individual_sum);
}
```

**Apply to:** any library where a "batch" API and "single-item" API should agree
(e.g. string width vs sum of character widths), parallel vs sequential
implementations, different algorithm modes (e.g. NFA vs DFA in a regex engine),
or different encoding paths that should produce identical output.

## Pattern 7: Large Input Sizes

Small inputs (< 20 elements) often fit in a single tree/trie node. Traversal
bugs between nodes are never exercised. Draw the size separately to force large
inputs:

```rust
#[hegel::test(test_cases = 1000)]
fn test_with_large_input(tc: hegel::TestCase) {
    let n = tc.draw(generators::integers::<usize>().max_value(300));
    let keys: Vec<i32> = tc.draw(generators::vecs(generators::integers())
        .min_size(n).max_size(n));
    // ... test with large data structure
}
```

Tree/trie data structures are especially vulnerable — bugs in B-tree node
splitting, rebalancing, or cross-node traversal only manifest when the tree
has enough keys to require multiple levels of internal nodes.

## Pattern 8: Feature Flag Testing

Non-default features are often less tested. Check `Cargo.toml` for features and
enable them:

```bash
grep -A20 "\[features\]" /path/to/library/Cargo.toml
cargo +nightly test --test test_foo  # for unstable features
```

SIMD, nightly-only, and experimental features are prime targets. Sometimes the
README or documentation will even say "this feature has not been tested" — take
that as a direct invitation.

## Bug Patterns by Category

| Category | What to look for |
|---|---|
| **Integer overflow** | Boundary values (MIN, MAX, 0) in arithmetic, GCD, negation, display |
| **Idempotence failure** | Case conversion / normalization with Unicode (ß → SS), word splitting on case transitions |
| **Precision loss** | Numbers routed through f64 lose precision for integers > 2^53 |
| **Roundtrip failure** | Format/parse on edge cases: zero, empty strings, unusual path components |
| **Parse panic** | `from_str` delegates to a constructor that panics instead of returning Err |
| **Stale state** | Update operations that modify one index but don't clean up the old entry in another |
| **Unicode line breaks** | `\u{85}` (NEL), `\u{2028}` (LS), `\u{2029}` (PS) treated inconsistently as line breaks |
| **SIMD divergence** | SIMD code path produces different results than the scalar fallback |
| **Deep structure bugs** | Traversal that only fails when data structure has multiple internal levels (50-200+ elements) |
