---
name: liken
description: >-
  Overview and entry point for Liken, a Python library (`import liken as lk`) for
  near-deduplication, fuzzy matching, and record linkage over pandas, Polars, PySpark,
  Dask, Ray and Modin DataFrames. Use when the user wants to deduplicate, fuzzy-match,
  or link DataFrame records with Liken, mentions `lk.dedupe`, or asks which Liken API to
  use. Routes to the liken-dedupers, liken-pipelines, liken-record-linkage,
  liken-custom-dedupers, and liken-backends-performance skills.
license: Apache-2.0
---

# Liken

Liken adds near-deduplication and record linkage to DataFrames. You compose **dedupers**
(matching rules) and apply them to a DataFrame to either drop or link duplicate records.
The same code runs on pandas, Polars, Modin, Dask, Ray and PySpark.

Targets `liken >= 0.8`. Always `import liken as lk`.

## Install

```shell
pip install liken                 # pandas + Polars (core)
pip install 'liken[pyspark]'      # also: [dask] [modin] [ray] [all]
```

## Simplest example — exact dedupe

`lk.dedupe(df)` wraps a DataFrame; with no deduper applied it does an exact dedupe:

```python
import liken as lk

df = lk.dedupe(df).drop_duplicates("address")              # single column
df = lk.dedupe(df).drop_duplicates(("address", "email"))   # multiple columns
```

But real data is rarely *exactly* equal — `fizzpop@yahoo.com` vs `FizzPop@yahoo.com` are
"the same" to a human. That is **near deduplication**, and is what Liken is for. Apply a
deduper to match approximately:

```python
df = lk.dedupe(df).apply(lk.fuzzy()).drop_duplicates("address")
```

Use `lk.datasets.fake_10()` (columns include `address`, `email`, `account`, ...) to
experiment without your own data.

## The API progression — pick the right tier

Liken's APIs get incrementally more powerful. Reach for the simplest one that fits:

| Tier | Looks like | Use when | Skill |
| --- | --- | --- | --- |
| **Single deduper** | `.apply(lk.fuzzy()).drop_duplicates("col")` | one rule on one column | [liken-dedupers](../liken-dedupers/SKILL.md) |
| **Dict collection** | `.apply({"email": lk.exact(), "address": (lk.fuzzy(), lk.tfidf())})` | different rules per column (OR across columns) | [liken-dedupers](../liken-dedupers/SKILL.md) |
| **Pipeline** | `.apply(lk.pipeline().step(...).step(...))` | AND/OR/NOT rules, preprocessors, tiered matching | [liken-pipelines](../liken-pipelines/SKILL.md) |

Cross-cutting capabilities (combine with any tier):

- **Link instead of drop** (entity resolution, `canonical_id`, golden records) → [liken-record-linkage](../liken-record-linkage/SKILL.md)
- **A matching rule the built-ins don't cover** → [liken-custom-dedupers](../liken-custom-dedupers/SKILL.md)
- **Large data, backend choice, slow runs** → [liken-backends-performance](../liken-backends-performance/SKILL.md)

See [references/api-decision-guide.md](references/api-decision-guide.md) for a fuller
breakdown of the tiers and the deduper categories (similarity vs predicate, single vs
compound column).

## Key terms

- **Deduper** — a matching rule. Either a **similarity** deduper (has a `threshold`:
  `exact`, `fuzzy`, `tfidf`, `lsh`, `jaccard`, `cosine`) or a **predicate** deduper (a
  filter-like yes/no: `isna`, `isin`, `str_contains`, `str_startswith`, `str_endswith`,
  `str_len`).
- **`drop_duplicates(...)`** — enacts the applied dedupers and removes duplicates.
- **`canonicalize(...)`** — links duplicates with a `canonical_id` instead of dropping
  them; returns a `Dedupe` you then `.collect()`.

## Gotchas

- Column placement depends on the tier: single deduper → columns go in
  `drop_duplicates(...)`; **dict** → columns are the keys, so `drop_duplicates()` takes
  none; **pipeline** → columns go in `lk.col(...)`.
- The pandas accessor affordance (`df.fuzzy.drop_duplicates("address", threshold=0.6)`)
  only works *after* `import liken`. See [liken-dedupers](../liken-dedupers/SKILL.md).
