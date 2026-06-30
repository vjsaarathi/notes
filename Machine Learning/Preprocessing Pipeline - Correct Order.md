---
tags: [ml, preprocessing]
---

# Preprocessing Pipeline — Correct Order

The full workflow from raw CSV to a trained model, in the right sequence.

## Order cheat sheet

```
1.  Load data
2.  Audit — shape, dtypes, missing counts, value_counts on categoricals
3.  Fix inconsistent labels         ← safe before split (no stats involved)
4.  Remove duplicates               ← safe before split
5.  Cap hard sentinel outliers      ← safe before split if threshold is domain-known
6.  Create new features             ← safe before split (except target encoding)
7.  ── SPLIT into train / val / test ── ← everything below is fit on train only
8.  Fit imputer on train → transform all
9.  Fit scaler on train  → transform all
10. Fit encoder on train → transform all
11. Feature selection on train
12. Train model on train
13. Evaluate on val → tune hyperparameters
14. Final evaluation on test (once only — never go back and change the model)
```

## Full code example

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.ensemble import RandomForestClassifier

# ── 1. Load ───────────────────────────────────────────────────────────────────
df = pd.read_csv('loans.csv')

# ── 2. Audit ──────────────────────────────────────────────────────────────────
print(df.shape)
print(df.isnull().sum())
for col in df.select_dtypes('object').columns:
    print(df[col].value_counts())

# ── 3–5. Clean (before split) ─────────────────────────────────────────────────
df['gender'] = df['gender'].replace({'Female': 'F', 'Male': 'M'})
df['city']   = df['city'].str.strip().str.title()
df = df.drop_duplicates()
df['income'] = df['income'].clip(upper=500_000)   # cap sentinel

# ── 6. Create features (before split) ─────────────────────────────────────────
df['debt_to_income'] = df['loan_amount'] / df['income'].replace(0, np.nan)
df['is_weekend']     = pd.to_datetime(df['ts']).dt.dayofweek.isin([5,6]).astype(int)

# ── 7. SPLIT ──────────────────────────────────────────────────────────────────
X = df.drop(columns=['loan_approved'])
y = df['loan_approved']

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# ── 8–10. Preprocessing pipelines (fit on train only) ─────────────────────────
numeric_cols     = ['age', 'income', 'credit_score', 'debt_to_income']
categorical_cols = ['city', 'gender']

numeric_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler',  StandardScaler()),
])

categorical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False)),
])

preprocessor = ColumnTransformer([
    ('num', numeric_pipeline,     numeric_cols),
    ('cat', categorical_pipeline, categorical_cols),
])

# ── 11–12. Full pipeline with model ───────────────────────────────────────────
full_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('model',        RandomForestClassifier(n_estimators=100, random_state=42)),
])

full_pipeline.fit(X_train, y_train)

# ── 13–14. Evaluate ───────────────────────────────────────────────────────────
train_score = full_pipeline.score(X_train, y_train)
test_score  = full_pipeline.score(X_test,  y_test)

print(f"Train accuracy: {train_score:.3f}")
print(f"Test  accuracy: {test_score:.3f}")
# Large gap → overfitting → more data, regularisation, or simpler model
```

## Interpreting the scores

| Situation | Diagnosis |
|---|---|
| Train high, Test low | Overfitting — model memorised training data |
| Train low, Test low | Underfitting — model too simple |
| Train ≈ Test (both high) | Good generalisation |

## Related
- [[What is ML Preprocessing]]
- [[Data Leakage]]
- [[Train Validation Test Split]]
- [[ML Preprocessing MOC]]
