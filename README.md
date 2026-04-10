# hegel-skill

An [Agent Skill](https://agentskills.io/home) that teaches agents how to write property-based tests using [Hegel](https://hegel.dev/).

When you ask an agent to write property-based tests, this skill provides:

- A methodology for identifying testable properties from code evidence
- Generator discipline guidelines to avoid over-constraining inputs
- Language-specific API references and idiomatic patterns
- Guidance on evolving existing unit tests into property-based tests

This currently supports [hegel-rust](https://github.com/hegeldev/hegel-rust) and [hegel-go](https://github.com/hegeldev/hegel-go). We will update it for each language's Hegel library as new ones come out.

## Installation

`hegel-skill` works with Claude Code, Codex, and any agent that supports the Agent Skills standard.

### Claude Code

Add this repository as a marketplace, and install the skill:

```bash
/plugin marketplace add hegeldev/hegel-skill
/plugin install hegel-skill@hegeldev-hegel-skill
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
