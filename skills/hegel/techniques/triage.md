# Failures: triage and reporting

Keep a running ledger of every failing test you observe. Each entry ends in exactly one of:

- a finding: reduced, root-caused in the library, reported;
- a written note that the bug was in your test, with the fix.

A finding is not done until its standalone repro has been compiled and run and you have watched it fail. A repro written from memory of the API is a guess; check every name and signature against the source, like any other test.

Decide bug-in-code vs bug-in-test from evidence, never from convenience. Never silently delete, weaken, or narrow a failing property.

Write the final report from a fresh run of the full suite, not from memory: every failure in that run appears in the report, and any earlier failure absent from it gets its ledger note cited.
