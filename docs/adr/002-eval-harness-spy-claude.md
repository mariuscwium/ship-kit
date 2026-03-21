# ADR 002: Eval Harness with SpyClaude

## Status
Accepted

## Context
AI-integrated apps have two categories of bugs: code bugs (your logic mishandles Claude's response) and prompt bugs (Claude gives the wrong response to your prompt). Integration tests with StubClaude catch the first category. They cannot catch the second.

## Decision
Run a separate eval suite against the real Claude API. Use a SpyClaude decorator that wraps the real client, records tool calls and responses, and exposes them for assertions.

Eval scenarios define inputs (messages, images) and assertions (response contains X, tool Y was called, JSON schema valid, score above threshold). The runner executes each scenario, collects results, and reports pass/fail.

## Consequences

**Good:**
- Catches prompt regressions before production. If you change the system prompt and a scoring rubric shifts, the eval fails.
- SpyClaude is transparent to application code. No test-specific branches.
- Assertions cover response text, tool calls, structured output, and scoring consistency.
- Prompt version tracking links eval results to specific prompt iterations.
- Fixed date injection makes time-dependent responses reproducible.

**Bad:**
- Evals call the real API. Each run costs $0.01-0.05. Not free.
- Claude's responses are non-deterministic. Some assertions need tolerance (scoring within a range, response contains one of several valid phrases).
- Evals are slower than unit tests (1-5 seconds per scenario). Run them separately from the main test suite.
- Variance testing (same input scored multiple times) multiplies cost.

## Alternatives considered
- Test only with StubClaude: fast and free, but doesn't test the prompt. Prompt regressions ship undetected.
- Manual testing: ask Claude in the playground and eyeball the response. Doesn't scale, no regression detection.
- Snapshot testing: record Claude's response and assert exact match. Brittle. Breaks on harmless phrasing changes.
