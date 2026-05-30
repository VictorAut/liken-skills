# Liken API decision guide

A fuller breakdown of the API tiers and deduper categories. Load this when you need to
choose between Single / Dict / Pipeline, or to understand how dedupers are classified.

## Choosing a collection tier

| Collection | Pandas accessor | Quick tasks | Multiple columns | Logical rules (AND/OR/NOT) | Preprocessors |
| --- | --- | --- | --- | --- | --- |
| **Single** | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Dict** | ❌ | ✅ | ✅ | ❌ (OR-across-columns only) | ❌ |
| **Pipeline** | ❌ | ❌ | ✅ | ✅ | ✅ |

Rules of thumb:

- **Single deduper** — one rule, one set of columns. Simplest. Columns go in
  `drop_duplicates(...)`. Only this tier supports the pandas accessor affordance.
- **Dict** — map each column (or column tuple) to a deduper or a tuple of dedupers run
  sequentially. Columns are the dict keys, so `drop_duplicates()` takes no column
  argument. Different keys are effectively OR'd. Good for "different rules per column"
  without logical composition.
- **Pipeline** — `lk.pipeline()` + `lk.col()`. The only tier with AND/OR/NOT rule
  semantics, preprocessors, and the rule-predication optimization. Use for tiered or
  multi-condition matching.

A dict is shorthand for a pipeline that only uses OR. If you need anything beyond
OR-across-columns, use a pipeline.

## Deduper categories

Every built-in deduper is classified two ways.

### Similarity vs predicate

- **Similarity** dedupers link records whose similarity exceeds a `threshold` (default
  `0.95` for all of them): `exact` (implicit threshold), `fuzzy`, `tfidf`, `lsh`,
  `jaccard`, `cosine`.
- **Predicate** dedupers are filter-like — a record either satisfies the condition or it
  does not: `isna`, `isin`, `str_contains`, `str_startswith`, `str_endswith`, `str_len`.
  They are most useful inside pipelines, AND-combined with a similarity deduper, and can
  be negated with `~`.

### Single-column vs compound-column

- **Single-column**: `exact`, `fuzzy`, `tfidf`, `lsh`, and all predicates. Operate on one
  column.
- **Compound-column** (set operations across several columns of a row): `jaccard`
  (categorical data) and `cosine` (numerical data). `exact` also works on multiple
  columns.

See the [liken-dedupers deduper catalog](../../liken-dedupers/references/deduper-catalog.md)
for every deduper's parameters.

## Drop vs link

- `drop_duplicates(columns=None, *, keep="first")` removes duplicates and returns the
  DataFrame.
- `canonicalize(columns=None, *, keep="first", drop_duplicates=False, id=None)` keeps all
  rows but adds a `canonical_id`; returns a `Dedupe` — call `.collect()` for the
  DataFrame, `.canonicals()` for group counts, `.synthesize()` for golden records. See
  [liken-record-linkage](../../liken-record-linkage/SKILL.md).
