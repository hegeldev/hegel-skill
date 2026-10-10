# Adversarial review before finishing

The cases you failed to imagine when writing the code are the same ones you fail to generate when testing it: your tests inherit the blind spots of your implementation. A reviewer who never saw your code or your tests does not share those blind spots, and finds the property you could not.

So before you stop, launch a **subagent** (the Task tool, or your harness's equivalent) to review the implementation adversarially. It must be a separate agent — reviewing your own work reproduces the gaps you already have, which is the whole failure this step exists to catch.

Give the subagent the implementation and the contract it must meet (doc comments, README, type signatures, documented complexity bounds). Do **not** give it your tests: it derives what should hold from the contract alone, so its properties do not inherit your oracle. Tell it to assume the code is wrong until shown otherwise, and to return:

- **properties** the contract implies — roundtrips, invariants, agreement between equivalent paths, the documented meaning of named values — that a test could check;
- for each, the **input classes** most likely to break it: empty, singleton, degenerate, boundary, maximal, malformed, and anything the documentation singles out;
- which **public surface** exercises each.

Take what it returns, write those as hegel properties (one per test), run them, and fix what they catch. Drop one only with a stated reason — it duplicates a test you already have, or you cannot ground it. Everything else is written and run before you finish.
