---
name: liken-custom-dedupers
description: >-
  How to define and register custom dedupers in Liken with the `@lk.custom.register`
  decorator — backend-agnostic generator functions of the form `(array, *, **kwargs)`
  that yield pairs of row indices `(i, j)` for records that should link. Use when Liken's
  built-in dedupers (fuzzy, tfidf, lsh, jaccard, cosine, predicates) cannot express the
  matching rule you need and bespoke logic is required. Custom dedupers are single-column
  and usable in the single, dict, and pipeline APIs.
license: Apache-2.0
---

# Custom Liken dedupers

When no built-in deduper expresses your rule, write one. A custom deduper is a plain
Python function, registered with `@lk.custom.register`, that Liken then treats like any
other deduper. Always `import liken as lk`. Targets `liken >= 0.8`.

## The contract

A custom deduper function:

- takes a generic, unlabelled `array` (an iterable standing in for one column) as its
  first argument, plus **keyword-only** parameters: `def f(array, *, **kwargs)`;
- assumes **no** knowledge of the DataFrame or backend (pure Python — works across
  pandas/Polars/Spark/etc.);
- **yields `(i, j)` index pairs** for records that should be linked (a generator is
  preferred over building and returning a list);
- is single-column (Liken guarantees custom dedupers for single columns).

## Define and register

```python
import liken as lk

@lk.custom.register
def str_same_len(array, *, min_len: int):
    n = len(array)
    for i in range(n):
        for j in range(i + 1, n):
            if len(array[i]) == len(array[j]) and len(array[i]) > min_len:
                yield i, j
```

You never pass `array` yourself — Liken builds it from the column you target. The
decorator enforces keyword-only arguments at call time (`str_same_len(12)` raises
`TypeError`; `str_same_len(min_len=12)` is correct).

## Use it (works in every API)

```python
import liken as lk

# Single deduper — column in drop_duplicates
df = lk.dedupe(df).apply(str_same_len(min_len=12)).drop_duplicates("address")

# Dict collection — column is the key
df = lk.dedupe(df).apply({"address": str_same_len(min_len=12)}).drop_duplicates()

# Pipeline — registered name is available as an lk.col() method
df = (
    lk.dedupe(df)
    .apply(lk.pipeline().step(lk.col("address").str_same_len(min_len=12)))
    .drop_duplicates()
)
```

In a pipeline a custom deduper can be AND-combined with built-ins inside a `.step([...])`
— see [liken-pipelines](../liken-pipelines/SKILL.md).

## Gotchas

- **Keyword-only args.** Positional arguments raise `TypeError`. Define them after `*`.
- **No `~` negation.** Custom dedupers cannot be inverted with `~`. Register a separate
  function for the inverse (e.g. `not_str_same_len`) that yields the complementary pairs.
- **Single-column only** is what's guaranteed; don't rely on compound-column behaviour.
- **Yield, don't return where possible.** Generators are consumed more efficiently than
  returning a full list.
- **Guard against nulls / non-strings.** `array` may contain `None` (and non-string
  values), so `len(array[i])` can raise. Skip them explicitly, e.g.
  `if array[i] is None or array[j] is None: continue` before comparing.
- Indices are positions in the (preprocessed) column array, not DataFrame labels.
