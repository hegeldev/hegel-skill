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

# Hegel: property-based testing

Hegel generates random inputs for your code and shrinks failing cases to minimal counterexamples. Libraries exist for Rust (`hegeltest`), Go, C++, TypeScript, Java, and OCaml, all integrating with the standard test runner.

**API details come from the library, not from this skill.** Before writing your first test, read the hegel library copy available in your environment — vendored or registry source, generated docs, its README and examples — and check the exact name and signature of everything you use. Do not guess syntax.

## The loop

1. Read the code under test. List its risky surfaces: parsers and decoders, arithmetic and boundary logic, construction and configuration parameters, optimized or unsafe paths, stateful APIs, anything with an assert or a documented precondition.
2. For each surface, state properties grounded in evidence: documented contracts, names and signatures, invariants the code itself asserts, existing tests. Never test behavior nothing promises.
3. Write one property per test, in the project's existing test files.
4. Generate broadly. Every parameter the API exposes is an input to generate, including construction and configuration knobs. Never narrow a generator's domain to avoid a failure. Bound resource use where values are materialized (collection sizes, recursion depth), not in the drawn domain.
5. Run with high case counts — thousands, not the library default.
6. Investigate every failure to root cause. Decide bug-in-code vs bug-in-test from evidence and report it either way. Never silently delete, weaken, or narrow a failing property.
7. Before stopping, walk the surface list from step 1: each entry has properties, or a stated reason it does not.

Report what you tested, what you found, and what you left untested and why.
