---
tags: [ml, preprocessing, data-cleaning]
---

# Missing Values

`NaN` means "no value was recorded here." Most ML models cannot handle `NaN` — they either crash or silently produce wrong predictions.

## Step 1 — Audit

```python
missing_pct = df.isnull().mean() * 100
print(missing_pct)
```

**Rule of thumb:**
- `< 5%` missing → safe to drop those rows, or fill
- `5–30%` missing → fill using a strategy
- `> 30%` missing → consider dropping the entire column

## Step 2 — Choose a strategy per column

### A — Drop rows

```python
df.dropna(subset=['age'], inplace=True)
```

Best when: very few rows are missing and you can afford to lose them.

### B — Fill with median or mean

```python
# Median: better when column has outliers
df['age'] = df['age'].fillna(df['age'].median())

# Mean: fine when distribution is symmetric (no big outliers)
df['age'] = df['age'].fillna(df['age'].mean())

# Mode: for categorical columns
df['city'] = df['city'].fillna(df['city'].mode()[0])
```

### C — KNN Imputer (most accurate)

Looks at the 5 most similar rows and fills with their average.

```python
from sklearn.impute import KNNImputer

imputer = KNNImputer(n_neighbors=5)
imputer.fit(X_train)                        # fit on training data only
X_train_imputed = imputer.transform(X_train)
X_test_imputed  = imputer.transform(X_test) # same imputer, NOT re-fitted
```

### D — "Missing" as its own feature

Sometimes *the fact that a value is missing* is itself informative.

```python
df['income_missing'] = df['income'].isnull().astype(int)  # 1 = was missing
df['income'] = df['income'].fillna(df['income'].median())  # then fill
```

## ⚠️ Key rule
Never fit the imputer on test data. Fit on training data, then `.transform()` on test. See [[Data Leakage]].

## Related
- [[Raw Data]]
- [[Data Leakage]]
- [[Preprocessing Pipeline - Correct Order]]
