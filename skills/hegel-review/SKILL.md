---
name: hegel-review
description: >
  Review property-based tests for the failure modes that make them weak or
  misleading. Use after writing property-based tests with hegel (or
  proptest, quickcheck, fast-check, jqwik, etc.), when asked to review
  property-based tests, or before shipping a change to a PBT suite.
---

# Reviewing property-based tests

Check every test against this list. When a check fails, quote the offending code and give the smallest change that fixes it.

1. **Narrowed generators**: drawn domains trimmed to dodge failures — ranges shrunk, characters filtered, sizes capped in the domain rather than at materialization. The domain should be as wide as the contract.
2. **Fixed inputs in property clothing**: a "property" that only ever sees one value or a few hand-picked cases.
3. **Missing direction**: a parser or validator tested only on valid input; or invalid input checked only for "doesn't crash" when rejection should be asserted.
4. **More than one property per test**: assertions about unrelated contracts sharing one test and one failure signal.
5. **Evidence-free properties**: asserting behavior nothing documents or implies, or restating the implementation instead of a contract.
6. **Tolerance hacks**: an epsilon widened until the test passes; approximate equality where the contract is exact.
7. **Weakened oracles**: asserting only `is_ok()`, only a length, or only "doesn't panic" where the actual value could be checked cheaply.
8. **`should_panic` or ignored tests**: tests that pass while the bug is present, or are excluded from the default run.
9. **Config parameters fixed**: constructor and configuration knobs pinned to defaults instead of generated.
10. **Untested variants**: one property instantiated for one type or variant when the contract covers several; or the same property copy-pasted with drift.
11. **Silent-triage residue**: commented-out asserts, deleted failing tests, TODOs hiding a red test.
12. **Resource bounds in the wrong place**: recursion depth or collection size limited in the drawn domain instead of at materialization.
