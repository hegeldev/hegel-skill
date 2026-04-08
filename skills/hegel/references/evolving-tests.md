# Evolving Unit Tests into Property-Based Tests

This guide helps you recognize what property a unit test is hiding and translate it into a hegel PBT. Examples use pseudocode — see the language reference for exact syntax.

## Recognizing Properties in Unit Tests

### Multiple tests, same assertion shape

```pseudocode
# Before: three tests that all do the same thing with different inputs
test parse_1:  assert parse("1") == 1
test parse_42: assert parse("42") == 42
test parse_neg: assert parse("-7") == -7
```

These are all instances of a **roundtrip** property: `parse(format(n)) == n`.

```pseudocode
# After
n = tc.draw(integers())
assert parse(format(n)) == n
```

### Loop over hardcoded examples

```pseudocode
# Before
for (input, expected) in [("a", "A"), ("hello", "HELLO"), ("ABC", "ABC")]:
    assert to_upper(input) == expected
```

The loop body is the property — but using `expected` as the oracle won't generalize. Look for a structural property instead: the output should equal the input when compared case-insensitively, and every character in the output should be uppercase.

```pseudocode
# After
s = tc.draw(text())
result = to_upper(s)
assert result.lower() == s.lower()  # case-insensitive equality
```

### Tests asserting "no error" on various inputs

```pseudocode
# Before
test parse_empty:   assert parse("").is_ok()
test parse_garbage: assert parse("xyz").is_ok() or parse("xyz").is_err()  # just shouldn't crash
test parse_unicode: assert parse("café").is_ok() or ...
```

This is a **robustness** property: the function should handle any input without panicking.

```pseudocode
# After
s = tc.draw(text())
_ = parse(s)  # should never panic, error results are fine
```

### Setup, operations, check final state

```pseudocode
# Before
test stack_operations:
    s = Stack()
    s.push(1)
    s.push(2)
    assert s.pop() == 2
    assert s.pop() == 1
    assert s.is_empty()
```

This is a **stateful model test** candidate. The test encodes a specific operation sequence — generalize it by drawing random operations. See the stateful testing section in the language reference.

### Tests with manually seeded RNGs

```pseudocode
# Before
test sample_distribution:
    rng = ChaChaRng(seed=42)
    result = sample(weights, rng)
    assert result in valid_range
```

Replace the seeded RNG with hegel's random generator. The fixed seed gives reproducibility but prevents exploration — hegel gives you both exploration and shrinkable counterexamples.

### Multiple tests asserting the same invariant

```pseudocode
# Before
test sort_empty:    assert sort([]) == []
test sort_single:   assert sort([5]) == [5]
test sort_reversed: assert sort([3,2,1]) == [1,2,3]
test sort_sorted:   assert sort([1,2,3]) == [1,2,3]
```

Every test checks that the output is sorted and is a permutation of the input. Those are the properties.

```pseudocode
# After: two separate properties
v = tc.draw(lists(integers()))
result = sort(v)
assert is_sorted(result)

v = tc.draw(lists(integers()))
assert sorted(sort(v)) == sorted(v)  # same elements
```

## What to Do with the Old Tests

**Usually the PBT subsumes them.** If your PBT covers the full input space, the specific examples in the unit tests are redundant — hegel will explore those cases and many more.

**Keep edge-case tests that serve as documentation.** If a unit test encodes a subtle edge case that was discovered through a bug report, it may be worth keeping as documentation even if the PBT covers it. The unit test communicates "this specific case matters" in a way a PBT doesn't.

**Replace inline, don't create a new file.** Add hegel tests in the same file where the unit tests live. Either replace the unit tests or add the PBTs alongside them.
