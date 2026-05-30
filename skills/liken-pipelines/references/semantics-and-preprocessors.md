# Pipeline semantics & preprocessors (deep dive)

Load this for the details behind the [liken-pipelines](../SKILL.md) skill: rule
predication, the full preprocessor list with selection guidance, and the complete tiered
recipe.

## Rule predication (the AND optimization)

When you AND dedupers inside one `.step([...])`, Liken reorders execution so **predicate
dedupers run first**, regardless of how you wrote the list. Predicates are roughly O(n)
filters; similarity dedupers are roughly O(n²). Running the predicate first shrinks the
candidate set the similarity deduper has to scan.

Practical consequences:

- Order inside a `.step([...])` list does not matter for correctness or performance —
  Liken sorts predicates first.
- AND-combining a similarity deduper with a cheap predicate (e.g. `~isna()`,
  `str_len(...)`, `str_startswith(...)`) is both more precise *and* faster than the
  similarity deduper alone.
- This is why predicate dedupers are recommended almost exclusively inside pipelines.

## AND / OR / NOT recap

- **AND** — list in one step; all conditions must hold. `~` negation available on
  predicates within the list.
- **OR** — separate steps; any linking step suffices. Equivalent to a dict collection
  when no other features are needed.
- **NOT** — `~lk.col("col").<predicate>()`. Similarity dedupers cannot be negated
  (`TypeError`); write a [custom deduper](../../liken-custom-dedupers/SKILL.md) for the
  inverse instead.

## Preprocessor scoping

Three scopes, most-specific wins:

1. **Pipeline** — `lk.pipeline(preprocessors=...)`. Default for every step/col.
2. **Step** — `.step(cols, preprocessors=...)`. Overrides the pipeline default for that
   step.
3. **Col** — `lk.col("c", preprocessors=...)`. Overrides step and pipeline for that col.

Preprocessors propagate top-down and are overridden bottom-up. A scope that sets its own
preprocessors replaces (does not merge with) the inherited ones.

## Available preprocessors

All from `lk.preprocessors`:

| Preprocessor | Effect |
| --- | --- |
| `strip()` | trim leading/trailing whitespace |
| `lower()` | lowercase |
| `alnum()` | remove all non-alphanumerics **including spaces** |
| `remove_punctuation()` | remove punctuation, **keep** spaces |
| `normalize_unicode(form="NFKD")` | Unicode normalization (`NFC`/`NFKC`/`NFD`/`NFKD`) |
| `ascii_fold()` | map accented/non-ASCII chars to ASCII (à → a) |
| `remove_stopwords(words=None, language="english")` | drop stopwords |
| `normalize_names()` | keep first/middle/last; strip titles & nicknames |
| `normalize_company()` | strip company suffixes (Ltd, LLC, ...) |

## Which preprocessor when?

- `strip()`, `remove_punctuation()`, `normalize_unicode()` — nearly always safe; use
  liberally.
- `lower()`, `ascii_fold()` — usually helpful but more nuanced (case/diacritics can carry
  meaning); apply with care.
- `alnum()` — strips spaces, so it pairs well with `fuzzy` but is risky with the
  tokenization dedupers `tfidf` and `lsh`, which rely on token/character structure.
- `normalize_names()` / `normalize_company()` — domain-specific; use on person/company
  name columns respectively.

Prefer preprocessors over hand-rolled preprocessing before Liken: less boilerplate, fewer
"dummy" columns, and the original values are preserved in your output.

## Full tiered recipe

```python
import liken as lk

pipeline = (
    lk.pipeline(preprocessors=[lk.preprocessors.strip(), lk.preprocessors.lower()])
    .step(
        [
            lk.col("address").exact(),
            lk.col("address").str_len(max_len=5),
            ~lk.col("address").isna(),
        ],
    )
    .step(
        [
            lk.col("address").fuzzy(threshold=0.95),
            lk.col("address").str_len(min_len=5, max_len=10),
        ],
    )
    .step(
        [
            lk.col("address").fuzzy(threshold=0.85),
            lk.col("address").str_len(min_len=10, max_len=20),
        ],
    )
    .step(
        [
            lk.col("address").fuzzy(threshold=0.75),
            lk.col("address").str_len(min_len=20),
        ],
    )
)

df = lk.dedupe(df).apply(pipeline).drop_duplicates()
```
