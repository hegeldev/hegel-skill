# hegel-skill

An [Agent Skill](https://agentskills.io/home) that teaches agents how to write property-based tests using [Hegel](https://hegel.dev/).
The skill should be language agnostic and work with any Hegel implementation. It may even be useful for other property-based testing libraries, but this has not been validated.

When you ask an agent to write property-based tests, this skill provides guidance on how to identify things to test and validate the tests written.

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
