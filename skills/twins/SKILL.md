---
name: twins
description: Use when setting up tests for a project that calls external services (Supabase, Claude API, Redis). Creates stateful in-memory clones (digital twins) that enforce real API contracts without network calls.
---

# Digital Twins

Stateful in-memory behavioral clones of external APIs. They implement the same interface as production clients, enforce real API contracts, and maintain state across a test.

## When to use

- Setting up a test suite for a project that calls Supabase, Claude, or Redis
- Replacing jest.mock() or vi.mock() with something that catches real bugs
- Adding integration tests that run without network access
- Making tests fast enough for pre-push hooks

## How it works

1. Define dependency interfaces for your external services
2. Production code receives dependencies as parameters (never imports concrete clients)
3. Tests inject twins that implement the same interfaces
4. Same code path, no network, deterministic results

## Supabase Twin

Create `twins/supabase.ts`:

```typescript
import type { SupabaseClient } from "@/lib/deps";

interface StoredRow {
  [key: string]: unknown;
}

export class SupabaseTwin implements SupabaseClient {
  private tables: Map<string, StoredRow[]> = new Map();
  private currentUserId: string | null = null;
  private buckets: Map<string, Map<string, Buffer>> = new Map();

  // Auth
  setUser(userId: string): void {
    this.currentUserId = userId;
  }

  clearUser(): void {
    this.currentUserId = null;
  }

  getUserId(): string | null {
    return this.currentUserId;
  }

  // Query builder
  from(table: string) {
    const rows = this.tables.get(table) ?? [];
    return new QueryBuilder(rows, table, this.tables, this.currentUserId);
  }

  // Storage
  storage = {
    from: (bucket: string) => {
      if (!this.buckets.has(bucket)) {
        this.buckets.set(bucket, new Map());
      }
      const store = this.buckets.get(bucket)!;
      return {
        upload: async (path: string, data: Buffer) => {
          store.set(path, data);
          return { data: { path }, error: null };
        },
        getPublicUrl: (path: string) => ({
          data: { publicUrl: `https://fake-storage.test/${bucket}/${path}` },
        }),
        download: async (path: string) => {
          const file = store.get(path);
          if (!file) return { data: null, error: { message: "Not found" } };
          return { data: file, error: null };
        },
      };
    },
  };

  reset(): void {
    this.tables.clear();
    this.currentUserId = null;
    this.buckets.clear();
  }

  // Seed data for tests
  seed(table: string, rows: StoredRow[]): void {
    this.tables.set(table, [...rows]);
  }
}

class QueryBuilder {
  private filters: Array<{ column: string; value: unknown }> = [];
  private orderColumn: string | null = null;
  private orderAsc = true;
  private limitCount: number | null = null;
  private selectColumns: string | null = null;
  private isSingle = false;

  constructor(
    private rows: StoredRow[],
    private table: string,
    private tables: Map<string, StoredRow[]>,
    private userId: string | null,
  ) {}

  select(columns?: string) {
    this.selectColumns = columns ?? "*";
    return this;
  }

  eq(column: string, value: unknown) {
    this.filters.push({ column, value });
    return this;
  }

  order(column: string, { ascending = true } = {}) {
    this.orderColumn = column;
    this.orderAsc = ascending;
    return this;
  }

  limit(count: number) {
    this.limitCount = count;
    return this;
  }

  single() {
    this.isSingle = true;
    return this;
  }

  async insert(row: StoredRow | StoredRow[]) {
    const newRows = Array.isArray(row) ? row : [row];
    const existing = this.tables.get(this.table) ?? [];
    const withIds = newRows.map((r) => ({
      id: r.id ?? crypto.randomUUID(),
      created_at: new Date().toISOString(),
      ...r,
    }));
    this.tables.set(this.table, [...existing, ...withIds]);
    return { data: withIds, error: null };
  }

  async update(values: Partial<StoredRow>) {
    const existing = this.tables.get(this.table) ?? [];
    let updated: StoredRow[] = [];
    const result = existing.map((row) => {
      const matches = this.filters.every((f) => row[f.column] === f.value);
      if (matches) {
        const merged = { ...row, ...values };
        updated.push(merged);
        return merged;
      }
      return row;
    });
    this.tables.set(this.table, result);
    return { data: updated, error: null };
  }

  async delete() {
    const existing = this.tables.get(this.table) ?? [];
    const remaining = existing.filter(
      (row) => !this.filters.every((f) => row[f.column] === f.value),
    );
    this.tables.set(this.table, remaining);
    return { data: null, error: null };
  }

  async then(resolve: (result: { data: unknown; error: null }) => void) {
    let result = [...this.rows];
    for (const f of this.filters) {
      result = result.filter((row) => row[f.column] === f.value);
    }
    if (this.orderColumn) {
      const col = this.orderColumn;
      const dir = this.orderAsc ? 1 : -1;
      result.sort((a, b) => {
        if (a[col]! < b[col]!) return -1 * dir;
        if (a[col]! > b[col]!) return 1 * dir;
        return 0;
      });
    }
    if (this.limitCount !== null) {
      result = result.slice(0, this.limitCount);
    }
    const data = this.isSingle ? result[0] ?? null : result;
    resolve({ data, error: null });
  }
}
```

## Claude Twin (StubClaude)

Create `twins/claude.ts`:

```typescript
import type { ClaudeClient, ClaudeMessage, CreateMessageParams } from "@/lib/deps";

interface QueuedResponse {
  type: "text" | "tool_use";
  text?: string;
  toolName?: string;
  toolInput?: Record<string, unknown>;
}

export class StubClaude implements ClaudeClient {
  private queue: ClaudeMessage[] = [];
  public calls: CreateMessageParams[] = [];

  constructor(responses: QueuedResponse[] = []) {
    for (const r of responses) {
      this.queue.push(this.buildMessage(r));
    }
  }

  async createMessage(params: CreateMessageParams): Promise<ClaudeMessage> {
    this.calls.push(params);
    const next = this.queue.shift();
    if (!next) {
      return this.buildMessage({ type: "text", text: "No response queued." });
    }
    return next;
  }

  enqueue(response: QueuedResponse): void {
    this.queue.push(this.buildMessage(response));
  }

  private buildMessage(r: QueuedResponse): ClaudeMessage {
    if (r.type === "tool_use") {
      return {
        id: crypto.randomUUID(),
        type: "message",
        role: "assistant",
        stop_reason: "tool_use",
        content: [
          { type: "tool_use", id: crypto.randomUUID(), name: r.toolName!, input: r.toolInput ?? {} },
        ],
        usage: { input_tokens: 100, output_tokens: 50 },
      };
    }
    return {
      id: crypto.randomUUID(),
      type: "message",
      role: "assistant",
      stop_reason: "end_turn",
      content: [{ type: "text", text: r.text ?? "" }],
      usage: { input_tokens: 100, output_tokens: 50 },
    };
  }
}
```

## Redis Twin

Create `twins/redis.ts`:

```typescript
import type { RedisClient } from "@/lib/deps";

interface Entry {
  value: string | string[];
  type: "string" | "list";
  expiresAt: number | null;
}

export class RedisTwin implements RedisClient {
  private store: Map<string, Entry> = new Map();
  private time = Date.now();

  tick(ms: number): void {
    this.time += ms;
  }

  private now(): number {
    return this.time;
  }

  private isExpired(entry: Entry): boolean {
    return entry.expiresAt !== null && this.now() >= entry.expiresAt;
  }

  private getEntry(key: string): Entry | undefined {
    const entry = this.store.get(key);
    if (!entry) return undefined;
    if (this.isExpired(entry)) {
      this.store.delete(key);
      return undefined;
    }
    return entry;
  }

  async get(key: string): Promise<string | null> {
    const entry = this.getEntry(key);
    if (!entry || entry.type !== "string") return null;
    return entry.value as string;
  }

  async set(key: string, value: string): Promise<void> {
    this.store.set(key, { value, type: "string", expiresAt: null });
  }

  async setex(key: string, seconds: number, value: string): Promise<void> {
    this.store.set(key, {
      value,
      type: "string",
      expiresAt: this.now() + seconds * 1000,
    });
  }

  async del(key: string): Promise<number> {
    return this.store.delete(key) ? 1 : 0;
  }

  async exists(key: string): Promise<number> {
    return this.getEntry(key) ? 1 : 0;
  }

  async keys(pattern: string): Promise<string[]> {
    const regex = new RegExp("^" + pattern.replace(/\*/g, ".*") + "$");
    const result: string[] = [];
    for (const [key] of this.store) {
      if (this.getEntry(key) && regex.test(key)) {
        result.push(key);
      }
    }
    return result;
  }

  async lpush(key: string, ...values: string[]): Promise<number> {
    const entry = this.getEntry(key);
    if (entry && entry.type !== "list") {
      throw new Error("WRONGTYPE Operation against a key holding the wrong kind of value");
    }
    const list = entry ? (entry.value as string[]) : [];
    list.unshift(...values);
    this.store.set(key, { value: list, type: "list", expiresAt: entry?.expiresAt ?? null });
    return list.length;
  }

  async rpush(key: string, ...values: string[]): Promise<number> {
    const entry = this.getEntry(key);
    if (entry && entry.type !== "list") {
      throw new Error("WRONGTYPE Operation against a key holding the wrong kind of value");
    }
    const list = entry ? (entry.value as string[]) : [];
    list.push(...values);
    this.store.set(key, { value: list, type: "list", expiresAt: entry?.expiresAt ?? null });
    return list.length;
  }

  async lrange(key: string, start: number, stop: number): Promise<string[]> {
    const entry = this.getEntry(key);
    if (!entry || entry.type !== "list") return [];
    const list = entry.value as string[];
    const end = stop === -1 ? list.length : stop + 1;
    return list.slice(start, end);
  }

  async expire(key: string, seconds: number): Promise<number> {
    const entry = this.getEntry(key);
    if (!entry) return 0;
    entry.expiresAt = this.now() + seconds * 1000;
    return 1;
  }

  async ttl(key: string): Promise<number> {
    const entry = this.getEntry(key);
    if (!entry) return -2;
    if (entry.expiresAt === null) return -1;
    return Math.ceil((entry.expiresAt - this.now()) / 1000);
  }

  async incr(key: string): Promise<number> {
    const entry = this.getEntry(key);
    const current = entry ? parseInt(entry.value as string, 10) : 0;
    if (isNaN(current)) {
      throw new Error("ERR value is not an integer or out of range");
    }
    const next = current + 1;
    this.store.set(key, { value: String(next), type: "string", expiresAt: entry?.expiresAt ?? null });
    return next;
  }

  reset(): void {
    this.store.clear();
    this.time = Date.now();
  }
}
```

## Dependency interfaces pattern

Create `lib/deps.ts` in your project:

```typescript
export interface ClaudeClient {
  createMessage(params: CreateMessageParams): Promise<ClaudeMessage>;
}

export interface SupabaseClient {
  from(table: string): QueryBuilder;
  storage: StorageClient;
}

export interface RedisClient {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
  setex(key: string, seconds: number, value: string): Promise<void>;
  del(key: string): Promise<number>;
  exists(key: string): Promise<number>;
  keys(pattern: string): Promise<string[]>;
  lpush(key: string, ...values: string[]): Promise<number>;
  rpush(key: string, ...values: string[]): Promise<number>;
  lrange(key: string, start: number, stop: number): Promise<string[]>;
  expire(key: string, seconds: number): Promise<number>;
  ttl(key: string): Promise<number>;
  incr(key: string): Promise<number>;
}

export interface Clock {
  now(): Date;
}

export interface Deps {
  claude: ClaudeClient;
  supabase: SupabaseClient;
  clock: Clock;
}
```

## Test example

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { StubClaude } from "@/twins/claude";
import { SupabaseTwin } from "@/twins/supabase";
import { rateRide } from "@/lib/ai/rate-ride";

describe("rateRide", () => {
  let claude: StubClaude;
  let supabase: SupabaseTwin;

  beforeEach(() => {
    claude = new StubClaude([
      {
        type: "text",
        text: JSON.stringify({
          car_make: "Nissan",
          car_model: "Silvia S15",
          stance: 9,
          style: 8,
          wheels: 9,
          power: 7,
          sendit: 9,
          roast: "Spec R with those TE37s? Chef's kiss, bro.",
        }),
      },
    ]);
    supabase = new SupabaseTwin();
  });

  it("parses Claude response into ride scores", async () => {
    const result = await rateRide(
      { claude, supabase, clock: { now: () => new Date("2026-03-10") } },
      { imageBase64: "abc123", userId: "user-1" },
    );
    expect(result.total_score).toBe(42);
    expect(result.car_make).toBe("Nissan");
    expect(result.roast).toContain("TE37");
  });

  it("sends image to Claude as base64", async () => {
    await rateRide(
      { claude, supabase, clock: { now: () => new Date("2026-03-10") } },
      { imageBase64: "abc123", userId: "user-1" },
    );
    expect(claude.calls).toHaveLength(1);
    const content = claude.calls[0].messages[0].content;
    expect(content).toEqual(
      expect.arrayContaining([expect.objectContaining({ type: "image" })]),
    );
  });
});
```

## Why twins over mocks

See [ADR 001: Digital Twins Over Mocks](../../docs/adr/001-digital-twins-over-mocks.md).

Short version: jest.mock() replaces the interface. Twins implement it. A mock won't throw WRONGTYPE when you LPUSH to a string key. A twin will, the same way Redis will in production.
