# Architecture patterns

## Reference file structure

Phases are functions, not scripts. Tuff owns concurrency, retry, and resume — you write the phase logic. Keep logic functions pure and DB-free so they're importable across projects.

```
scripts/
  pipeline.mjs          # tuff runner entry point
  phases/
    query.mjs           # ctx.stage() + ctx.upsert() per phase
    fetch.mjs
    extract.mjs
  logic/
    query-icp.mjs       # pure functions — no DB, testable, importable
    fetch-urls.mjs
    extract-sources.mjs
  db/
    schema.mjs           # CREATE TABLE SQL strings
    readers.mjs          # parse JSON columns, join helpers
    progress.mjs         # completion counts per phase
```

Design question: could another skill or project import your logic functions?

## Pipeline runner

```js
const PHASES = ['query', 'fetch', 'extract'];
const PHASE_FNS = { query: queryPhase, fetch: fetchPhase, extract: extractPhase };

await tuff(runId, { stateDir: workDir, concurrency: 5, setup }, async (ctx) => {
  for (const p of targetPhase ? [targetPhase] : PHASES) {
    await PHASE_FNS[p](ctx, config);
  }
});
```

Per-phase scripts let you re-run individual phases:

```json
{
  "pipeline": "node scripts/pipeline.mjs",
  "pipeline:query": "node scripts/pipeline.mjs query",
  "pipeline:fetch": "node scripts/pipeline.mjs fetch",
  "pipeline:progress": "node scripts/db/progress.mjs"
}
```

## Phase function

```js
// phases/fetch.mjs
export async function fetchPhase(ctx, config) {
  ctx.stage('fetch', { concurrency: 20 });
  const urls = collectUrls(config);
  await ctx.upsert(urls, {
    table: sources,
    key: (u) => `fetch-${u.url}`,
    run: (u) => fetchAndParse(u.url),
  });
}
```

## Logic functions — composable, no DB

```js
// logic/fetch-urls.mjs — importable by other projects
export async function fetchAndParse(url) { ... }
export function extractContent(html) { ... }
export function collectUrls(queryResults) { ... }
```

## Typing domain table columns

When a domain table column stores a JSON blob, define the shape once in Zod. Apply at both the write boundary (LLM structured output) and the read boundary (DB reader):

```js
// logic/schemas/core.mjs — single source of truth
export const MentionSchema = z.array(z.object({
  name: z.string(),
  mention_type: z.enum(['endorsed', 'listed', 'qualified']),
  salience: z.number(),
}));

// db/readers.mjs — reader is the type boundary
import { MentionSchema } from '../logic/schemas/core.mjs';

export function readMentions(row) {
  return MentionSchema.parse(JSON.parse(row.mentions));
}
```

The import in the reader *is* the documentation — anyone reading it sees which schema governs the blob. It can't drift because it breaks.

If the stored shape diverges from the LLM output shape (e.g. you enrich before storing), define a separate schema for the stored shape.

## Designing for composition

If a meta-pipeline might run your phases multiple times with different contexts (segments, organisations, regions) against a single DB, your phase functions need a partition key. Accept an optional `partition` in config and thread it through:

- **Table PKs** — so rows from different runs don't collide
- **Step IDs** — so tuff memoisation stays correct per-context

```js
// phases/fetch.mjs — composable
export async function fetchPhase(ctx, config) {
  const prefix = config.partition ? `${config.partition}-` : '';
  ctx.stage(`${prefix}fetch`, { concurrency: 20 });
  await ctx.upsert(urls, {
    table: sources,
    key: (u) => `${prefix}fetch-${u.url}`,
    run: (u) => fetchAndParse(u.url),
    map: (r) => ({ ...r, partition: config.partition ?? '_' }),
  });
}
```

`partition` is deliberately generic — the consumer decides what it represents.

## Fan-out patterns

```js
// Parallel — all must succeed
ctx.stage('classify');
const results = await Promise.all(
  docs.map((d) => ctx.step(`classify-${d.id}`, () => classify(d)))
);

// Parallel — tolerate failures
const settled = await Promise.allSettled(
  docs.map((d) => ctx.step(`enrich-${d.id}`, () => enrich(d)))
);
```

Always include a unique identifier in loop step IDs.

## Progress script

Query tuff's SQLite tables to report completion per stage:

```js
// scripts/db/progress.mjs
const stages = db.prepare(`
  SELECT stage, COUNT(*) as total,
    SUM(CASE WHEN output IS NOT NULL THEN 1 ELSE 0 END) as ok
  FROM tuff_steps WHERE run_id = ?
  GROUP BY stage
`).all(runId);
// → query: 24/24 · fetch: 18/24 (ok: 16, err: 2) · extract: 16/16
```

Add to package.json: `"pipeline:progress": "node scripts/db/progress.mjs"`

### Checking failures

```sql
SELECT step_id, error, created_at
FROM tuff_step_failures
WHERE run_id = ? ORDER BY created_at DESC LIMIT 20;
```

## Agent integration (Claude Code)

When orchestrating tuff from a Claude Code skill, use task dependencies for phase sequencing:

```
TaskCreate("Run query phase")    → npm run pipeline:query
TaskCreate("Run fetch phase")    → blockedBy: [query], npm run pipeline:fetch
TaskCreate("Run extract phase")  → blockedBy: [fetch], npm run pipeline:extract
```

Run phases as background Bash jobs. Poll with `npm run pipeline:progress` or check task status. Mark tasks done as phases complete.

This pattern is Claude Code specific — other agents will have their own task/job primitives.
