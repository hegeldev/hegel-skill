# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repo is the `hegel-skill` Agent Skill (distributed via the [Agent Skills](https://agentskills.io) standard). It teaches coding agents how to write property-based tests with [Hegel](https://hegel.dev) — a PBT protocol with language-specific libraries (currently Rust, Go, C++, TypeScript, Java, and OCaml), all powered by libhegel, a shared native engine based on Hypothesis and shipped by [hegel-rust](https://github.com/hegeldev/hegel-rust).

This repo is **content, not code**: there is no build, test, or lint step. Edits here directly change agent behavior, so wording and structure matter.

## Layout

```
.claude-plugin/         # Claude Code marketplace + plugin manifests
skills/hegel/
  SKILL.md              # Entry point — methodology, property catalogue, generator discipline
  references/
    evolving-tests.md   # Shared: turning unit tests into PBTs
    rust/reference.md   # Rust API surface (hegeltest)
    rust/extras.md      # Rust extras integrations (chrono, jiff, serde_json, rand)
    rust/porting.md     # Porting from proptest/quickcheck -> hegel
    go/reference.md     # Go API surface
    go/porting.md       # Porting from rapid/gopter -> hegel
    cpp/reference.md    # C++ API surface (hegel-cpp, GTest integration)
    cpp/porting.md      # Porting from rapidcheck -> hegel
    typescript/reference.md # TypeScript API surface (@hegeldev/hegel)
    typescript/porting.md   # Porting from fast-check -> hegel
    java/reference.md       # Java API surface (dev.hegel:hegel, JUnit integration)
    java/porting.md         # Porting from jqwik/junit-quickcheck/QuickTheories -> hegel
    ocaml/reference.md      # OCaml API surface (hegel opam package, PPX deriver)
    ocaml/porting.md        # Porting from QCheck/Crowbar -> hegel
```

`SKILL.md` is loaded first; reference files are loaded lazily per the workflow in step 1. Keep that split intact — don't inline language-specific syntax into `SKILL.md`, and don't put cross-language methodology into the per-language references.

## Conventions When Editing Skill Content

- **`SKILL.md` uses pseudocode.** It is language-agnostic. Exact syntax lives in `references/<lang>/reference.md`.
- **Evidence-based properties.** The skill's core principle is that properties must be grounded in the code under test (names, signatures, docs, existing tests, usage). Don't add advice that encourages inventing properties.
- **Don't weaken generator discipline.** The "broad generators find bugs" stance in `SKILL.md` is load-bearing — edits that add hedges like "consider narrowing ranges for speed" undo the skill's main lesson.
- **One property per test.** Reinforced throughout; preserve this when adding examples.
- **Modify existing test files.** The skill explicitly tells agents not to create separate files for PBTs. Keep this consistent.
- **Tier 1 patterns are ordered by real-world bug-finding rate** (Model > Idempotence > Parse robustness > Roundtrip > Boundary). Don't reorder without a reason.
- **References must track released library versions.** When a hegel library ships a release, audit the corresponding `references/<lang>/` files against its CHANGELOG — stale API names (renamed generators, changed builder signatures) cause agent compile errors and, in the worst case, agents abandoning hegel entirely. Field testing found exactly this failure mode.

## Adding a New Language

When Hegel ships a new language library, mirror the existing structure: add `references/<lang>/reference.md` and `references/<lang>/porting.md`, and update the language list in `SKILL.md`'s frontmatter `description` and in `README.md`. The reference should cover the same sections as the Rust/Go references (setup, test structure, TestCase methods, generators, combinators, stateful testing, gotchas).

## Distribution

- `.claude-plugin/plugin.json` — plugin metadata; bump `version` on user-visible changes.
- `.claude-plugin/marketplace.json` — marketplace entry pointing at `hegeldev/hegel-skill` on GitHub.
- Installation paths (Claude Code `/plugin`, Codex `$skill-installer`, manual) are documented in `README.md`.
