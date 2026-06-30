---
tags: [ml, preprocessing, feature-engineering]
---

# Distribution Transforms

Reshaping the distribution of a numeric column so the model can learn from it more effectively.

> **Not the same as [[Scaling Numeric Features|scaling]].**
> - Transforms change the *shape* of the distribution (fix skewness)
> - Scaling changes the *range/magnitude* (e.g. compress to 0–1)
> You often do both: transform first, then scale.

## What is skewness?

A right-skewed column has a long tail to the right — a few very large values (income, revenue, page views). Linear models and neural networks get confused by this because the extreme values dominate.

```python
print(df['income'].skew())
# 0  = perfectly symmetric
# > 1 or < -1 = meaningfully skewed → apply a transform
```

## Log transform (most common)

```python
import numpy as np

# np.log1p = log(x + 1). The +1 makes it safe when x = 0
df['income_log'] = np.log1p(df['income'])

print(df['income'].skew())      # Before: 4.2
print(df['income_log'].skew())  # After:  0.3  ← much better
```

Use for: income, revenue, prices, counts — anything right-skewed with no negatives.

## Square root (gentler)

```python
df['visits_sqrt'] = np.sqrt(df['page_visits'])
```

Use for: counts with moderate skew.

## Yeo-Johnson (automatic — finds the best power)

```python
from sklearn.preprocessing import PowerTransformer

pt = PowerTransformer(method='yeo-johnson')
df[['income_yt']] = pt.fit_transform(df[['income']])
# Works with zeros and negative values (Box-Cox requires positive)
```

## Binning — convert continuous to groups

```python
df['age_band'] = pd.cut(
    df['age'],
    bins=[0, 25, 35, 50, 65, 100],
    labels=['<25', '25-35', '35-50', '50-65', '65+']
)
```

Use when: the exact value matters less than the group (age bands, income brackets).

## Which models need transforms?

| Model | Needs transform? |
|---|---|
| Linear Regression, Logistic Regression | ✅ Yes |
| SVM, k-NN, Neural Networks | ✅ Yes |
| Random Forest, XGBoost, LightGBM | ❌ No — trees split on thresholds, don't care about shape |

## Related
- [[Outliers]]
- [[Scaling Numeric Features]]
