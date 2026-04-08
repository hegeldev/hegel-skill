# Reviewing Hegel Tests for Quality

Evaluate quality of existing property-based tests and suggest improvements.

## Quick Reference

| Issue | Severity | Detection | Fix |
|-------|----------|-----------|-----|
| Tautological | CRITICAL | Assertion compares same expression or is always true | Rewrite with falsifiable property |
| Vacuous | CRITICAL | `tc.assume()` filters all/most inputs | Remove or restructure generators |
| Reimplementation | HIGH | Assertion mirrors function logic | Use algebraic property instead |
| Weak (no assertion) | HIGH | Test body has no assert | Add meaningful assertion |
| Over-constrained generators | HIGH | Unnecessary `.min_value()` / `.max_value()` / `.min_size()` | Remove unjustified bounds |
| Missing properties | MEDIUM | Only weak properties tested | Add stronger properties |
| Poor generator structure | MEDIUM | Heavy use of `.filter()` / `tc.assume()` | Restructure generators |

## Quality Issues

### Tautological Properties (CRITICAL)

Properties that are always true regardless of implementation.

```rust
// BAD - compares function to itself
#[hegel::test]
fn test_sort_tautology(tc: hegel::TestCase) {
    let xs: Vec<i32> = tc.draw(gs::vecs(gs::integers()));
    let sorted = my_sort(&xs);
    assert_eq!(sorted, sorted);  // Always true!
}

// BAD - tests nothing about the function
#[hegel::test]
fn test_useless(tc: hegel::TestCase) {
    let x: i32 = tc.draw(gs::integers());
    let result = compute(x);
    assert_eq!(result, result);  // Always true!
}

// BAD - trivially true for any Vec
#[hegel::test]
fn test_trivial(tc: hegel::TestCase) {
    let xs: Vec<i32> = tc.draw(gs::vecs(gs::integers()));
    assert!(xs.len() >= 0);  // Always true for usize!
}
```

**Detection**: Assertions comparing same expression, or conditions that hold
for any value regardless of what the function does.

### Vacuous Tests (CRITICAL)

Tests where assumptions filter out most or all inputs.

```rust
// BAD - contradictory assumptions
#[hegel::test]
fn test_vacuous(tc: hegel::TestCase) {
    let x: i32 = tc.draw(gs::integers());
    tc.assume(x > 100);
    tc.assume(x < 50);  // Impossible — hegel will give up
    assert!(compute(x) > 0);
}

// BAD - overly restrictive
#[hegel::test]
fn test_too_filtered(tc: hegel::TestCase) {
    let x: i32 = tc.draw(gs::integers());
    tc.assume(x == 42);  // Only tests one value!
    assert_eq!(compute(x), expected);
}
```

**Detection**: Multiple `tc.assume()` calls, or `tc.assume()` with very narrow
conditions. Also look for `.filter()` on generators that rejects most values.

**Fix**: Build constraints into the generator instead:

```rust
// GOOD - constraint in generator
let x: i32 = tc.draw(gs::integers::<i32>().min_value(50).max_value(100));
```

### Reimplementing the Function (HIGH)

```rust
// BAD - just reimplements the logic
#[hegel::test]
fn test_reimplements(tc: hegel::TestCase) {
    let a: i32 = tc.draw(gs::integers());
    let b: i32 = tc.draw(gs::integers());
    assert_eq!(add(a, b), a + b);  // Tests nothing if add() is just a + b
}
```

**Fix**: Use algebraic properties instead:

```rust
// GOOD - tests actual properties
#[hegel::test]
fn test_add_commutative(tc: hegel::TestCase) {
    let a: i32 = tc.draw(gs::integers());
    let b: i32 = tc.draw(gs::integers());
    assert_eq!(add(a, b), add(b, a));  // commutativity
}

#[hegel::test]
fn test_add_identity(tc: hegel::TestCase) {
    let a: i32 = tc.draw(gs::integers());
    assert_eq!(add(a, 0), a);  // identity
}
```

### Over-Constrained Generators (HIGH)

```rust
// BAD - unnecessarily narrow range hides bugs
#[hegel::test]
fn test_narrow(tc: hegel::TestCase) {
    let x: i32 = tc.draw(gs::integers::<i32>()
        .min_value(1).max_value(10));  // Why not full range?
    assert!(compute(x) >= 0);
}

// BAD - unnecessary min_size prevents testing empty collections
#[hegel::test]
fn test_no_empty(tc: hegel::TestCase) {
    let xs: Vec<i32> = tc.draw(gs::vecs(gs::integers::<i32>())
        .min_size(1));  // Does the function actually require non-empty?
    assert!(process(&xs).is_ok());
}
```

**Detection**: `.min_value()`, `.max_value()`, `.min_size()`, `.max_size()`
without a comment or documented reason.

**Fix**: Remove bounds unless the function's contract requires them. If the
function panics on empty input, that might be a bug — ask the user.

### Weak Properties (MEDIUM)

```rust
// WEAK - only tests no crash
#[hegel::test]
fn test_only_no_crash(tc: hegel::TestCase) {
    let s: String = tc.draw(gs::text());
    let _ = process(&s);  // No assertion at all
}

// WEAK - only tests return type
#[hegel::test]
fn test_only_type(tc: hegel::TestCase) {
    let x: i32 = tc.draw(gs::integers());
    let result = compute(x);
    assert!(result > i32::MIN || result <= i32::MIN);  // Always true
}
```

**Detection**: Tests without assertions, or with assertions that are trivially
true.

**Fix**: Look for stronger properties. Check the Property Catalog in SKILL.md.
Can you assert roundtrip? Idempotence? An invariant?

## Review Process

### 1. Locate Hegel Tests

```bash
rg "#\[hegel::test" --type rust
rg "use hegel" --type rust
```

### 2. Check Each Test

For each test, evaluate in order of severity:

1. **Is it tautological?** Does the assertion depend on the function's output
   at all? Could a completely wrong implementation pass?
2. **Is it vacuous?** Do assumptions filter out most inputs?
3. **Does it reimplement the function?** Is the assertion just the same logic?
4. **Are generators over-constrained?** Are bounds justified by the function's
   contract?
5. **Is the property strong enough?** Can you find a stronger property from
   the catalog?

### 3. Check Generator Quality

- Are generators as broad as the function's domain allows?
- Is `.filter()` / `tc.assume()` used where generator restructuring would be better?
- For code using randomness, is `gs::randoms()` used instead of manual seeding?

### 4. Check for Missing Properties

Compare tested properties against the property catalog. For each function
under test, ask:

- Does it have a paired operation? → Roundtrip
- Is applying it twice the same as once? → Idempotence
- Does it preserve some quantity? → Invariant
- Is there a reference implementation? → Oracle
- Can the output be easily verified? → Easy-to-verify

### 5. Check for Flakiness Risk

- Non-determinism in code under test (without hegel-controlled RNG)
- Floating point comparisons without tolerance
- Global state dependencies
- Time-dependent assertions

## Quality Checklist

For each test, verify:
- [ ] Not tautological (a buggy implementation could actually fail this test)
- [ ] Not vacuous (inputs are not over-filtered)
- [ ] Doesn't reimplement the function logic
- [ ] Generators are as broad as the function's domain
- [ ] Generator bounds are justified by the function's contract
- [ ] Property is as strong as possible (not just "no crash")
- [ ] No unnecessary `.filter()` or `tc.assume()` — generators are structured
- [ ] Tests live in the appropriate file (not a separate `test_hegel.rs`)

## Mutation Testing Verification

To verify tests actually catch bugs, consider what mutations would pass:

```
To verify test_sort catches bugs:

1. Return input unchanged: `return xs`
   → Should fail: test_ordering

2. Drop last element: `return sorted[..sorted.len()-1]`
   → Should fail: test_length_preserved

3. Reverse order: `return sorted(xs, reverse=true)`
   → Should fail: test_ordering

4. Return empty: `return vec![]`
   → Should fail: test_length_preserved, test_elements_preserved
```

If a plausible mutation wouldn't be caught, the test suite needs a stronger property.
