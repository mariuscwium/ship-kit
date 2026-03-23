# Agentic Development Harness - TypeScript

Quality harnesses for AI-integrated web apps. Add what you need.

## Harnesses

| Harness | What it does | Install time |
|---------|-------------|--------------|
| [`twins/`](skills/twins/SKILL.md) | Test without network calls (Supabase, Claude, Redis) | 10 min |
| [`evals/`](skills/evals/SKILL.md) | Catch prompt regressions with real Claude API | 15 min |
| [`quality-gates/`](skills/quality-gates/SKILL.md) | ESLint, Vitest, TypeScript strict configs | 5 min |
| [`a11y-gates/`](skills/a11y-gates/SKILL.md) | WCAG + Lighthouse checks in CI | 10 min |
| [`di-pattern/`](skills/di-pattern/SKILL.md) | Testable dependency injection (3 files, no framework) | 5 min |

Each harness has its own docs with install steps and code examples. They work independently.

## Stack

Built for: TypeScript, Next.js, Supabase, Claude API.

Adaptable to other stacks. The patterns are transferable. The code examples assume this stack.

## Install as Claude Code plugin

Add to your Claude Code settings:

```json
{
  "extraKnownMarketplaces": {
    "ship-kit": {
      "source": {
        "source": "github",
        "repo": "mariuscwium/ship-kit"
      }
    }
  }
}
```

Then enable:

```json
{
  "enabledPlugins": {
    "ship-kit@ship-kit": true
  }
}
```

## Install manually

Clone and copy the harnesses you need into your project:

```bash
git clone https://github.com/mariuscwium/ship-kit.git
cp -r ship-kit/skills/twins/ your-project/twins/
```

## Architecture decisions

Each harness is backed by an ADR explaining why it exists:

- [001: Digital Twins Over Mocks](docs/adr/001-digital-twins-over-mocks.md)
- [002: Eval Harness with SpyClaude](docs/adr/002-eval-harness-spy-claude.md)
- [003: Dependency Injection via Interfaces](docs/adr/003-dependency-injection-via-interfaces.md)
- [004: System Prompt Versioning](docs/adr/004-prompt-versioning.md)
- [005: Accessibility in CI](docs/adr/005-accessibility-first.md)

## Origin

These patterns are extracted from [Zoe the Robot](https://github.com/mariuscwium/zoe_the_robot), a production AI agent with 258 tests, digital twins for 5 services, and a prompt eval harness. Built with AI-native development.

## License

MIT
