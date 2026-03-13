# Porting from Other Property-Based Testing Libraries

When a project already has property-based tests in another framework, porting
to hegel is usually mechanical. The core concepts are the same — generate
random inputs, check a property, shrink failures — but the API surface differs.

## General Principles

### Hegel is imperative

Most PBT libraries use a declarative style: you describe what to generate in a
function signature or strategy combinator, and the framework calls your test
with the generated values. Hegel is imperative: your test function receives a
`TestCase` handle and calls `tc.draw()` whenever it needs a value.

This means:
- There's no limit on how many values you generate per test
- You can generate values conditionally (e.g., inside an `if` or loop)
- Later draws can depend on earlier values without needing `flat_map`

### Shrinking is automatic

Hegel's shrinking is handled server-side by Hypothesis. You don't implement
`Shrink` traits or define shrinking strategies. Every value drawn through
`tc.draw()` is automatically shrinkable.

### Standard assertions

Hegel uses standard `assert!` / `assert_eq!` macros. No special `prop_assert!`
or return-a-bool pattern needed.

## What to Port and What to Rewrite

Not every existing PBT is worth porting line-for-line. Before mechanically
translating, consider:

- **Is the existing test over-constrained?** Many proptest/quickcheck tests use
  narrow strategies because shrinking was slow or unreliable. Hegel's shrinking
  is more robust — try broader generators first.
- **Are the generators too complex?** If the existing test has elaborate strategy
  combinators just to produce valid inputs, hegel's imperative style might let
  you simplify significantly with sequential `tc.draw()` calls.
- **Is the property still the right one?** Porting is a good time to reassess.
  The existing test might test something trivial or use the implementation as
  its own oracle.
