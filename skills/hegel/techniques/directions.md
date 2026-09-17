# Test both directions

For every contract, state two kinds of property:

- **Accept**: what valid input must produce — roundtrips, invariants, agreement with a model or reference implementation.
- **Reject**: what invalid or hostile input must *not* do — be accepted, crash, hang, or corrupt state silently. Parsers and validators earn most of their bugs here: feed them near-valid input (wrong family, out-of-range field, leading zeros, trailing garbage, misplaced separators) and assert rejection.

Test documented claims exactly as written — an off-by-one against the docs is a bug even when the behavior looks sane.
