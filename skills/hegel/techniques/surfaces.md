# Choosing what to test

Before writing any test, list the crate's risky surfaces: parsers and decoders, arithmetic and boundary logic, construction and configuration parameters, optimized or unsafe paths, stateful APIs, anything with an assert or a documented precondition, and any operation with a documented complexity bound.

Rank by risk — hand-written low-level code over derived or trivial code. Keep the list: it is your coverage checklist before stopping.
