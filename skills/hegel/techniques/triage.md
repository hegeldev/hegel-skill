# Failures: triage and reporting

Keep a running ledger of every failing test you observe. Each entry ends in exactly one of:

- a finding: reduced, root-caused in the library, reported;
- a written note that the bug was in your test, with the fix.

A finding is not done until its standalone repro has been compiled and run and you have watched it fail. A repro written from memory of the API is a guess; check every name and signature against the source, like any other test.

Decide bug-in-code vs bug-in-test from evidence, never from convenience. Never silently delete, weaken, or narrow a failing property.

There is no third verdict. "A floating-point limitation", "a degenerate input", "the docs don't pin this down" — these assess a finding's severity, not whether it exists. When the failing property states an exact contract — an equality, ordering, or hashing law, a trait or protocol contract, a roundtrip or endpoint the docs state exactly — there is no tolerance to appeal to: a disagreement of one ulp or one nanosecond is a finding. Floating-point precision may explain the mechanism of such a failure; it never changes the verdict.

Before loosening a failing oracle as approximate or underspecified, check the library's own evidence for the disputed value: the documentation's exact words, sibling implementations of the same operation (other algorithms, modes, backends), downstream consumers, the changelog. One path disagreeing with all its siblings is a finding even where the docs are silent. A tolerance added, or a value class (subnormals, tiny magnitudes, exact bounds) excluded from the generator, after you watched it fail is a weakening like any other, however well-commented — legitimate only with a filed finding standing behind it.

Write the final report from a fresh run of the full suite, not from memory: every failure in that run appears in the report, and any earlier failure absent from it gets its ledger note cited.
