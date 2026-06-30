---
tags: [ml, preprocessing, scaling]
---

# Scaling Numeric Features

Numeric columns often have very different ranges — `age` (18–80) vs `income` (20,000–5,00,000). Many models treat larger numbers as more important. Scaling puts all columns on a comparable scale.

> Scaling changes the **range/magnitude** of values.
> [[Distribution Transforms]] change the *shape* of the distribution.
> Do transforms first, then scale.

---

## Min-Max Normalisation → [0, 1]

Formula: `x_scaled = (x - min) / (max - min)`

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler()
scaler.fit(X_train)                          # learn min/max from training data
X_train_scaled = scaler.transform(X_train)
X_test_scaled  = scaler.transform(X_test)    # use the SAME scaler
```

Use when: you need values strictly between 0 and 1 (e.g. neural network inputs), or the distribution is not Gaussian.

**Downside:** Sensitive to outliers — one extreme value compresses everything else to near-zero.

---

## Standardisation (Z-score) → mean=0, std=1

Formula: `x_scaled = (x - mean) / std`

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
scaler.fit(X_train)
X_train_scaled = scaler.transform(X_train)
X_test_scaled  = scaler.transform(X_test)
```

Use when: features are roughly normally distributed. Most common default choice for linear models and SVMs.

---

## Robust Scaling (uses median and IQR)

```python
from sklearn.preprocessing import RobustScaler

scaler = RobustScaler()
scaler.fit(X_train)
X_train_scaled = scaler.transform(X_train)
X_test_scaled  = scaler.transform(X_test)
```

Use when: data has significant [[Outliers]] that you don't want to remove. Median and IQR are not affected by extreme values.

---

## Which models need scaling?

| Model | Needs scaling? |
|---|---|
| Linear Regression, Logistic Regression | ✅ Yes |
| SVM | ✅ Yes (very sensitive) |
| k-Nearest Neighbours | ✅ Yes (distance-based) |
| Neural Networks | ✅ Yes |
| PCA | ✅ Always scale before PCA |
| Decision Tree | ❌ No |
| Random Forest | ❌ No |
| XGBoost / LightGBM / CatBoost | ❌ No |
| Naive Bayes | ❌ No |

## ⚠️ Key rule

Always fit the scaler on **training data only**, then `.transform()` on test. Fitting on test data leaks its statistics into training. See [[Data Leakage]].

## Related
- [[Distribution Transforms]]
- [[Outliers]]
- [[Data Leakage]]
- [[Feature Extraction - PCA]]
