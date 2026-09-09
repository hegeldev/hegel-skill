# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repo is the `hegel-skill` Agent Skill (distributed via the [Agent Skills](https://agentskills.io) standard). It teaches coding agents how to write property-based tests with [Hegel](https://hegel.dev) — a PBT protocol with language-specific libraries (currently Rust, Go, C++, TypeScript, Java, and OCaml), all powered by libhegel, a shared native engine based on Hypothesis and shipped by [hegel-rust](https://github.com/hegeldev/hegel-rust).

This repo is **content, not code**: there is no build, test, or lint step. Edits here directly change agent behavior, so wording and structure matter.

## Layout

```
.claude-plugin/         # Claude Code marketplace + plugin manifests
skills/hegel/
  SKILL.md              # Entry point — evidence rule, API read-order, technique index
  techniques/
    surfaces.md         # Choosing what to test
    directions.md       # Accept- and reject-direction properties
    generators.md       # Generator breadth: every parameter, hostile inputs
    scale.md            # The scale probe for recursion/complexity bugs
    running.md          # Case counts and run configuration
    triage.md           # Failure ledger, honest reporting
skills/hegel-review/
  SKILL.md              # Checklist for reviewing property-based tests
```

`SKILL.md` is a near-empty entry point: API details come from the project's existing hegel tests, the docs, and the installed library source, and each technique file is a short, specific prompt loaded on demand (progressive disclosure). Keep the entry small — new advice goes in a technique file plus one index line, and the index doubles as the pre-stop checklist, which is what makes techniques actually get applied.

Every line of skill content is benchmark-driven: it exists because a hegel-skill-bench case showed agents missing bugs without it. Don't add advice the benchmark hasn't justified.

## Conventions When Editing Skill Content

- **Language-agnostic.** No per-language syntax anywhere; the skill tells agents to read the library's own docs and source instead.
- **Evidence-based properties.** Properties must be grounded in the code under test (names, signatures, docs, existing tests, usage). Don't add advice that encourages inventing properties.
- **Don't weaken generator discipline.** "Broad generators find bugs" is load-bearing — edits that add hedges like "consider narrowing ranges for speed" undo the skill's main lesson. Resource bounds belong at materialization, never in the drawn domain.
- **One property per test.** Preserve this when adding examples.
- **Modify existing test files.** The skill tells agents not to create separate files for PBTs. Keep this consistent.
- **hegel-review mirrors the bench idiom rubric.** When the benchmark's rubric/checklist.json gains a bench-context item, consider mirroring it here (and vice versa).

## Adding a New Language

No per-language work is needed: the skill is language-agnostic and points agents at the installed library's own docs and source. Update the language list in `SKILL.md`'s frontmatter `description` and in `README.md` when Hegel ships a new library.

## Distribution

- `.claude-plugin/plugin.json` — plugin metadata; bump `version` on user-visible changes.
- `.claude-plugin/marketplace.json` — marketplace entry pointing at `hegeldev/hegel-skill` on GitHub.
- Installation paths (Claude Code `/plugin`, Codex `$skill-installer`, manual) are documented in `README.md`.
