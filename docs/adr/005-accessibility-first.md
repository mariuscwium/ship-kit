# ADR 005: Accessibility in CI

## Status
Accepted

## Context
Accessibility is treated as an afterthought in most projects. Issues accumulate until a compliance audit forces a painful remediation sprint. Automated checks catch ~57% of WCAG issues. The remaining 43% require manual testing, but catching 57% automatically is better than catching 0%.

## Decision
Run axe-core against rendered pages in Playwright tests. Fail CI on any WCAG 2.1 AA violation. Run Lighthouse CI with a 95% accessibility score threshold. Include keyboard navigation tests.

## Consequences

**Good:**
- Accessibility regressions are caught before merge. A missing alt text or broken focus trap fails the build.
- Developers learn WCAG rules by fixing violations, not by reading documentation.
- Lighthouse scores track performance, SEO, and best practices alongside accessibility.
- Bundle size budgets prevent accidental dependency bloat.

**Bad:**
- axe-core catches 57% of issues. The remaining 43% (reading order, cognitive load, plain language) require manual testing.
- Custom components (drag-and-drop, custom selects, complex modals) may need axe-core rule exclusions while being fixed.
- Lighthouse CI adds build time (~30 seconds per URL per run, 3 runs per URL).
- 95% accessibility threshold is high. New projects with third-party components may need to start at 90%.

## Alternatives considered
- Manual accessibility audits: thorough, but expensive and infrequent. Issues accumulate between audits.
- Linting only (eslint-plugin-jsx-a11y): catches some issues at write time, but misses runtime problems (dynamic content, focus management, color contrast with computed styles).
- No automated checks: the default. Accessibility issues ship and accumulate.
