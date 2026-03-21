# ADR 004: System Prompt Versioning

## Status
Accepted

## Context
Changing a system prompt can shift AI behavior in unexpected ways. Without tracking which prompt version produced which output, you cannot diagnose regressions or compare performance across iterations.

## Decision
Assign a `PROMPT_VERSION` constant to each system prompt. Store it alongside every production output (database column, API response metadata). Include it in eval results. Compare eval pass rates across versions.

## Consequences

**Good:**
- Eval reports group results by prompt version. You can see that v2 scores 8% lower on accuracy than v1.
- Production outputs are traceable to the prompt that generated them. If users report bad results, you can check which version was active.
- Enables A/B testing by deploying two versions and comparing output quality.
- Cheap to implement: one constant, one database column.

**Bad:**
- Requires discipline to bump the version when changing the prompt. Forgetting defeats the purpose.
- Version is a string, not a diff. You still need git history to see what changed between v1 and v2.

## Alternatives considered
- Git commit hash as version: automatic, but unreadable in reports and database queries.
- No versioning: simpler, but prompt regressions are invisible until users complain.
- Full prompt text stored with each output: complete, but bloats storage and makes queries slow.
