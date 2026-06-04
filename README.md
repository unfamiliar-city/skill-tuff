# tuff

![tuff-lil-unit](https://raw.githubusercontent.com/unfamiliar-city/tuff-lil-unit/main/assets/tuff.jpg)

A Claude Code skill for building resumable AI pipelines with [tuff-lil-unit](https://github.com/unfamiliar-city/tuff-lil-unit).

Tell Claude what you want to build. Tuff handles the rest — crash recovery, concurrency, retry, and a queryable SQLite database of everything that happened.

## An example pipeline

| Phase | What it does | Model | Calls | Concurrency |
|-------|-------------|-------|-------|-------------|
| Collect | Fetch 200 source URLs | *none* | 200 | 50 |
| Extract | Pull structured data from each page | GPT-5-nano | 200 | 10 |
| Classify | Score and categorise each result | GPT-5-mini | 200 | 5 |
| Distil | Aggregate into final report | *none* | 1 | 1 |

~400 LLM calls, 200 HTTP fetches, 4 phases. One `/tuff` invocation.

## Install

Via the [unfamiliar-city marketplace](https://github.com/unfamiliar-city/marketplace):

```
/plugin marketplace add unfamiliar-city/marketplace
/reload-plugins
/plugin install tuff
```

Or add the skill directly to your project:

```
mkdir -p .claude/skills
git submodule add https://github.com/unfamiliar-city/skill-tuff .claude/skills/tuff
```

## Use

```
Claude Code
Opus 4.6 · Claude API
~/Projects/myproject

 > /tuff build a pipeline that scrapes 200 URLs, extracts key data
   with GPT-5-nano, and generates a summary report
```

Claude designs the phases, sets up the file structure, installs `tuff-lil-unit`, and writes the code. If it crashes, re-run — it resumes from where it left off.

## What the skill knows

- Pipeline design — phases, table schemas, concurrency tuning
- The full tuff API — `ctx.step`, `ctx.upsert`, `ctx.model.*`, `ctx.agent.*`
- Three execution modes — LLM APIs (Anthropic/OpenAI), Claude Code headless (play at your own risk), any async function
- Architecture patterns — file structure, fan-out, composition, progress tracking

## Library

The skill installs [tuff-lil-unit](https://github.com/unfamiliar-city/tuff-lil-unit) (`npm i tuff-lil-unit`) — a TypeScript toolkit that implements the step-function pattern with SQLite persistence. See the library repo for API docs and code examples.

## Status

Alpha. See [tuff-lil-unit](https://github.com/unfamiliar-city/tuff-lil-unit#status) for details.

## License

MIT
