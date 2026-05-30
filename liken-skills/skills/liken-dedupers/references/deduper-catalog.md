# Liken deduper catalog

Every built-in deduper, with its category and parameters. Signatures are the public
functions exported from `liken` (verified against the `liken >= 0.8` source). All are
called as `lk.<name>(...)`.

## Similarity dedupers (have a `threshold`, default `0.95`)

| Deduper | Columns | Signature | Notes |
| --- | --- | --- | --- |
| `exact` | single or compound | `exact()` | Default deduper if none applied. Exact equality. |
| `fuzzy` | single | `fuzzy(threshold=0.95, scorer="simple_ratio")` | RapidFuzz string similarity. |
| `tfidf` | single | `tfidf(threshold=0.95, ngram=3, topn=2, **vectorizer_kwargs)` | Char n-gram TF-IDF; `ngram` int or `(low, high)` range; extra kwargs → sklearn `TfidfVectorizer`. |
| `lsh` | single | `lsh(threshold=0.95, ngram=3, num_perm=128)` | MinHash LSH; `ngram` is a single int; scales well. |
| `jaccard` | compound | `jaccard(threshold=0.95)` | Set similarity over categorical columns; nulls treated as their own category. |
| `cosine` | compound | `cosine(threshold=0.95)` | Cosine similarity over numerical columns; nulls dropped from the dot product (caution on sparse data). |

`fuzzy` scorers: `"simple_ratio"`, `"partial_ratio"`, `"token_sort_ratio"`,
`"token_set_ratio"`, `"weighted_ratio"`, `"quick_ratio"`.

## Predicate dedupers (filter-like; negatable with `~` in pipelines)

| Deduper | Signature | Matches |
| --- | --- | --- |
| `isna` | `isna()` | null / missing values |
| `isin` | `isin(values)` | value is a member of `values` (an iterable, e.g. a list) |
| `str_contains` | `str_contains(pattern, case=True, regex=False)` | string contains `pattern` (literal or regex) |
| `str_startswith` | `str_startswith(pattern, case=True)` | string starts with `pattern` (no regex) |
| `str_endswith` | `str_endswith(pattern, case=True)` | string ends with `pattern` (no regex) |
| `str_len` | `str_len(min_len=0, max_len=None)` | string length in `(min_len, max_len]`; for an exact length use `max_len = min_len + 1` |

Notes:

- `~` negation is supported on predicate dedupers (e.g. `~lk.col("address").isna()`) and
  only inside the pipeline API — see [liken-pipelines](../../liken-pipelines/SKILL.md).
- `str_startswith` / `str_endswith` do not accept regex; use `str_contains(regex=True)`.
- `isin(values=...)` expects an iterable; pass a list (`["london", "paris"]`) — a bare
  string would be iterated character-by-character.

## Pandas affordance support

Only `fuzzy`, `tfidf`, `lsh`, `jaccard`, `cosine` are available as the pandas accessor
(`df.fuzzy.drop_duplicates(...)`), only for single dedupers, only on pandas DataFrames,
and only after `import liken`.

## Choosing between fuzzy / tfidf / lsh

- `fuzzy` — best default for short strings (names, addresses); intuitive `threshold`.
- `tfidf` — token/character-overlap similarity; tune `ngram` and `topn`; good for longer
  strings; experiment.
- `lsh` — approximate, scales to large data (sub-quadratic); needs tuning of `ngram` and
  `num_perm`; verify quality on a sample. See
  [liken-backends-performance](../../liken-backends-performance/SKILL.md).
