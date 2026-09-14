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

Learn the API from, in order: existing hegel tests in this repo, the library's documentation, and the hegel library source available in your environment (vendored or registry copy). Check the exact name and signature of everything you use. Do not guess syntax.

Ground every property in evidence — documented contracts, names and signatures, invariants the code itself asserts, existing tests. An undocumented but ordinary expectation (a round-trip, two equivalent spellings agreeing, a rescaled input giving the same answer) is evidence too: name the expectation, and when such a test fails, judge the failure by whether the behavior is defensible, not by whether anything promised it. Write one property per test, in the project's existing test files.

## Techniques

Short, specific guides in this skill's `techniques/` directory. Load one when you reach that stage, when you are stuck, or when a result surprises you:

- `techniques/surfaces.md` — choosing what to test: the full public API, sibling entry points, every instantiation
- `techniques/directions.md` — properties in the accept and the reject direction
- `techniques/generators.md` — generators: full domains, hostile inputs, targeted subsets
- `techniques/scale.md` — the scale probe for recursion and complexity bugs
- `techniques/running.md` — case counts and run configuration
- `techniques/triage.md` — investigating failures and reporting honestly

Before stopping, re-check your surface list (from `techniques/surfaces.md`) against the library's public API index: every public module, and every sibling entry point and instantiation of a surface you tested, gets a test or a stated reason. Then check each tested surface against this whole menu — anything unchecked gets a stated reason — and review what you wrote against the `hegel-review` checklist.
