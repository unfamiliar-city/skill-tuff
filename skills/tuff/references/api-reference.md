# API reference

## Token budgets

Optional — only use when the user specifically needs spend control.

### Pipeline-level budget

```ts
await tuff('run-id', {
  stateDir: './work',
  concurrency: 5,
  budget: { tokens: 500_000 },  // input + output + cache-creation
}, async (ctx) => { ... });
```

Budget is checked between steps, not mid-step. A single expensive call can overshoot.

### Per-step budget

```ts
const { output } = await ctx.step('summarize', () =>
  ctx.model.anthropic('claude-sonnet-4-5', prompt, {
    maxTokens: 4096,         // cap output tokens (API-enforced)
    maxInputTokens: 50_000,  // pre-call estimate (~chars/4), refuses if over
    onExceed: 'warn',        // 'throw' (default) | 'warn'
  })
);
```

- `maxInputTokens` — estimated before the call, throws `BudgetExceededError` if over (or warns with `onExceed: 'warn'`)
- `maxTokens` — passed to the API as max output tokens
- Post-call: actual usage checked against both limits
- `isExceeded()` uses `>` not `>=` — exactly at budget = not exceeded

## State tables

All progress is queryable SQLite. Schema auto-synced at startup.

| Table | PK | Key columns |
|---|---|---|
| `tuff_runs` | `id` | `config`, `created_at`, `updated_at` |
| `tuff_steps` | `(run_id, step_id)` | `output` (JSON), `usage_input`, `usage_output`, `duration_ms` |
| `tuff_step_failures` | `(run_id, step_id)` | `error`, `duration_ms` |

Failures cleared on successful re-execution.

## Standalone exports

Providers, errors, and table schemas are importable independently:

```ts
import { AbortError, BudgetExceededError } from 'tuff-lil-unit';
import { SCHEMA_SQL, syncSchema, autoStringify, estimateTokens } from 'tuff-lil-unit';
import type { TuffRun, TuffStep, TuffStepFailure } from 'tuff-lil-unit';
```
