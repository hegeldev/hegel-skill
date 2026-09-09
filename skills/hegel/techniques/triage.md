# Failures: triage and reporting

Keep a running ledger of every failing test you observe. Each entry ends in exactly one of:

- a finding: reduced, root-caused in the library, reported;
- a written note that the bug was in your test, with the fix.

Decide bug-in-code vs bug-in-test from evidence, never from convenience. Never silently delete, weaken, or narrow a failing property.

Write the final report from a fresh run of the full suite, not from memory: every failure in that run appears in the report, and any earlier failure absent from it gets its ledger note cited.
