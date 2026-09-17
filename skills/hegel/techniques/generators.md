# Generators

Default to the full documented domain of every parameter the API exposes, including construction and configuration knobs: capacities, degrees, precisions, radii, feature toggles.

- The full domain includes hostile inputs: empty input, control characters and NUL, extreme sizes and nesting, invalid shapes alongside valid ones.
- A generator targeting a structured subset (inputs that parse, problems with a known feasible point) is often the sharpest tool: construct members of the subset directly rather than filtering or clamping the full domain, and keep a full-domain property alongside it.
- Bound resource use where values are materialized (collection sizes, recursion depth), not in the drawn domain.
