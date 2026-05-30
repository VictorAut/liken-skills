---
name: liken-pipelines
description: >-
  How to build Liken pipelines with `lk.pipeline()` and `lk.col()` for complex matching
  rules: AND semantics (a list of dedupers in one `.step()`), OR semantics (separate
  `.step()` calls), NOT (`~` on a predicate deduper), preprocessor scoping
  (pipeline/step/col), and the rule-predication optimization. Use for multi-stage or
  tiered deduplication, for combining predicate + similarity dedupers, or for applying
  preprocessing inside Liken. For single dedupers or simple dict collections use
  liken-dedupers instead.
license: Apache-2.0
---

# Liken pipelines

Pipelines are Liken's most expressive collection: logical rules (AND/OR/NOT), built-in
preprocessors, and an automatic optimization. Build one with `lk.pipeline()`, add steps
with `.step(...)`, and reference columns with `lk.col(...)`. Always `import liken as lk`.
Targets `liken >= 0.8`.

For deeper detail (rule predication, the full preprocessor list and selection guidance,
the tiered recipe), see
[references/semantics-and-preprocessors.md](references/semantics-and-preprocessors.md).

## Anatomy

- `lk.pipeline(preprocessors=...)` — start a pipeline (optional pipeline-wide preprocessors).
- `.step(cols, *, preprocessors=...)` — one deduplication step; `cols` is a single
  `lk.col(...)` or a **list** of them.
- `lk.col("column").<deduper>(...)` — a column with a deduper as a chained method
  (`.fuzzy()`, `.tfidf()`, `.isna()`, `.str_len(...)`, etc., plus any registered
  [custom deduper](../liken-custom-dedupers/SKILL.md)).

```python
import liken as lk

pipeline = (
    lk.pipeline()
    .step(lk.col("email").exact())
    .step(lk.col("address").tfidf(threshold=0.9, ngram=(1, 2), topn=1))
)

df = lk.dedupe(df).apply(pipeline).drop_duplicates()
```

## AND / OR / NOT

```python
import liken as lk

pipeline = (
    lk.pipeline()
    # AND: a list inside one step — ALL conditions must hold for records to link
    .step(
        [
            lk.col("address").fuzzy(threshold=0.9),
            lk.col("address").str_len(min_len=10),
            ~lk.col("address").isna(),          # NOT: negate a predicate deduper
        ]
    )
    # OR: a separate step — either step linking is enough
    .step(lk.col("email").fuzzy(threshold=0.98))
)

df = lk.dedupe(df).apply(pipeline).drop_duplicates()
```

- **AND** = multiple dedupers in one `.step([...])`. All must match.
- **OR** = multiple `.step()` calls. Any step linking records is enough. (If you only need
  OR, a [dict collection](../liken-dedupers/SKILL.md) is simpler.)
- **NOT** = `~` on the `lk.col(...)` expression. Only **predicate** dedupers can be
  negated (`isna`, `isin`, `str_*`); negating a similarity deduper raises `TypeError`.

AND is most powerful combining a predicate with a similarity deduper (e.g. "fuzzy address
AND address not null"). Liken applies **rule predication**: within an AND step the
predicate dedupers run first (≈O(n)), shrinking the data the similarity deduper
(≈O(n²)) then scans — so order inside the list doesn't matter.

## Preprocessors

Preprocessors (from `lk.preprocessors`) transform values *only inside* the deduplication —
your returned DataFrame keeps its original values. Pass one or a list, at three scopes:

```python
import liken as lk

pipeline = (
    lk.pipeline(preprocessors=[lk.preprocessors.lower()])     # pipeline scope (default for all)
    .step(
        [
            lk.col("email").fuzzy(),
            ~lk.col(
                "address",
                preprocessors=[lk.preprocessors.ascii_fold()],  # col scope (most specific)
            ).isna(),
        ],
        preprocessors=[lk.preprocessors.alnum()],              # step scope
    )
    .step(lk.col("address").tfidf())                            # inherits pipeline's lower()
)
```

**Scoping rule:** preprocessors propagate **top-down** (pipeline → step → col) but are
**overridden bottom-up** — the most specific scope wins. A col with its own preprocessors
ignores the step's and pipeline's; a step with its own ignores the pipeline's.

Common preprocessors: `strip()`, `lower()`, `alnum()` (also strips spaces),
`remove_punctuation()`, `normalize_unicode(form="NFKD")`, `ascii_fold()`,
`remove_stopwords(language="english")`, `normalize_names()`, `normalize_company()`. See
the reference for which to use when.

## Tiered matching (the headline use case)

Loosen the threshold as strings get longer, so long addresses tolerate more variation
while short ones stay strict:

```python
import liken as lk

pipeline = (
    lk.pipeline()
    .step([lk.col("address").exact(), lk.col("address").str_len(max_len=5), ~lk.col("address").isna()])
    .step([lk.col("address").fuzzy(threshold=0.95), lk.col("address").str_len(min_len=5, max_len=10)])
    .step([lk.col("address").fuzzy(threshold=0.85), lk.col("address").str_len(min_len=10, max_len=20)])
    .step([lk.col("address").fuzzy(threshold=0.75), lk.col("address").str_len(min_len=20)])
)

df = lk.dedupe(df).apply(pipeline).drop_duplicates()
```

## Gotchas

- `.step([...])` (a list) = AND; `.step(a).step(b)` (separate) = OR. Easy to confuse.
- `~` only works on predicate dedupers. To "negate" a similarity rule, write a
  [custom deduper](../liken-custom-dedupers/SKILL.md).
- Pipelines define columns via `lk.col(...)`, so `drop_duplicates()` / `canonicalize()`
  take no column argument.
- `str_len` bounds are `min_len`/`max_len` (not `min`/`max`).
- To link instead of drop, apply the same pipeline and call `.canonicalize().collect()` —
  see [liken-record-linkage](../liken-record-linkage/SKILL.md).
