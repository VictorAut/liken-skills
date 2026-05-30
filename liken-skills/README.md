# liken-skills

Agent skills for [**Liken**](https://github.com/VictorAut/liken) — a Python library for
near-deduplication, fuzzy matching, and record linkage over pandas, Polars, Modin, Dask,
Ray and PySpark DataFrames.


## The skills

| Skill | API tier | Teaches |
| --- | --- | --- |
| [`liken`](skills/liken/SKILL.md) | overview / router | What Liken is, the simplest exact dedupe, and which skill/API to reach for. |
| [`liken-dedupers`](skills/liken-dedupers/SKILL.md) | apply dedupers | Built-in dedupers (exact, fuzzy, tfidf, lsh, jaccard, cosine + predicates), `.apply()` with a single deduper or a dict collection, pandas affordances. |
| [`liken-pipelines`](skills/liken-pipelines/SKILL.md) | pipelines | `lk.pipeline()`/`lk.col()`, AND/OR/NOT semantics, preprocessor scoping, rule predication. |
| [`liken-custom-dedupers`](skills/liken-custom-dedupers/SKILL.md) | extension | Defining and registering custom dedupers with `@lk.custom.register`. |
| [`liken-record-linkage`](skills/liken-record-linkage/SKILL.md) | entity resolution | `.canonicalize()`, `.collect()`, `.canonicals()`, `.synthesize()`. |
| [`liken-backends-performance`](skills/liken-backends-performance/SKILL.md) | scaling | Backend selection/extras, `spark_session`, blocking keys, LSH, cost model. |

## Install

### Via the tessl.io registry (end users)

```shell
tessl install <your-tessl-workspace>/liken-skills@0.1.0
```

Installed skills land under `.tessl/tiles/<workspace>/liken-skills/` in your project and
are made available to your agent.

### Locally for Claude Code (development)

Copy the individual skill folders into a skills directory Claude Code reads:

```shell
# project-scoped
cp -r skills/* .claude/skills/

# or user-scoped
cp -r skills/* ~/.claude/skills/
```

Each `skills/<name>/` folder is a standalone [Agent Skill](https://agentskills.io/specification)
(a `SKILL.md` plus optional `references/`).

## Publishing workflow

```shell
tessl skill lint   ./skills/<name>     # validate structure/frontmatter
tessl skill review ./skills/<name>     # quality score — aim for >= 70%
tessl skill publish                    # publish the bundle (bumps version)
```

> ⚠️ The exact tessl manifest path/fields (`.tessl-plugin/plugin.json`, the `skills` glob)
> and CLI commands here are a best-effort starting point. Confirm against
> <https://docs.tessl.io> and prefer generating the canonical scaffold with the tessl CLI
> (`tessl skill new` / `tessl init`), then reconcile [.tessl-plugin/plugin.json](.tessl-plugin/plugin.json).

## Versioning

Single bundle, one semver track (starts at `0.1.0`; `1.0.0` reserved for the first
"complete" release):

- **patch** — wording fixes, snippet corrections
- **minor** — new skill or new section
- **major** — restructuring / breaking reorganisation

When Liken's public API changes, update the affected skills and bump accordingly.

## Links

- Library docs: <https://victoraut.github.io/liken/>
- Repository: <https://github.com/VictorAut/liken>
- Agent Skills spec: <https://agentskills.io/specification>
