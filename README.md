# hegel-skill

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that teaches agents how to write property-based tests using [Hegel](https://github.com/hegeldev/hegel-rust).

## Installation

Use as a plugin with Claude Code:

```bash
claude --plugin-dir path/to/hegel-skill
```

Or add to your project settings (`.claude/settings.json`):

```json
{
  "enabledPlugins": {
    "hegel-skill": true
  }
}
```

## What It Does

When you ask Claude Code to write property-based tests, fuzz code, or find edge cases, this skill provides:

- A methodology for identifying testable properties from code evidence
- Generator discipline guidelines to avoid over-constraining inputs
- Language-specific API references and idiomatic patterns
- Guidance on evolving existing unit tests into property-based tests

## Supported Languages

- Rust
