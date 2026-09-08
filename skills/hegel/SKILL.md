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

**API details come from the library, not from this skill.** Before writing your first test, find the library's actual API: a locally available copy (vendored or registry source, generated docs) if there is one, its published documentation otherwise. Check the exact name and signature of everything you use. Do not guess syntax.

## The loop

1. Read the code under test. List its risky surfaces: parsers and decoders, arithmetic and boundary logic, construction and configuration parameters, optimized or unsafe paths, stateful APIs, anything with an assert or a documented precondition.
2. For each surface, state properties grounded in evidence: documented contracts, names and signatures, invariants the code itself asserts, existing tests. Never test behavior nothing promises. Test documented claims exactly as written, and state properties in both directions — what valid input must produce, and what invalid or hostile input must not do (be accepted, crash, hang, corrupt silently).
3. Write one property per test, in the project's existing test files.
4. Generate broadly. Every parameter the API exposes is an input to generate, including construction and configuration knobs. Broad includes hostile: empty, control characters, extreme sizes and nesting, invalid shapes alongside valid ones. Never narrow a generator's domain to avoid a failure. Bound resource use where values are materialized (collection sizes, recursion depth), not in the drawn domain.
5. Run with high case counts — thousands, not the library default. Also probe scale directly, outside the generators: build one very large instance (hundreds of thousands of elements, deeply nested input) and exercise every operation and derived trait on it once — recursion and complexity bugs only appear at scale.
6. Investigate every failure to root cause. Decide bug-in-code vs bug-in-test from evidence and report it either way. Never silently delete, weaken, or narrow a failing property.
7. Before stopping, walk the surface list from step 1 and check each entry off against all of: properties in the accept direction, properties in the reject/hostile direction, and a scale probe where the surface holds data. Anything unchecked gets a stated reason.

Report what you tested, what you found, and what you left untested and why. Write the report from a fresh run of the full suite, not from memory: every test that fails in that run appears in the report, and any failure seen earlier that is absent from it gets a stated reason.
