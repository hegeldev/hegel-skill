# hegel-skill

An [Agent Skill](https://agentskills.io/home) that teaches agents how to write property-based tests using [Hegel](https://hegel.dev/).

When you ask an agent to write property-based tests, this skill provides:

- A small core: ground every property in evidence, learn the API from the project's own tests, docs, and library source
- A menu of technique guides loaded on demand: choosing surfaces, testing both directions, generator breadth, the scale probe, case counts, and failure triage
- A pre-stop coverage walk that checks every surface against the whole menu

It ships with a second skill, `hegel-review`: a twelve-point checklist of the failure modes that make property-based tests weak or misleading, applied to tests after they are written (whether or not they use hegel).

Supported hegel libraries:
* [hegel-rust](https://github.com/hegeldev/hegel-rust)
* [hegel-go](https://github.com/hegeldev/hegel-go)
* [hegel-cpp](https://github.com/hegeldev/hegel-cpp)
* [hegel-typescript](https://github.com/hegeldev/hegel-typescript)
* [hegel-java](https://github.com/hegeldev/hegel-java)
* [hegel-ocaml](https://github.com/hegeldev/hegel-ocaml)

## Installation

`hegel-skill` works with Claude Code, Codex, and any agent that supports the Agent Skills standard.

### Claude Code

Add this repository as a marketplace, and install the skill:

```bash
/plugin marketplace add hegeldev/hegel-skill
/plugin install hegel-skill@hegeldev
```

Or see https://code.claude.com/docs/en/skills for local installation instructions.

### Codex

Use the built-in skill installer:

```
$skill-installer install https://github.com/hegeldev/hegel-skill/tree/main/skills/hegel
```

Or see https://developers.openai.com/codex/skills for local installation instructions.

### Other agents

Varies by installation. Refer to the docs of your agentic tool of choice.
