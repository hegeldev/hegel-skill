# Field-Tested Property Patterns

These patterns are drawn from testing 120+ of the top 1000 most-downloaded Rust
crates with hegel, finding 13 real bugs. They are ordered by effectiveness —
patterns that found more bugs are listed first.

## Pattern 1: Model Tests for Data Structures

**Bug yield:** Found bugs in `multi-map`, `roaring` (SIMD), and `im` (B-tree).

Compare every operation on the library under test against a known-good std
reference. Assert agreement after **every** operation, not just at the end.

```rust
#[hegel::test(test_cases = 1000)]
fn test_model(tc: hegel::TestCase) {
    let mut lib = LibraryMap::new();
    let mut model = std::collections::HashMap::new();

    let num_ops = tc.draw(generators::integers::<usize>().max_value(100));
    for _ in 0..num_ops {
        let op = tc.draw(generators::integers::<u8>().max_value(4));
        match op {
            0 => {
                let k = tc.draw(generators::integers::<i32>());
                let v = tc.draw(generators::integers::<i32>());
                assert_eq!(lib.insert(k, v), model.insert(k, v), "insert mismatch");
            }
            1 => {
                let k = tc.draw(generators::integers::<i32>());
                assert_eq!(lib.remove(&k), model.remove(&k), "remove mismatch");
            }
            2 => {
                let k = tc.draw(generators::integers::<i32>());
                assert_eq!(lib.get(&k), model.get(&k), "get mismatch");
            }
            3 => {
                let k = tc.draw(generators::integers::<i32>());
                assert_eq!(lib.contains_key(&k), model.contains_key(&k));
            }
            _ => {
                assert_eq!(lib.len(), model.len(), "len mismatch");
            }
        }
        assert_eq!(lib.len(), model.len(), "len mismatch after op");
    }
}
```

**Key points:**
- Assert **return values** of mutating operations (insert, remove), not just
  final state. The `roaring` SIMD bug was `insert` returning `false` instead of
  `true`.
- Include `len()` checks after every operation to catch subtle state corruption.
- Use unconstrained key generators — the `im` bug needed 98+ unique keys.

**Oracle selection:**

| Data structure type | Oracle |
|---|---|
| Sequential containers (SmallVec, TinyVec, heapless::Vec) | `Vec` |
| Deque-like (CircularBuffer, im::Vector) | `VecDeque` |
| Hash maps (AHashMap, IndexMap, DashMap, FxHashMap) | `HashMap` |
| Ordered maps (im::OrdMap, rpds::RedBlackTreeMap) | `BTreeMap` |
| Ordered sets / bitmaps (RoaringBitmap, im::OrdSet) | `BTreeSet` |
| Unordered sets (IndexSet, BitSet) | `HashSet` |

## Pattern 2: Idempotence Tests for String Processing

**Bug yield:** Found bugs in `heck` (5 failures) and `convert_case` (3 failures).

Any normalization, case conversion, or formatting function should be idempotent.
The critical ingredient is `generators::text()` — ASCII-only inputs miss the bugs.

```rust
#[hegel::test(test_cases = 1000)]
fn test_case_conversion_idempotent(tc: hegel::TestCase) {
    let s: String = tc.draw(generators::text());
    let once = s.to_upper_camel_case();
    let twice = once.to_upper_camel_case();
    assert_eq!(once, twice,
        "not idempotent for {:?}: {:?} -> {:?}", s, once, twice);
}
```

**Why Unicode matters:** The German sharp-s (`ß`) uppercases to `SS` (two
characters). On the first pass, `"ß".to_title_case()` → `"SS"`. On the second
pass, `"SS".to_title_case()` → `"Ss"`. This breaks idempotence but is invisible
with ASCII-only generators.

**Apply to:** case conversion, URL normalization, path canonicalization, HTML
escaping, string slugification, Unicode normalization.

## Pattern 3: Parse Robustness

**Bug yield:** Found bug in `fraction` (panics on "0/0" instead of returning Err).

Every `from_str`, `parse`, or `decode` function should handle all input without
panicking — even invalid input. The property is simple:

```rust
#[hegel::test(test_cases = 1000)]
fn test_parse_robustness(tc: hegel::TestCase) {
    let s: String = tc.draw(generators::text());
    let _ = MyType::from_str(&s);  // Should never panic
}
```

**Why this finds bugs:** Parsers often delegate to constructors that panic on
invalid values. `Fraction::from_str("0/0")` successfully parses numerator and
denominator, then calls `Ratio::new(0, 0)` which panics with "denominator == 0".
A well-behaved parser should return `Err`.

**Apply to:** any `FromStr` impl, any `parse()` method, any `decode()` function,
XML/JSON/YAML/TOML parsers, URL parsers, date/time parsers.

## Pattern 4: Roundtrip Tests

**Bug yield:** Found bugs in `url` (make_relative), `rust_decimal` (scientific
notation), and `json5` (integer precision loss).

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
- **Zero:** `format!("{:e}", Decimal::ZERO)` produces `"e0"` (missing
  coefficient) instead of `"0e0"`.
- **Large integers through f64:** `json5` routes all numbers through f64
  internally, losing precision for integers > 2^53.
- **Double slashes in URLs:** `url.make_relative(&target)` returns `""` for
  paths with `//`, but `url.join("")` preserves the base's double slash instead
  of navigating to the target.

## Pattern 5: Boundary Value Tests for Numeric Code

**Bug yield:** Found bugs in `num-rational` (4 overflow panics) and `num-complex`
(3 overflow panics).

Integer boundary values (`MIN`, `MAX`, `0`) are where overflow bugs hide. Don't
add bounds to avoid them — they ARE the test.

```rust
#[hegel::test(test_cases = 1000)]
fn test_ratio_construction(tc: hegel::TestCase) {
    let num = tc.draw(generators::integers::<i64>());  // includes i64::MIN
    let den = tc.draw(generators::integers::<i64>());
    tc.assume(den != 0);
    // This panics for num = i64::MIN because GCD tries to negate it
    let _ratio = Ratio::new(num, den);
}
```

**Common overflow patterns:**
- Negating `MIN` (`-i32::MIN` overflows because `|MIN| > MAX`)
- Computing `a * b + c` where intermediate products overflow
- GCD/LCM computations that internally negate values
- `Display` implementations that check `if value < 0` then negate

## Pattern 6: API Consistency Tests

**Bug yield:** Found bug in `unicode-width` (string width vs char width disagree).

When a library provides multiple ways to compute the same thing, they should agree:

```rust
#[hegel::test(test_cases = 1000)]
fn test_width_consistency(tc: hegel::TestCase) {
    let s: String = tc.draw(generators::text());
    let str_width = s.width();
    let char_sum: usize = s.chars().map(|c| c.width().unwrap_or(0)).sum();
    assert_eq!(str_width, char_sum);
}
```

**Apply to:** any library where a "batch" API and "single-item" API should agree,
parallel vs sequential implementations, different algorithm modes (NFA vs DFA).

## Pattern 7: Large Input Sizes

**Bug yield:** Found bug in `im` (B-tree traversal bug only manifests at 98+ keys).

Small inputs (< 20 elements) often fit in a single tree/trie node. Traversal bugs
between nodes are never exercised. Draw the size separately to force large inputs:

```rust
#[hegel::test(test_cases = 1000)]
fn test_deep_tree(tc: hegel::TestCase) {
    let n = tc.draw(generators::integers::<usize>().max_value(300));
    let keys: Vec<i32> = tc.draw(generators::vecs(generators::integers())
        .min_size(n).max_size(n));
    // ... test with large data structure
}
```

## Pattern 8: Feature Flag Testing

**Bug yield:** Found bug in `roaring` (SIMD feature breaks insert return value).

Non-default features are often less tested. Check `Cargo.toml` for features and
enable them:

```bash
grep -A20 "\[features\]" /path/to/library/Cargo.toml
cargo +nightly test --test test_foo  # for unstable features
```

The `roaring` README explicitly said "The simd feature has not been tested" — and
indeed it was broken.

## Bug Patterns by Category

| Category | Libraries affected | What to look for |
|---|---|---|
| **Integer overflow** | num-rational, num-complex, fraction | Boundary values (MIN, MAX, 0) in arithmetic, GCD, negation |
| **Idempotence failure** | heck, convert_case | Case conversion with Unicode (ß → SS), word splitting on case transitions |
| **Precision loss** | json5 | Numbers routed through f64 lose precision for integers > 2^53 |
| **Roundtrip failure** | url, rust_decimal | Format/parse on edge cases (zero, double slashes, scientific notation) |
| **Parse panic** | fraction | `from_str` delegates to constructor that panics instead of returning Err |
| **Stale state** | multi-map | Update operations that don't clean up old index entries |
| **Unicode line breaks** | textwrap | `\u{85}` (NEL) treated as line break by some functions but not others |
| **SIMD divergence** | roaring | SIMD code path produces different results than scalar path |
| **Deep tree bugs** | im | B-tree traversal that only fails when tree has multiple levels (98+ keys) |
