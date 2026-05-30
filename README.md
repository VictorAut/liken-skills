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


```shell
tessl install liken/liken-skills@0.1.0
```

## Links

- Library docs: <https://victoraut.github.io/liken/>
- Repository: <https://github.com/VictorAut/liken>
