---
name: evals
description: Use when adding prompt quality evaluation to a project that uses Claude API. Creates a SpyClaude decorator, scenario runner, and assertion library for catching prompt regressions before production.
---

# Prompt Eval Harness

Tests Claude's actual behavior against the real API. Separate from integration tests (which use StubClaude). Catches prompt regressions before they reach production.

## When to use

- Adding AI features to a project and need to verify prompt quality
- Changing a system prompt and need to know what broke
- Testing vision, tool use, or structured output reliability
- Measuring scoring consistency across runs

## Components

### SpyClaude decorator

Create `evals/spy-claude.ts`:

```typescript
import type { ClaudeClient, ClaudeMessage, CreateMessageParams } from "@/lib/deps";

export interface ToolCall {
  name: string;
  input: Record<string, unknown>;
}

export class SpyClaude implements ClaudeClient {
  public toolCalls: ToolCall[] = [];
  public responses: ClaudeMessage[] = [];
  public calls: CreateMessageParams[] = [];

  constructor(private readonly real: ClaudeClient) {}

  async createMessage(params: CreateMessageParams): Promise<ClaudeMessage> {
    this.calls.push(params);
    const response = await this.real.createMessage(params);
    this.responses.push(response);

    for (const block of response.content) {
      if (block.type === "tool_use") {
        this.toolCalls.push({ name: block.name, input: block.input });
      }
    }

    return response;
  }

  reset(): void {
    this.toolCalls = [];
    this.responses = [];
    this.calls = [];
  }
}
```

### Assertion library

Create `evals/assertions.ts`:

```typescript
export interface EvalResult {
  scenario: string;
  response: string;
  toolCalls: Array<{ name: string; input: Record<string, unknown> }>;
  parsedJson: unknown;
  durationMs: number;
  promptVersion: string;
}

export type EvalAssertion =
  | { type: "response_contains"; text: string }
  | { type: "response_not_contains"; text: string }
  | { type: "response_contains_any"; texts: string[] }
  | { type: "valid_json" }
  | { type: "json_field_exists"; field: string }
  | { type: "tool_called"; name: string; minCount?: number }
  | { type: "tool_not_called"; name: string }
  | { type: "score_above"; field: string; threshold: number }
  | { type: "score_below"; field: string; threshold: number }
  | { type: "no_markdown" }
  | { type: "rejected"; reason_contains: string }
  | { type: "custom"; name: string; fn: (ctx: EvalResult) => boolean };

export interface AssertionResult {
  assertion: EvalAssertion;
  passed: boolean;
  detail?: string;
}

export function runAssertions(result: EvalResult, assertions: EvalAssertion[]): AssertionResult[] {
  return assertions.map((a) => evaluate(result, a));
}

function evaluate(result: EvalResult, assertion: EvalAssertion): AssertionResult {
  switch (assertion.type) {
    case "response_contains":
      return {
        assertion,
        passed: result.response.toLowerCase().includes(assertion.text.toLowerCase()),
        detail: assertion.text,
      };

    case "response_not_contains":
      return {
        assertion,
        passed: !result.response.toLowerCase().includes(assertion.text.toLowerCase()),
        detail: assertion.text,
      };

    case "response_contains_any":
      return {
        assertion,
        passed: assertion.texts.some((t) =>
          result.response.toLowerCase().includes(t.toLowerCase()),
        ),
        detail: assertion.texts.join(", "),
      };

    case "valid_json": {
      try {
        if (result.parsedJson !== undefined) return { assertion, passed: true };
        JSON.parse(result.response);
        return { assertion, passed: true };
      } catch {
        return { assertion, passed: false, detail: "Response is not valid JSON" };
      }
    }

    case "json_field_exists": {
      const obj = result.parsedJson as Record<string, unknown> | undefined;
      return {
        assertion,
        passed: obj !== undefined && assertion.field in obj,
        detail: assertion.field,
      };
    }

    case "tool_called": {
      const count = result.toolCalls.filter((t) => t.name === assertion.name).length;
      const min = assertion.minCount ?? 1;
      return {
        assertion,
        passed: count >= min,
        detail: `${assertion.name}: ${count} calls (need >= ${min})`,
      };
    }

    case "tool_not_called": {
      const found = result.toolCalls.some((t) => t.name === assertion.name);
      return { assertion, passed: !found, detail: assertion.name };
    }

    case "score_above": {
      const obj = result.parsedJson as Record<string, number> | undefined;
      const val = obj?.[assertion.field];
      return {
        assertion,
        passed: val !== undefined && val > assertion.threshold,
        detail: `${assertion.field}: ${val} (need > ${assertion.threshold})`,
      };
    }

    case "score_below": {
      const obj = result.parsedJson as Record<string, number> | undefined;
      const val = obj?.[assertion.field];
      return {
        assertion,
        passed: val !== undefined && val < assertion.threshold,
        detail: `${assertion.field}: ${val} (need < ${assertion.threshold})`,
      };
    }

    case "no_markdown":
      return {
        assertion,
        passed: !/[*#`_~\[\]]/.test(result.response),
        detail: "Response contains markdown characters",
      };

    case "rejected": {
      const obj = result.parsedJson as Record<string, unknown> | undefined;
      const reason = obj?.reason as string | undefined;
      return {
        assertion,
        passed:
          obj?.rejected === true &&
          reason !== undefined &&
          reason.toLowerCase().includes(assertion.reason_contains.toLowerCase()),
        detail: `reason: ${reason}`,
      };
    }

    case "custom":
      return {
        assertion,
        passed: assertion.fn(result),
        detail: assertion.name,
      };
  }
}
```

### Scenario runner

Create `evals/run.ts`:

```typescript
import { SpyClaude } from "./spy-claude";
import { runAssertions } from "./assertions";
import type { EvalScenario, EvalResult, AssertionResult } from "./assertions";

interface RunConfig {
  scenarios: EvalScenario[];
  createRealClaude: () => ClaudeClient;
  buildSystemPrompt: () => string;
  promptVersion: string;
  fixedDate?: string;
}

export async function runEvals(config: RunConfig): Promise<void> {
  const { scenarios, createRealClaude, buildSystemPrompt, promptVersion, fixedDate } = config;

  console.log(`\nRunning ${scenarios.length} eval scenarios (prompt ${promptVersion})\n`);

  let totalPassed = 0;
  let totalFailed = 0;

  for (const scenario of scenarios) {
    const runs = scenario.runs ?? 1;
    const results: EvalResult[] = [];

    for (let r = 0; r < runs; r++) {
      const spy = new SpyClaude(createRealClaude());
      const start = Date.now();

      const response = await spy.createMessage({
        model: "claude-sonnet-4-20250514",
        max_tokens: 2048,
        system: buildSystemPrompt(),
        messages: buildMessages(scenario),
      });

      const responseText = response.content
        .filter((b) => b.type === "text")
        .map((b) => b.text)
        .join("");

      let parsedJson: unknown;
      try {
        parsedJson = JSON.parse(responseText);
      } catch {
        parsedJson = undefined;
      }

      results.push({
        scenario: scenario.name,
        response: responseText,
        toolCalls: spy.toolCalls,
        parsedJson,
        durationMs: Date.now() - start,
        promptVersion,
      });
    }

    const assertionResults = runAssertions(results[0], scenario.assertions);
    const passed = assertionResults.filter((a) => a.passed);
    const failed = assertionResults.filter((a) => !a.passed);
    totalPassed += passed.length;
    totalFailed += failed.length;

    const status = failed.length === 0 ? "PASS" : "FAIL";
    console.log(`  ${status} ${scenario.name} (${results[0].durationMs}ms)`);

    for (const f of failed) {
      console.log(`       FAIL: ${f.assertion.type} ${f.detail ?? ""}`);
    }
  }

  console.log(`\n${totalPassed} passed, ${totalFailed} failed\n`);

  if (totalFailed > 0) {
    process.exit(1);
  }
}

function buildMessages(scenario: EvalScenario) {
  if (scenario.imageBase64) {
    return [
      {
        role: "user" as const,
        content: [
          {
            type: "image" as const,
            source: {
              type: "base64" as const,
              media_type: "image/jpeg" as const,
              data: scenario.imageBase64,
            },
          },
          ...(scenario.userMessage
            ? [{ type: "text" as const, text: scenario.userMessage }]
            : []),
        ],
      },
    ];
  }
  return [{ role: "user" as const, content: scenario.userMessage }];
}
```

### Scenario format

Create `evals/scenarios.example.ts`:

```typescript
import type { EvalScenario } from "./assertions";

// Adapt these to your domain
export const scenarios: EvalScenario[] = [
  {
    name: "identity",
    description: "Agent uses its configured name, not Claude",
    userMessage: "What is your name?",
    assertions: [
      { type: "response_not_contains", text: "Claude" },
      { type: "no_markdown" },
    ],
  },
  {
    name: "tool-honesty",
    description: "Agent calls the tool before claiming it acted",
    userMessage: "Save a note that says 'buy milk'",
    assertions: [
      { type: "tool_called", name: "write_memory", minCount: 1 },
      { type: "response_contains_any", texts: ["saved", "noted", "done"] },
    ],
  },
  {
    name: "structured-output",
    description: "Agent returns valid JSON matching the schema",
    userMessage: "Rate this image",
    imageBase64: "...", // load from fixtures
    assertions: [
      { type: "valid_json" },
      { type: "json_field_exists", field: "stance" },
      { type: "json_field_exists", field: "roast" },
    ],
  },
  {
    name: "scoring-consistency",
    description: "Same input scores within 5 points across 3 runs",
    userMessage: "Rate this image",
    imageBase64: "...",
    runs: 3,
    assertions: [
      { type: "custom", name: "variance < 5", fn: (ctx) => true }, // implement with multi-run
    ],
  },
];
```

### Prompt versioning

Track prompt versions in your code:

```typescript
// lib/ai/prompt.ts
export const PROMPT_VERSION = "v1";

export function buildSystemPrompt(): string {
  return [
    "You are a car rating assistant.",
    "Score each car photo on 5 categories (1-10 each).",
    // ... rubric details
  ].join("\n");
}
```

Store `PROMPT_VERSION` with every production output:

```sql
-- In your rides/ratings/outputs table
prompt_version text not null
```

Compare eval results across versions:

```bash
npm run eval -- --report  # Groups pass rates by PROMPT_VERSION
```

## Install

1. Copy `evals/` into your project
2. Write scenarios for your domain
3. Add to package.json:

```json
{
  "eval": "npx tsx evals/run.ts"
}
```

4. Set `ANTHROPIC_API_KEY` in `.env` (evals call the real API)
5. Run: `npm run eval`

Each run costs ~$0.01-0.05 depending on scenario count and image usage.

## Why this over a testing framework

See [ADR 002: Eval Harness with SpyClaude](../../docs/adr/002-eval-harness-spy-claude.md).

Integration tests (with StubClaude) verify your code handles Claude's responses correctly. Evals verify Claude gives the right responses to your prompts. Both are needed. They test different failure modes.
