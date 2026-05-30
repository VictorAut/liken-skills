---
name: liken-record-linkage
description: >-
  How to use Liken for record linkage / entity resolution instead of dropping duplicates:
  `.canonicalize()` to add a `canonical_id` linking matched records, `.collect()` to get
  the DataFrame back, `.canonicals()` to count linked groups, and `.synthesize()` to build
  coalesced "golden" records. Use when the user wants to label/link duplicates rather than
  remove them, mentions entity resolution, canonical IDs, golden records, or
  canonicalization. Works with any deduper or pipeline.
license: Apache-2.0
---

# Record linkage with Liken

Sometimes you don't want to *drop* duplicates — you want to *keep* every record but mark
which ones refer to the same entity. Liken calls this **record linkage** (a.k.a. entity
resolution). The link is a `canonical_id` column: rows sharing a `canonical_id` are the
same entity. Always `import liken as lk`. Targets `liken >= 0.8`.

The dedupers, dict collections, and pipelines are identical to dropping duplicates — you
just swap the terminal call.

## Canonicalize instead of drop

Replace `.drop_duplicates(...)` with `.canonicalize(...)`, then **`.collect()`** the
DataFrame (canonicalize returns a `Dedupe`, not a DataFrame):

```python
import liken as lk

df = (
    lk.dedupe(df)
    .apply(lk.fuzzy(threshold=0.85))
    .canonicalize("email", keep="first")
    .collect()                     # <-- required; canonicalize() does not return the df
)
```

`df` is unchanged except for a new `canonical_id`. By default `canonical_id` is the
auto-incrementing index position of the canonical (kept) record, so matched rows share
that number.

`canonicalize(columns=None, *, keep="first", drop_duplicates=False, id=None)`:

- `drop_duplicates=True` — drop duplicates *while still* assigning a `canonical_id`.
- `id="uid"` — use an existing unique column's values as the `canonical_id` instead of an
  auto-increment. Recommended for Dask/Ray (and avoids collecting to the driver — see
  [liken-backends-performance](../liken-backends-performance/SKILL.md)).

```python
df = (
    lk.dedupe(df)
    .apply(lk.fuzzy(threshold=0.85))
    .canonicalize("email", keep="first", id="uid")
    .collect()
)
```

## Inspect linked groups — `.canonicals()`

After canonicalizing, get a dict of `canonical_id -> count` for groups with at least `n`
records (default `n=2`, i.e. only actual duplicates):

```python
result = lk.dedupe(df).apply(lk.fuzzy(threshold=0.85)).canonicalize("email", keep="first", id="uid")

groups = result.canonicals()        # {canonical_id: count} for groups of >= 2
df = result.collect()               # the DataFrame with canonical_id
```

`n < 2` raises `ValueError`; calling before `canonicalize()` raises `RuntimeError`.

## Build golden records — `.synthesize()`

Coalesce each linked group into one synthetic "golden" record (first non-null value per
field; singletons returned as-is):

```python
result = (
    lk.dedupe(df)
    .apply(lk.fuzzy(threshold=0.85))
    .canonicalize("email", keep="first", id="uid")
)

golden = result.synthesize()        # a DataFrame of synthesized records
```

## Gotchas

- **`.canonicalize()` returns a `Dedupe`, not a DataFrame.** Always `.collect()` (or call
  `.canonicals()` / `.synthesize()`) afterwards. This is the most common mistake.
- Column placement follows the API used to apply dedupers: single deduper → column in
  `canonicalize(...)`; dict → keys; pipeline → `lk.col(...)` (so `canonicalize()` takes no
  column).
- Default auto-increment `canonical_id` forces a collect-to-driver on Dask/Ray; pass
  `id=` your unique column to avoid it.
- `.canonicals()` and `.synthesize()` on PySpark require Spark v4+ and collect to the
  driver — see [liken-backends-performance](../liken-backends-performance/SKILL.md).
