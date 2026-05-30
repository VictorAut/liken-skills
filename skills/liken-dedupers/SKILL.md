---
name: liken-dedupers
description: >-
  How to choose and apply Liken's built-in dedupers — exact, fuzzy, tfidf, lsh, jaccard,
  cosine, and the isna/isin/str_contains/str_startswith/str_endswith/str_len predicate
  dedupers — via `.apply()` with a single deduper or a dict collection, including the
  pandas accessor affordances. Use when selecting a Liken deduper, setting a threshold,
  tuning tfidf/lsh, or deduplicating multiple columns with different methods. For rule
  composition (AND/OR/NOT) use liken-pipelines instead.
license: Apache-2.0
---

# Applying Liken dedupers

A **deduper** is a matching rule. You `.apply()` it to a `lk.dedupe(df)` and then call
`.drop_duplicates(...)`. Always `import liken as lk`. Targets `liken >= 0.8`.

For the full parameter list of every deduper, see
[references/deduper-catalog.md](references/deduper-catalog.md).

## Two families of deduper

- **Similarity** — link records whose similarity exceeds a `threshold` (default `0.95`):
  `exact`, `fuzzy`, `tfidf`, `lsh`, `jaccard`, `cosine`.
- **Predicate** — filter-like yes/no: `isna`, `isin`, `str_contains`, `str_startswith`,
  `str_endswith`, `str_len`. Predicates are most useful inside **pipelines**
  (AND-combined with a similarity deduper) — see [liken-pipelines](../liken-pipelines/SKILL.md).
  Standalone, their use is limited.

Single-column dedupers (`fuzzy`, `tfidf`, `lsh`, predicates) take one column; compound
dedupers (`jaccard` for categorical, `cosine` for numerical) take several; `exact` does
either.

## Single deduper

One rule, one set of columns. The column(s) go in `drop_duplicates(...)`.

```python
import liken as lk

df = (
    lk.dedupe(df)
    .apply(lk.fuzzy(threshold=0.9))
    .drop_duplicates("address")
)
```

Compound dedupers take a tuple of columns:

```python
df = lk.dedupe(df).apply(lk.jaccard()).drop_duplicates(("account", "birth_country", "marital_status"))
df = lk.dedupe(df).apply(lk.cosine()).drop_duplicates(("property_area_sq_ft", "property_num_rooms"))
```

### Pandas accessor affordance

After `import liken`, pandas DataFrames gain a deduper accessor so you can stay close to
`pandas.drop_duplicates`. Deduper kwargs are passed to `drop_duplicates`:

```python
import liken as lk          # required — registers the accessor
import pandas as pd

df = df.fuzzy.drop_duplicates("address", threshold=0.6)
df = df.tfidf.drop_duplicates("address", threshold=0.9, ngram=(1, 2))
```

Affordances exist only for the similarity dedupers `fuzzy`, `tfidf`, `lsh`, `jaccard`,
`cosine`, only for **single** dedupers (no collections), and only for pandas.

## Dict collection — different rules per column

Map a column (or column tuple) to a deduper, or a tuple of dedupers applied sequentially.
The keys are the columns, so `drop_duplicates()` takes **no** column argument:

```python
import liken as lk

collection = {
    "email": lk.exact(),
    "address": (
        lk.fuzzy(threshold=0.98),
        lk.tfidf(threshold=0.9, ngram=(1, 2), topn=1),
    ),
}

df = (
    lk.dedupe(df)
    .apply(collection)
    .drop_duplicates(keep="first")
)
```

Reads as: dedupe exact emails; and dedupe addresses that are fuzzy-similar then
TF-IDF-similar. Different keys are OR'd together.

## Common parameters

- `threshold` (similarity dedupers, default `0.95`): higher = stricter (fewer matches).
- `fuzzy(scorer=...)`: `"simple_ratio"` (default), `"partial_ratio"`,
  `"token_sort_ratio"`, `"token_set_ratio"`, `"weighted_ratio"`, `"quick_ratio"`.
- `tfidf(ngram=..., topn=..., **vectorizer_kwargs)`: `ngram` is an int **or** an
  `(low, high)` range of character n-grams; extra kwargs go to scikit-learn's
  `TfidfVectorizer`.
- `lsh(ngram=..., num_perm=...)`: `ngram` is a single int here (not a range); `num_perm`
  defaults to `128` (avoid `< 64`).
- `keep`: `"first"` (default) or `"last"` — which record of a duplicate group to keep.

## Gotchas

- **Columns belong in different places per tier.** Single deduper → `drop_duplicates(...)`;
  dict → the keys (so `drop_duplicates()` takes none). Mixing these up raises a
  `ValueError`.
- **`tfidf.ngram` is a range, `lsh.ngram` is a single int.** Larger n-grams reduce
  matching; too-small n-grams cause false positives.
- **`keep` applies to the whole collection**, not per-deduper.
- Predicate dedupers standalone are rarely what you want — compose them in a
  [pipeline](../liken-pipelines/SKILL.md).
- To **link** rather than drop, swap `.drop_duplicates()` for `.canonicalize()` — see
  [liken-record-linkage](../liken-record-linkage/SKILL.md).
