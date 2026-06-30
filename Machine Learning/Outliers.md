---
tags: [ml, preprocessing, data-cleaning]
---

# Outliers

A value far outside the normal range. Could be a genuine extreme, a data entry error, or a sentinel (e.g. `9999999` meaning "unknown").

**Why they matter:** Outliers drag the mean, distort [[Scaling Numeric Features|scaling]], and confuse distance-based models like k-NN.

## Step 1 — Detect

### IQR fence (robust — works on skewed data)

```python
Q1 = df['income'].quantile(0.25)
Q3 = df['income'].quantile(0.75)
IQR = Q3 - Q1

lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR

outliers = df[(df['income'] < lower) | (df['income'] > upper)]
print(f"Outliers: {len(outliers)}")
```

### Z-score (assumes normal distribution)

```python
from scipy import stats

z = stats.zscore(df['income'])
outliers = df[abs(z) > 3]  # more than 3 std deviations from the mean
```

## Step 2 — Decide what to do

| Situation | Action |
|---|---|
| Clear data entry error (age = 999) | Drop the row |
| Sentinel value (9999999 = "unknown") | Cap or drop |
| Real but extreme value | Log transform |

### Cap / clip

```python
df['income'] = df['income'].clip(lower=lower, upper=upper)
# Values above upper become exactly upper; below lower become exactly lower
```

### Drop the row

```python
df = df[(df['income'] >= lower) & (df['income'] <= upper)]
```

### Log transform

```python
import numpy as np
df['income_log'] = np.log1p(df['income'])  # log(x+1), safe when x=0
```

## ⚠️ Always look before removing

A hedge fund manager with a ₹10 crore income is real — don't delete them. A 999-year-old is not. Understanding *why* a value is an outlier determines what to do with it.

## Related
- [[Raw Data]]
- [[Distribution Transforms]]
- [[Scaling Numeric Features]]
