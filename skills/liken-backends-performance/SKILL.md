---
name: liken-backends-performance
description: >-
  How to run Liken across DataFrame backends (pandas, Polars, Modin, Dask, Ray, PySpark)
  and make deduplication scale: install extras, pass a `spark_session` for PySpark,
  partition with a blocking key, choose the LSH deduper for large data, and use
  predicate pushdown to cut O(n^2) cost. Use when scaling Liken to large datasets, picking
  a backend, fixing slow or expensive deduplication, or running on Spark/Dask/Ray.
license: Apache-2.0
---

# Liken backends & performance

Near-deduplication is compute-intensive — it scales ≈O(n²) because each record is
compared against every other. This skill covers running Liken on different DataFrame
backends and the levers that make it scale. Always `import liken as lk`. Targets
`liken >= 0.8`.

## Backends

The same Liken code runs on several DataFrame libraries; the backend is **auto-detected**
from the DataFrame you pass to `lk.dedupe(df)`.

```shell
pip install liken                 # pandas + Polars (core, no extra needed)
pip install 'liken[pyspark]'      # PySpark
pip install 'liken[dask]'         # Dask
pip install 'liken[modin]'        # Modin (pulls dask + ray)
pip install 'liken[ray]'          # Ray Datasets
pip install 'liken[all]'          # everything
```

```python
import liken as lk

# pandas / Polars / Modin / Dask / Ray: just pass the DataFrame
df = lk.dedupe(df).apply(lk.fuzzy()).drop_duplicates("address")

# PySpark: pass the active SparkSession
df = (
    lk.dedupe(spark_df, spark_session=spark)
    .apply(lk.fuzzy())
    .drop_duplicates("address")
)
```

You get back a DataFrame of the same library you put in.

## Performance levers

### 1. Predicate pushdown (cheapest win)

In a pipeline AND-step, Liken runs predicate dedupers first (≈O(n)), shrinking the data
the similarity deduper (≈O(n²)) then scans. Qualify a similarity rule with a cheap,
well-motivated predicate:

```python
import liken as lk

df = (
    lk.dedupe(df)
    .apply(
        lk.pipeline().step(
            [
                lk.col("email").exact(),
                ~lk.col("address").isna(),   # runs first → fewer rows for exact
            ]
        )
    )
    .drop_duplicates()
)
```

See [liken-pipelines](../liken-pipelines/SKILL.md). Order inside the step doesn't matter —
Liken sorts predicates first automatically.

### 2. LSH for large data

`lk.lsh(...)` is approximately O(nk) with k ≪ n — far faster than the O(n²) dedupers and
able to scale to very large datasets (estimated ~10M rows in single-digit hours on a
standard machine). It is approximate, so **tune** `ngram` and `num_perm` and validate
quality on a sample.

### 3. Partition with a blocking key (distributed)

On PySpark, Liken is re-instantiated per worker and each worker processes its partition.
Partition by a **blocking key** — a column where duplicates never (or almost never) cross
partitions, e.g. the first letter of a name or a postcode prefix. Read data already
partitioned, or repartition before deduping. This bounds the O(n²) cost to within each
block instead of the whole dataset.

## Cost notes & gotchas

- **`str_*` dedupers explode on common patterns.** `str_contains("a")` matches almost
  everything and blows up; pick meaningful, sparse patterns (`"street"`, `"ltd"`). Same
  for `str_len` — fast when the chosen bounds are rare in the data.
- **`cosine`/`jaccard` are O(n²).** Doubling rows ≈ quadruples time (100K→200K ≈ 2h→8h on
  nominal data). Prefer `lsh` or a blocking key at scale.
- Average **string length** strongly affects per-deduper cost.
- **Record linkage on distributed backends:** with the default auto-increment
  `canonical_id`, `.canonicalize()` collects to the driver on Dask/Ray — pass `id=`
  your unique column instead. `.canonicals()` and `.synthesize()` collect to the driver
  and need PySpark v4+. See [liken-record-linkage](../liken-record-linkage/SKILL.md).
- Test scaling with `lk.datasets.fake_1K()`, `fake_100K()`, `fake_1M()` (each takes a
  `backend=` argument).
