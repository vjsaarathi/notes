---
tags: [ml, preprocessing, data-cleaning]
---

# Duplicates

Identical (or near-identical) rows that appear more than once. Usually caused by joining data sources, ETL jobs running twice, or form re-submissions.

**Why they matter:** The model trains on duplicate rows multiple times — it learns those patterns more strongly than it should, inflating apparent sample size.

## Exact duplicates

```python
# How many?
print(df.duplicated().sum())

# See them
print(df[df.duplicated(keep=False)])

# Drop — keep the first occurrence
df = df.drop_duplicates()

# Drop based on specific columns (same person, different timestamp — keep latest)
df = df.drop_duplicates(subset=['user_id', 'email'], keep='last')
```

## Near-duplicates (fuzzy matching)

When rows represent the same entity but with slight differences ("Bengaluru" vs "Bangalore"), exact matching won't catch them.

```python
# pip install thefuzz
from thefuzz import fuzz

score = fuzz.ratio('Bengaluru', 'Bangalore')
print(score)  # → 82 out of 100

# For large datasets, use the `recordlinkage` library
```

## ⚠️ Exception: time-series data

In time-series, the same value at different timestamps is NOT a duplicate — it is two separate events. Always check the timestamp column before dropping.

## Related
- [[Raw Data]]
- [[Inconsistent Labels]]
