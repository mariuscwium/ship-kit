---
name: di-pattern
description: Use when setting up dependency injection for a TypeScript project. Provides interfaces for external services, lazy initialization for serverless, and testable API route patterns. No framework needed.
---

# Dependency Injection Pattern

TypeScript interfaces for your external services. Lazy initialization for serverless. Testable by default. Three files, no framework.

## When to use

- Starting a project that calls external APIs (Claude, Supabase, Redis, Stripe, etc.)
- Making existing code testable without rewriting it
- Setting up serverless functions that need lazy dependency init
- Replacing direct imports of SDKs with injectable interfaces

## The pattern

Three files:

### 1. Interfaces (`lib/deps.ts`)

Define what your code needs from each service. Only the methods you use.

```typescript
// Types for Claude API
export interface CreateMessageParams {
  model: string;
  max_tokens: number;
  system: string;
  messages: Array<{
    role: "user" | "assistant";
    content: string | ContentBlock[];
  }>;
  tools?: ToolDefinition[];
}

export interface ClaudeMessage {
  id: string;
  type: string;
  role: string;
  stop_reason: string;
  content: ContentBlock[];
  usage: { input_tokens: number; output_tokens: number };
}

export interface ContentBlock {
  type: "text" | "tool_use" | "image";
  text?: string;
  name?: string;
  id?: string;
  input?: Record<string, unknown>;
  source?: { type: string; media_type: string; data: string };
}

// Service interfaces
export interface ClaudeClient {
  createMessage(params: CreateMessageParams): Promise<ClaudeMessage>;
}

export interface DatabaseClient {
  from(table: string): QueryBuilder;
}

export interface StorageClient {
  upload(bucket: string, path: string, data: Buffer): Promise<{ url: string }>;
  download(bucket: string, path: string): Promise<Buffer>;
  getPublicUrl(bucket: string, path: string): string;
}

export interface Clock {
  now(): Date;
}

// Compose into a single Deps object
export interface Deps {
  claude: ClaudeClient;
  db: DatabaseClient;
  storage: StorageClient;
  clock: Clock;
}
```

Adapt these interfaces to your services. Only include methods you call. A smaller interface is easier to twin.

### 2. Production clients (`lib/clients.ts`)

Wrap real SDKs to match your interfaces.

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { createClient } from "@supabase/supabase-js";
import type { ClaudeClient, DatabaseClient, StorageClient } from "./deps";

export function createClaudeClient(apiKey: string): ClaudeClient {
  const client = new Anthropic({ apiKey });
  return {
    async createMessage(params) {
      return client.messages.create(params) as Promise<ClaudeMessage>;
    },
  };
}

export function createDatabaseClient(url: string, key: string): DatabaseClient {
  const supabase = createClient(url, key);
  return {
    from(table: string) {
      return supabase.from(table);
    },
  };
}

export function createStorageClient(url: string, key: string): StorageClient {
  const supabase = createClient(url, key);
  return {
    async upload(bucket, path, data) {
      const { error } = await supabase.storage.from(bucket).upload(path, data);
      if (error) throw error;
      const { data: urlData } = supabase.storage.from(bucket).getPublicUrl(path);
      return { url: urlData.publicUrl };
    },
    async download(bucket, path) {
      const { data, error } = await supabase.storage.from(bucket).download(path);
      if (error) throw error;
      return Buffer.from(await data.arrayBuffer());
    },
    getPublicUrl(bucket, path) {
      const { data } = supabase.storage.from(bucket).getPublicUrl(path);
      return data.publicUrl;
    },
  };
}
```

### 3. Lazy initialization (`lib/prod-deps.ts`)

Singleton cache. Created on first use. Avoids throwing during test imports.

```typescript
import type { Deps } from "./deps";
import { createClaudeClient, createDatabaseClient, createStorageClient } from "./clients";

let _deps: Deps | undefined;

export function getProdDeps(): Deps {
  if (_deps) return _deps;

  _deps = {
    claude: createClaudeClient(process.env.ANTHROPIC_API_KEY!),
    db: createDatabaseClient(process.env.SUPABASE_URL!, process.env.SUPABASE_ANON_KEY!),
    storage: createStorageClient(process.env.SUPABASE_URL!, process.env.SUPABASE_ANON_KEY!),
    clock: { now: () => new Date() },
  };

  return _deps;
}
```

## Using in API routes

```typescript
// app/api/rate/route.ts
import { getProdDeps } from "@/lib/prod-deps";
import { rateImage } from "@/lib/ai/rate-image";

export async function POST(req: Request) {
  const deps = getProdDeps();
  const body = await req.json();
  const result = await rateImage(deps, body.imageBase64);
  return Response.json(result);
}
```

## Using in application code

Your business logic receives `Deps` as a parameter. It never imports SDK clients directly.

```typescript
// lib/ai/rate-image.ts
import type { Deps } from "@/lib/deps";
import { PROMPT_VERSION, buildSystemPrompt } from "./prompt";

interface RatingResult {
  scores: Record<string, number>;
  review: string;
  promptVersion: string;
}

export async function rateImage(deps: Deps, imageBase64: string): Promise<RatingResult> {
  const response = await deps.claude.createMessage({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    system: buildSystemPrompt(),
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: { type: "base64", media_type: "image/jpeg", data: imageBase64 },
          },
        ],
      },
    ],
  });

  const text = response.content
    .filter((b) => b.type === "text")
    .map((b) => b.text!)
    .join("");

  const parsed = JSON.parse(text);

  return {
    scores: parsed,
    review: parsed.roast,
    promptVersion: PROMPT_VERSION,
  };
}
```

## Using in tests

```typescript
import { describe, it, expect } from "vitest";
import { StubClaude } from "@/twins/claude";
import { SupabaseTwin } from "@/twins/supabase";
import { rateImage } from "@/lib/ai/rate-image";

it("parses scores from Claude response", async () => {
  const claude = new StubClaude([
    { type: "text", text: '{"stance": 8, "style": 7, "roast": "Nice."}' },
  ]);

  const result = await rateImage(
    {
      claude,
      db: new SupabaseTwin(),
      storage: new SupabaseTwin().storage,
      clock: { now: () => new Date("2026-03-10") },
    },
    "base64imagedata",
  );

  expect(result.scores.stance).toBe(8);
  expect(claude.calls).toHaveLength(1);
});
```

## Why interfaces over mocks

`vi.mock("@anthropic-ai/sdk")` replaces the module at import time. Your test no longer calls the same code path as production. If you change how you use the SDK, the mock doesn't break. The bug ships.

Interfaces keep the contract explicit. If `ClaudeClient.createMessage` changes its signature, both the production client and the twin fail to compile. The bug is caught at build time.

## Install

1. Create `lib/deps.ts` with interfaces for your services
2. Create `lib/clients.ts` wrapping your real SDKs
3. Create `lib/prod-deps.ts` with lazy singleton initialization
4. Pass `deps` as the first parameter to your business logic functions
5. In tests, construct deps from twins

No packages to install. No decorators. No reflection. TypeScript interfaces are the DI framework.

See [ADR 003: Dependency Injection via Interfaces](../../docs/adr/003-dependency-injection-via-interfaces.md).
