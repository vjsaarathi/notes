---
tags: [ml, preprocessing]
---

# Data Leakage

Information from outside the training set accidentally influences the model during training. The model appears to perform well in evaluation but **fails in production**.

This is one of the most common and dangerous mistakes in ML.

---

## Type 1 — Fitting preprocessors on the full dataset

```python
# ❌ WRONG — StandardScaler sees test data when it computes mean/std
scaler = StandardScaler()
X_all_scaled   = scaler.fit_transform(X)          # leaks test statistics
X_train_scaled = X_all_scaled[:train_size]
X_test_scaled  = X_all_scaled[train_size:]

# ✅ CORRECT — fit on training data only
scaler = StandardScaler()
scaler.fit(X_train)                               # learns only from train
X_train_scaled = scaler.transform(X_train)
X_test_scaled  = scaler.transform(X_test)         # applies train's stats to test
```

Same applies to: `SimpleImputer`, `KNNImputer`, `OneHotEncoder`, `OrdinalEncoder`, `PCA`, feature selectors.

---

## Type 2 — Using future information as a feature

```python
# ❌ WRONG — predicting loan default but including "missed_payments"
# At prediction time, the loan hasn't been issued yet.
# This feature only exists because the customer already defaulted.
df['missed_payments'] = ...   # this is a future outcome, not an input
```

Ask yourself: *"At the time I need to make this prediction, would I actually have this value?"*

---

## Type 3 — Target encoding on the full dataset

```python
# ❌ WRONG — city_mean includes information from test rows
city_means = df.groupby('city')['loan_approved'].mean()  # full df!
df['city_mean'] = df['city'].map(city_means)

# ✅ CORRECT — compute only from training rows
city_means = y_train.groupby(X_train['city']).mean()
X_train['city_mean'] = X_train['city'].map(city_means)
X_test['city_mean']  = X_test['city'].map(city_means)    # map, not recompute
```

---

## The golden rule

> **Split first. Fit everything on training data. Transform (never re-fit) on validation and test.**

---

## The clean solution — scikit-learn Pipelines

Pipelines make leakage structurally impossible:

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression

pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler',  StandardScaler()),
    ('model',   LogisticRegression()),
])

# When you call fit(), the pipeline fits the imputer and scaler on X_train,
# then transforms X_train, then trains the model — all in one go.
pipeline.fit(X_train, y_train)

# When you call predict/score, it transforms X_test using the already-fitted
# imputer and scaler — never re-fits them.
score = pipeline.score(X_test, y_test)
```

## Related
- [[Train Validation Test Split]]
- [[Missing Values]]
- [[Scaling Numeric Features]]
- [[Encoding Categorical Variables]]
- [[Feature Creation]]
- [[Preprocessing Pipeline - Correct Order]]
