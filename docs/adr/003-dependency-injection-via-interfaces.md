# ADR 003: Dependency Injection via Interfaces

## Status
Accepted

## Context
Application code that imports SDK clients directly (`import Anthropic from "@anthropic-ai/sdk"`) cannot be tested without mocking the module. Module mocks are fragile, hide interface changes, and require test-specific configuration.

## Decision
Define TypeScript interfaces for each external service. Production code receives a `Deps` object as a function parameter. Production clients implement these interfaces by wrapping real SDKs. Twins implement them with in-memory state. Lazy singleton initialization in `prod-deps.ts` avoids throwing on missing env vars during test imports.

## Consequences

**Good:**
- Production code and test code use the same code path. No mock configuration, no `vi.mock()` calls.
- Interface changes break both production clients and twins at compile time. No silent drift.
- Lazy initialization means test files can import application modules without setting env vars.
- The Deps object is explicit about what external services a function needs. No hidden dependencies.
- No DI framework, no decorators, no reflection. TypeScript interfaces are the entire mechanism.

**Bad:**
- Every external service needs a wrapper in `clients.ts`. Small overhead per service.
- Interfaces must be maintained alongside the SDK. If you start using a new SDK method, add it to the interface.
- Functions take `deps` as a first parameter, which adds noise to call sites. This is the cost of explicit dependencies.

## Alternatives considered
- `vi.mock()` / `jest.mock()`: hides the interface, breaks silently on SDK changes, requires per-test configuration.
- Dependency injection frameworks (tsyringe, inversify): adds runtime overhead, decorators, and a learning curve for a problem that TypeScript interfaces solve at compile time.
- Global singletons: testable via `beforeEach` reassignment, but mutation-based testing is error-prone and order-dependent.
