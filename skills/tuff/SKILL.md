---
name: tuff
description: >-
  Build resumable, crash-recoverable data pipelines with tuff-lil-unit.
  Use when: building multi-step AI pipelines, fan-out patterns, crash recovery,
  resume from crash, structured DB output, batch processing, concurrent API calls,
  or progress tracking.
  Triggers: "tuff pipeline", "durable pipeline", "resumable pipeline", "LLM orchestration",
  "crash-recoverable workflow", "step memoization", "fan-out LLM calls".
---

# tuff

tuff-lil-unit is a TypeScript library for building resumable data pipelines. Every step's result is cached to SQLite — if the process crashes or you re-run, it resumes from the first uncached step. Built-in concurrency control, retry, and batch DB persistence.

`npm i tuff-lil-unit`

## When to reach for tuff

- Multi-step workflow needing crash recovery / resume
- Fan-out: many parallel calls to LLMs, APIs, CLI tools
- Where a queryable SQLite database of results is useful
- Mix of execution modes: LLM APIs, Claude Code agents, shell commands, plain functions

## Pipeline design interview

Before writing code, work through these with the user. The answers directly shape implementation.

### 1. What are you building?

- What's the input? (URLs, documents, database rows, user list…)
- What's the desired output? (JSON report, database, transformed data…)
- What does "done" look like?

### 2. What are the phases?

Map the pipeline to sequential phases. Each phase:
- Has a clear input → output contract
- Is re-runnable independently
- Named verb-noun: query, fetch, extract, analyse, distill

### 3. Which providers and models?

Ask the user which provider(s) they want to use. If they're unsure, present the options neutrally:

- **`ctx.model.anthropic`** — Anthropic API. Requires `ANTHROPIC_API_KEY` in `.env`.
- **`ctx.model.openai`** — OpenAI API. Requires `OPENAI_API_KEY` in `.env`.
- **`ctx.agent.claudeCode`** — Headless Claude Code subprocess. No API key — uses the user's CLI subscription quota. Risk: can burn through a usage window fast. Play at your own risk.
- **Plain `ctx.step`** — any async work, no LLM provider needed.

Providers can be mixed per phase. Match model size to task — cheapest that can handle the job.

Ask the user if they want token budget controls (pipeline-level or per-step limits). Useful for cost management, but adds implementation complexity — token windows often need tuning through trial and error. See `references/api-reference.md` for budget API details.

**Before writing pipeline code**, verify `.env` has keys for chosen providers, then fetch available models. Do not hardcode model names from memory:

```bash
# Anthropic
curl -s https://api.anthropic.com/v1/models \
  -H "x-api-key: $(grep ANTHROPIC_API_KEY .env | cut -d= -f2)" \
  -H "anthropic-version: 2023-06-01" | jq '.data[].id'

# OpenAI
curl -s https://api.openai.com/v1/models \
  -H "Authorization: Bearer $(grep OPENAI_API_KEY .env | cut -d= -f2)" \
  | jq '.data[].id'

# Claude Code — model aliases: sonnet, opus, haiku
```

### 4. What needs to be queryable?

- Which results land in SQLite tables?
- What queries will you run against completed data?
- Design tables around those queries

### How answers shape implementation

| Answer | Implementation decision |
|---|---|
| Result needs querying/joining | → `ctx.upsert` into a domain table |
| Result is intermediate / only feeds next phase | → plain `ctx.step` returning JSON |
| Phase has many items | → `ctx.upsert` (gets concurrency + retry per item) |
| Phase is a single expensive call | → `ctx.step` |
| Multiple consumers query same data differently | → one table per entity, not per phase |
| Pipeline may run for different segments/orgs | → partition key pattern (see `references/architecture.md`) |
| No LLM needed (HTTP fetch, parsing, transforms) | → plain `ctx.step` |

## Implementation guidance

- **API keys** — `ctx.model.anthropic` requires `ANTHROPIC_API_KEY` and `ctx.model.openai` requires `OPENAI_API_KEY` in `.env`. Add `--env-file=.env` to the node command or use `dotenv`.
- **Progress script** — always create one for pipelines with significant fan-out. Users will want it even if they don't ask. See `references/architecture.md` for the query pattern.
- **Cache invalidation** — warn users that cached step results persist even when step logic changes. They need `force: true` or stage-level `force` to invalidate.
- **Concurrency values** — Library default is 5 (conservative for LLM rate limits). Tell the user what you've set and why, and invite adjustment. LLM APIs can typically handle 10–20 if the user has rate-limit headroom. Claude Code agents (`ctx.agent.claudeCode`) tolerate high concurrency (60+ tested). Shell commands and external services vary — flag these specifically.
- **Table design** — one table per entity (sources, analyses, reports), not per phase. Phases write to tables; tables outlive phases.
- **Use `.mjs` for pipelines** — runs directly with `node`, no build step or `tsx` dependency. The library ships `.d.ts` files so editors still provide full type intelligence. Only use `.ts` if the project already has a TS toolchain.

## Core API

```ts
import { tuff } from 'tuff-lil-unit';

await tuff('run-id', {
  stateDir: './work',
  concurrency: 5,
  setup,                     // CREATE TABLE SQL strings for domain tables
  onProgress: ({ stage, completed, total, usage }) => {},
}, async (ctx) => { /* pipeline body */ });
```

### ctx.step(id, fn, opts?)

Resumable step — cached to SQLite, skipped on resume.

```ts
const data = await ctx.step('fetch-data', () => fetchAll());
```

`{ force: true }` invalidates cached result. Use when changing what a step returns — the old cached value persists otherwise.

### ctx.stage(label, opts?)

Set pipeline phase. Resets counters, optionally override concurrency.

```ts
ctx.stage('extract', { concurrency: 10 });
```

`{ force: true }` invalidates all steps in stage.

### ctx.upsert(items, opts)

Batch step+upsert: each item wrapped in step() for memoization/retry/concurrency, then upserted to SQLite.

```ts
await ctx.upsert(urls, {
  table: sources,
  key: (url) => `fetch-${url.id}`,
  run: (url) => fetchAndParse(url),
  skip: (r) => r.status === 'error',
  map: (r) => ({ ...r, fetchedAt: new Date().toISOString() }),
  update: ['content', 'status'],
});
```

Step IDs in loops must include a unique identifier — `fetch-${url.id}` not `'fetch'`.

### Execution modes

Any async work runs inside `ctx.step()` — shell commands, HTTP fetches, file processing, anything. Three built-in providers add LLM-specific conveniences (structured output, token tracking, retry).

**ctx.model.anthropic** / **ctx.model.openai** — Direct API calls returning `ProviderResult<T>`. Destructure `{ output }` to get the value. Wrap in `ctx.step()` for memoization and per-step usage tracking.

```ts
// Plain text
const { output: text } = await ctx.step('summarize', () =>
  ctx.model.anthropic('claude-sonnet-4-5', prompt)
);

// Structured output — pass a Zod schema
const { output: data } = await ctx.step('extract', () =>
  ctx.model.openai<Summary>('gpt-5-mini', prompt, { schema: SummarySchema })
);

// Additional options
const { output } = await ctx.step('analyse', () =>
  ctx.model.anthropic('claude-sonnet-4-5', prompt, {
    temperature: 0.2,
    system: 'You are a data analyst.',
  })
);

// Inside ctx.upsert — unwrap output; upsert stores the return value as the row
await ctx.upsert(items, {
  key: (item) => `analyse-${item.id}`,
  run: async (item) => {
    const { output } = await ctx.model.anthropic('claude-sonnet-4-5', buildPrompt(item));
    return output;
  },
});
```

**ctx.agent.claudeCode** (experimental) — Spawns headless `claude -p` subprocess.

```ts
const { output } = await ctx.step('refactor', () =>
  ctx.agent.claudeCode('claude-sonnet-4-5', prompt)
);
```

> **Subscription warning:** Consumes the user's Claude Code CLI subscription quota. Fan-out patterns can burn through a usage window quickly. Confirm with the user before using this provider, especially at scale.

### ctx.db

Direct better-sqlite3 access for custom queries.

## Error handling

3 retries, exponential backoff (1s–60s, factor 2). Failures logged to `tuff_step_failures`; cleared on successful re-execution.

| Type | Retriable | Triggers |
|---|---|---|
| `rate_limit` | yes | 429, "rate limit" in message |
| `transient` | yes | 5xx, timeout, AbortError |
| `token_limit` | no | "token limit", "context limit" |
| `budget_exceeded` | no | step or pipeline budget exceeded |
| `permanent` | no | everything else |

## Key constraints

- **Serializable returns only** — step results stored as JSON. No Dates, Buffers, class instances. Use ISO strings, plain objects.
- **Unique step IDs** — within a stage, IDs must be unique. In loops: `` `fetch-${url}` ``.
- **No nested steps** — `ctx.step()` throws if called inside another step or upsert callback. Provider calls inside a step are fine; nesting `ctx.step()` inside `ctx.step()` is not.
- **Composite PK upsert** — pass all primary key columns in the row object so INSERT … ON CONFLICT matches correctly.

## Reference files

| Resource | When to read |
|---|---|
| `references/architecture.md` | Read before writing any pipeline. Required for file structure, phase function patterns, composition with partition keys, fan-out patterns, and progress script examples. |
| `references/api-reference.md` | Read when the user asks about token budgets, spend control, per-step limits, or needs to query tuff's internal state tables directly. |
