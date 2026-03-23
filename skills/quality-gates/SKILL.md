---
name: quality-gates
description: Use when setting up or tightening code quality standards for a TypeScript project. Provides ESLint, Vitest, TypeScript, and Prettier configs with enforced complexity limits, coverage thresholds, and pre-push hooks.
---

# Quality Gates

ESLint, TypeScript, Vitest, and Prettier configs that enforce real standards. Complexity limits that prevent spaghetti. Coverage thresholds that prevent shipping blind.

## When to use

- Starting a new TypeScript project and want strict defaults
- Tightening quality on an existing project
- Adding coverage thresholds to CI
- Setting up pre-push hooks to block bad code

## ESLint config

Create `eslint.config.js`:

```javascript
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import sonarjs from "eslint-plugin-sonarjs";

export default [
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  sonarjs.configs.recommended,
  {
    languageOptions: {
      parserOptions: {
        projectService: {
          allowDefaultProject: ["eslint.config.js", "vitest.config.ts"],
        },
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      // Complexity
      complexity: ["error", 10],
      "sonarjs/cognitive-complexity": ["error", 12],

      // Size
      "max-lines": ["error", { max: 200, skipBlankLines: true, skipComments: true }],
      "max-lines-per-function": [
        "error",
        { max: 60, skipBlankLines: true, skipComments: true },
      ],
      "max-params": ["error", 4],

      // Safety
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      "@typescript-eslint/restrict-template-expressions": "error",
      "no-console": "error",
      eqeqeq: ["error", "always"],
      "prefer-const": "error",
    },
  },
  {
    // Relax limits for test files
    files: ["**/*.test.ts", "**/*.spec.ts", "tests/**/*.ts", "evals/**/*.ts"],
    rules: {
      "max-lines": "off",
      "max-lines-per-function": "off",
      "@typescript-eslint/no-floating-promises": "off",
    },
  },
  {
    // Ignore config files from type-aware linting
    files: ["*.config.*"],
    rules: {
      "max-lines": "off",
      "@typescript-eslint/no-unsafe-assignment": "off",
      "@typescript-eslint/no-unsafe-member-access": "off",
    },
  },
];
```

**Note:** The config uses a plain array export (not `tseslint.config()` wrapper) to avoid plugin namespace resolution issues with `strictTypeChecked`. The `allowDefaultProject` option in `parserOptions` lets ESLint type-check root config files that sit outside your `tsconfig.json` include paths.

Requires:
```bash
npm install -D eslint@^9 @eslint/js typescript-eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint-plugin-sonarjs
```

**Important:** Pin `eslint@^9`. The flat config conventions in this setup target ESLint 9. The rule is `complexity` (not `max-complexity`, which was removed in newer versions).

## TypeScript config

Add to your `tsconfig.json` compilerOptions:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true
  }
}
```

Each flag catches a different class of bug:
- `strict`: enables all strict type checks
- `noUnusedLocals`: dead code detection
- `noUnusedParameters`: unused function args
- `noFallthroughCasesInSwitch`: missing break statements
- `noUncheckedIndexedAccess`: array/object indexing returns `T | undefined`

**Note:** Your `tsconfig.json` `include` array must point to a directory that contains at least one `.ts` file, or TypeScript will error with TS18003. If you're starting fresh, create a placeholder like `src/index.ts` before running `tsc --noEmit`.

## Vitest config

Create `vitest.config.ts`:

```typescript
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      thresholds: {
        statements: 85,
        branches: 78,
        functions: 88,
        lines: 85,
      },
      exclude: [
        "**/*.config.*",
        "evals/**",
        "twins/**",
        "docs/**",
      ],
    },
    testTimeout: 10_000,
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "."),
    },
  },
});
```

Requires:
```bash
npm install -D vitest @vitest/coverage-v8
```

**Note on fresh projects:** Coverage thresholds only apply when there are source files to measure. On a project with no `src/` files yet, `vitest run --coverage` will either pass vacuously or error depending on your setup. Start with thresholds at 0 and ratchet up as you add source files and tests, or wait to enable thresholds until you have your first module and test in place.

## Prettier config

Create `.prettierrc`:

```json
{
  "printWidth": 100,
  "tabWidth": 2,
  "trailingComma": "all",
  "semi": true,
  "singleQuote": false
}
```

Requires:
```bash
npm install -D prettier
```

## Pre-push hook

Create `.githooks/pre-push`:

```bash
#!/bin/sh
npm run quality || exit 1
```

Enable:

```bash
chmod +x .githooks/pre-push
git config core.hooksPath .githooks
```

## Package scripts

Add to `package.json`:

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "quality": "npm run typecheck && npm run lint && npm run test:coverage"
  }
}
```

## Install

1. Copy configs into your project root
2. Install dependencies (see each section — note the `eslint@^9` pin)
3. Make sure `tsconfig.json` includes at least one directory with a `.ts` file
4. Add scripts to package.json
5. Set up git hooks: `git config core.hooksPath .githooks`
6. Run `npm run quality` to verify

## Tuning thresholds

Start strict. Relax if needed. These defaults come from a production AI agent with 258 tests.

If 85% coverage is too high for a new project, start at 0% and ratchet up as you add tests. The threshold should represent "below this, we're shipping blind."

Complexity limits (10 cyclomatic, 12 cognitive) are tight. If a function hits them, it needs refactoring, not a higher limit. Extract helpers, use early returns, simplify conditionals.

File size limit (200 lines) forces modular code. If a file grows past it, split by responsibility. The limit excludes blanks and comments, so 200 lines means 200 lines of logic.
