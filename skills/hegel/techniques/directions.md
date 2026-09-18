# Test both directions

For every contract, state two kinds of property:

- **Accept**: what valid input must produce — roundtrips, invariants, agreement with a model or reference implementation.
- **Reject**: what invalid or hostile input must *not* do — be accepted, crash, hang, or corrupt state silently. Parsers and validators earn most of their bugs here: feed them near-valid input (wrong family, out-of-range field, leading zeros, trailing garbage, misplaced separators) and assert rejection.

Test documented claims exactly as written — an off-by-one against the docs is a bug even when the behavior looks sane.

## Pin the value, not just its validity

A property that only asserts the output is in-range, well-formed, or crash-free is the weakest oracle: it can fail only where the code loses control of the representation, usually at extreme inputs. Give every tested surface at least one property that pins what the output *means*:

- roundtrips measured against near-identity, not merely "converts back without error";
- the documented meaning of named values — when the docs say what zero, the empty input, an identity element, or an endpoint of a scale denotes, that sentence is a testable equation;
- agreement between paths the documentation presents as equivalent.

The two oracle strengths fail on different bugs: validity checks catch representation escapes, value-pinning oracles catch wrong answers. A surface covered only by validity checks is not covered.
