# Generator breadth

Broad generators find bugs; narrow ones hide them.

- Every parameter the API exposes is a generated input, including construction and configuration knobs: capacities, degrees, precisions, radii, feature toggles.
- Broad includes hostile: empty input, control characters and NUL, extreme sizes and nesting, invalid shapes alongside valid ones.
- Never narrow a generator's domain to avoid a failure — investigate the failure instead.
- Bound resource use where values are materialized (collection sizes, recursion depth), not in the drawn domain.
