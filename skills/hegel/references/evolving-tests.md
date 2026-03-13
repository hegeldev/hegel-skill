# Evolving Example-Based Tests into Property-Based Tests

Existing unit tests are often the best starting point for property-based tests.
They encode domain knowledge about what the code should do, and they frequently
contain implicit properties that can be generalized.

## Good Candidates for Evolution

Look for tests that:

- **Have multiple similar test cases.** Three tests for `parse("1")`,
  `parse("42")`, `parse("-7")` suggest a round-trip property: parse then
  format recovers the original.
- **Use simple input types.** Tests with integer, string, or collection inputs
  are easy to parameterize with generators.
- **Test round-trip behavior.** `assert_eq!(decode(encode(x)), x)` is already
  a property — just replace `x` with a generator.
- **Contain existing randomness.** Tests that use `rand` or hand-rolled random
  inputs are begging to be property-based tests.
- **Test invariants across examples.** If every test case checks the same
  condition (e.g., output is sorted, length is preserved), that's a property.

## Poor Candidates

- **Exact-output tests.** `assert_eq!(render(doc), "<html>...")` depends on a
  specific output format that's hard to express as a property.
- **Complex setup with fixtures.** Tests that require database state, network
  mocks, or elaborate setup are harder to parameterize (though not impossible).
- **UI / snapshot tests.** Visual regression tests don't have obvious
  properties.
- **Tests of specific error messages.** Checking exact error strings is a
  unit test concern; PBTs work better for testing that errors are *raised*
  rather than what they *say*.

## The Evolution Process

### Step 1: Identify the Property

Read the existing tests and ask: **what is true across all these examples?**

```rust
// Before: three unit tests
#[test]
fn test_abs_positive() { assert_eq!(my_abs(5), 5); }
#[test]
fn test_abs_negative() { assert_eq!(my_abs(-3), 3); }
#[test]
fn test_abs_zero() { assert_eq!(my_abs(0), 0); }
```

The property: `my_abs(x) >= 0` for all `x`, and `my_abs(x) == my_abs(-x)`.

### Step 2: Parameterize

Replace concrete values with generated ones:

```rust
#[hegel::test]
fn test_abs_non_negative(tc: hegel::TestCase) {
    let x = tc.draw(generators::integers::<i64>());
    assert!(my_abs(x) >= 0);
}

#[hegel::test]
fn test_abs_symmetric(tc: hegel::TestCase) {
    let x = tc.draw(generators::integers::<i64>());
    assert_eq!(my_abs(x), my_abs(-x));
}
```

### Step 3: Choose Generators

Start with the **broadest generator that matches the function's input type**.
Do not restrict the range to match the original test's examples. The whole
point is to explore inputs the original author didn't think of.

### Step 4: Adjust the Oracle

Unit tests often compare against a hardcoded expected value. PBTs need an
oracle that works for any input:

- **Use a reference implementation:** `assert_eq!(my_sort(v), v.sort())`
- **Use a structural property:** `assert!(is_sorted(my_sort(v)))`
- **Use a relationship:** `assert_eq!(my_abs(x), my_abs(-x))`

If you can't find a general oracle, the test may not be a good PBT candidate.

### Step 5: Handle Edge Cases

When the PBT finds failures on inputs the unit tests didn't cover, decide:

- **Is this a real bug?** Fix the code. This is PBT doing its job.
- **Is this outside the function's domain?** Add a constraint — but document
  why, and check whether the function's documentation should be updated.

## Example Transformations

### Parsing round-trip

Before:

```rust
#[test]
fn test_parse_int() {
    assert_eq!("123".parse::<i32>().unwrap(), 123);
    assert_eq!("-1".parse::<i32>().unwrap(), -1);
    assert_eq!("0".parse::<i32>().unwrap(), 0);
}
```

After:

```rust
#[hegel::test]
fn test_int_display_parse_round_trip(tc: hegel::TestCase) {
    let n = tc.draw(generators::integers::<i32>());
    let s = n.to_string();
    assert_eq!(s.parse::<i32>().unwrap(), n);
}
```

### Collection operations

Before:

```rust
#[test]
fn test_push_pop() {
    let mut stack = Stack::new();
    stack.push(1);
    stack.push(2);
    assert_eq!(stack.pop(), Some(2));
    assert_eq!(stack.pop(), Some(1));
    assert_eq!(stack.pop(), None);
}
```

After:

```rust
#[hegel::test]
fn test_push_then_pop_returns_last(tc: hegel::TestCase) {
    let items: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    let mut stack = Stack::new();
    for &item in &items {
        stack.push(item);
    }
    // Property: popping returns items in reverse order
    for &item in items.iter().rev() {
        assert_eq!(stack.pop(), Some(item));
    }
    assert_eq!(stack.pop(), None);
}
```

### Encoding/decoding

Before:

```rust
#[test]
fn test_base64_encode() {
    assert_eq!(base64_encode(b"hello"), "aGVsbG8=");
    assert_eq!(base64_encode(b""), "");
    assert_eq!(base64_encode(b"a"), "YQ==");
}
```

After:

```rust
#[hegel::test]
fn test_base64_round_trip(tc: hegel::TestCase) {
    let data: Vec<u8> = tc.draw(generators::binary());
    let encoded = base64_encode(&data);
    let decoded = base64_decode(&encoded).unwrap();
    assert_eq!(decoded, data);
}

#[hegel::test]
fn test_base64_output_is_valid(tc: hegel::TestCase) {
    let data: Vec<u8> = tc.draw(generators::binary());
    let encoded = base64_encode(&data);
    // Property: output contains only valid base64 characters
    assert!(encoded.chars().all(|c| c.is_ascii_alphanumeric() || c == '+' || c == '/' || c == '='));
}
```

### Sorting

Before:

```rust
#[test]
fn test_sort() {
    assert_eq!(my_sort(&mut [3, 1, 2]), [1, 2, 3]);
    assert_eq!(my_sort(&mut [1]), [1]);
    assert_eq!(my_sort(&mut Vec::<i32>::new()), Vec::<i32>::new());
}
```

After:

```rust
#[hegel::test]
fn test_sort_is_sorted(tc: hegel::TestCase) {
    let v: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    let sorted = my_sort(&v);
    for w in sorted.windows(2) {
        assert!(w[0] <= w[1]);
    }
}

#[hegel::test]
fn test_sort_is_permutation(tc: hegel::TestCase) {
    let v: Vec<i32> = tc.draw(generators::vecs(generators::integers()));
    let sorted = my_sort(&v);
    let mut expected = v.clone();
    expected.sort();
    assert_eq!(sorted, expected);
}
```

## Research Insights

Studies of evolving unit tests into property-based tests have found:

- **Most PBTs use simple generators.** Around 65% of property-based tests in
  practice use only basic generators (integers, strings, lists) without complex
  composition. Don't over-engineer generators.

- **PBTs find bugs that unit tests miss.** Even when unit tests pass, PBTs
  can discover failures — particularly around boundary conditions, empty inputs,
  and large values. In one study, PBTs found bugs in ~2% of cases where the
  corresponding unit tests all passed.

- **Parameterized unit tests are a stepping stone.** If you can't immediately
  see the right property, start by parameterizing the test (replacing concrete
  values with generated ones and keeping a simple oracle). You can refine the
  property later.

- **The biggest gain is coverage of edge cases.** PBTs typically add modest
  line coverage over unit tests, but their value is in exercising combinations
  and boundary conditions that humans don't think to write by hand.
