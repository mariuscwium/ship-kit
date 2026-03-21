---
name: a11y-gates
description: Use when adding accessibility and performance checks to a web project. Sets up axe-core WCAG testing in Playwright and Lighthouse CI with score thresholds.
---

# Accessibility and Performance Gates

Automated WCAG compliance and performance checks in CI. Catches accessibility regressions before deploy.

## When to use

- Adding accessibility testing to a Next.js or web project
- Setting up Lighthouse CI with performance budgets
- Ensuring WCAG 2.1 AA compliance on every deploy
- Adding bundle size limits to CI

## axe-core with Playwright

Create `tests/a11y/pages.spec.ts`:

```typescript
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

const pages = [
  { name: "home", path: "/" },
  { name: "login", path: "/login" },
  // Add your pages here
];

for (const page of pages) {
  test(`${page.name} has no WCAG AA violations`, async ({ page: p }) => {
    await p.goto(page.path);
    const results = await new AxeBuilder({ page: p })
      .withTags(["wcag2a", "wcag2aa"])
      .analyze();
    expect(results.violations).toEqual([]);
  });

  test(`${page.name} is keyboard navigable`, async ({ page: p }) => {
    await p.goto(page.path);

    // Tab through all focusable elements
    let focusCount = 0;
    const maxTabs = 50;
    for (let i = 0; i < maxTabs; i++) {
      await p.keyboard.press("Tab");
      const focused = await p.evaluate(() => {
        const el = document.activeElement;
        return el !== document.body ? el?.tagName : null;
      });
      if (focused) focusCount++;
    }

    // Page should have focusable elements
    expect(focusCount).toBeGreaterThan(0);
  });
}

test("mobile viewport has no violations", async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto("/");
  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa"])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

Checks:
- Color contrast ratios
- Alt text on images
- Focus management and tab order
- ARIA attributes
- Heading hierarchy
- Form labels and associations
- Keyboard navigation
- Mobile viewport accessibility

Requires:
```bash
npm install -D @playwright/test @axe-core/playwright
```

## Playwright config for a11y

Add an a11y project to `playwright.config.ts`:

```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  webServer: {
    command: "npm run dev",
    port: 3000,
    reuseExistingServer: true,
  },
  projects: [
    {
      name: "e2e",
      testDir: "./tests/e2e",
    },
    {
      name: "a11y",
      testDir: "./tests/a11y",
    },
  ],
});
```

## Lighthouse CI

Create `lighthouserc.json`:

```json
{
  "ci": {
    "collect": {
      "url": [
        "http://localhost:3000/",
        "http://localhost:3000/login"
      ],
      "startServerCommand": "npm run start",
      "startServerReadyPattern": "ready",
      "numberOfRuns": 3
    },
    "assert": {
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "categories:accessibility": ["error", { "minScore": 0.95 }],
        "categories:best-practices": ["error", { "minScore": 0.9 }],
        "categories:seo": ["error", { "minScore": 0.9 }],
        "resource-summary:script:size": [
          "error",
          { "maxNumericValue": 204800 }
        ]
      }
    }
  }
}
```

Thresholds:
- Performance: >= 90 (catches unoptimized images, render-blocking scripts)
- Accessibility: >= 95 (catches missing alt text, contrast, ARIA issues)
- Best Practices: >= 90 (catches deprecated APIs, console errors)
- SEO: >= 90 (catches missing meta, broken links)
- Bundle size: <= 200KB gzipped JS (catches dependency bloat)

Requires:
```bash
npm install -D @lhci/cli
```

## Package scripts

Add to `package.json`:

```json
{
  "scripts": {
    "test:e2e": "playwright test --project=e2e",
    "test:a11y": "playwright test --project=a11y",
    "lighthouse": "lhci autorun",
    "quality:full": "npm run quality && npm run test:a11y && npm run lighthouse"
  }
}
```

Run `quality:full` in CI. Run `quality` (typecheck + lint + coverage) on pre-push.

## Install

1. Install dependencies: `npm install -D @playwright/test @axe-core/playwright @lhci/cli`
2. Install browsers: `npx playwright install`
3. Copy `tests/a11y/pages.spec.ts` and add your pages
4. Copy `lighthouserc.json` and adjust URLs
5. Add a11y project to your Playwright config
6. Add scripts to package.json
7. Run `npm run test:a11y` to verify

## Tuning

The 95% Lighthouse accessibility score is high. If your project uses complex custom components (carousels, modals, drag-and-drop), you may need to drop to 90% and fix issues incrementally.

The 200KB bundle budget is for the total gzipped JS. If you're using heavy dependencies (charting libraries, rich text editors), adjust the threshold. The budget exists to catch accidental dependency additions, not to block intentional ones.

axe-core catches ~57% of WCAG issues automatically. The rest require manual testing (reading order, cognitive load, plain language). Automated checks are a floor, not a ceiling.
