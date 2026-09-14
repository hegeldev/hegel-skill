# Choosing what to test

Before writing any test, list the crate's risky surfaces: parsers and decoders, arithmetic and boundary logic, construction and configuration parameters, optimized or unsafe paths, stateful APIs, anything with an assert or a documented precondition, and any operation with a documented complexity bound.

Build the list from the library's own index of public modules and exports (top-level docs, re-exports, the package index) — enumerate mechanically first, then rank. Ranking sets the order you work in; it never removes an entry.

Entry points come in families — sibling methods of the same trait or interface, the same operation on each type it is implemented for, several operations or accessors taking the same input shape or answering the same question, and each type instantiation the docs advertise or users get by default (the narrowest float and any integer component types, not only the widest float, where arithmetic is most forgiving). List every member as its own line: a property on one member says nothing about its siblings, siblings must agree on the same state, and an input one member loudly rejects is a property for every sibling that accepts the same shape. Shapes a generator cannot draw (a duplicate key or type inside a structural input) get enumerated property variants.

Rank by risk — hand-written low-level code over derived or trivial code. Keep the list: it is your coverage checklist before stopping.
