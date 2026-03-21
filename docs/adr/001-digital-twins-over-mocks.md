# ADR 001: Digital Twins Over Mocks

## Status
Accepted

## Context
Projects that call external APIs (databases, AI models, storage) need test doubles. The standard approach is `jest.mock()` or `vi.mock()`, which replaces modules at import time.

## Decision
Use stateful in-memory behavioral clones ("digital twins") instead of mocks. Each twin implements the same interface as the production client, maintains internal state, and enforces real API contracts.

## Consequences

**Good:**
- Twins catch bugs that mocks miss. A mock won't throw WRONGTYPE when you LPUSH to a string key. A Redis twin will, the same way Redis does in production.
- Twins enforce the interface at compile time. If the interface changes, the twin fails to compile. A mock just silently passes.
- Twins maintain state across a test. You can insert a row, query it, update it, and verify the result. Mocks require manual wiring for each step.
- Tests run without network access, credentials, or running services.

**Bad:**
- Twins require upfront investment (~50-150 lines per service).
- Twins can drift from the real API if not maintained. They implement a subset, not the full API.
- Complex query builders (Supabase, Prisma) are hard to twin completely. Start with the subset you use.

## Alternatives considered
- `jest.mock()` / `vi.mock()`: replaces the module, not the behavior. Tests pass but production breaks.
- Docker test containers: real services, but slow startup, network required, flaky in CI.
- MSW (Mock Service Worker): intercepts HTTP, but you're still testing HTTP serialization, not business logic.
