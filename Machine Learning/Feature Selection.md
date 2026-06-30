---
tags: [ml, preprocessing, feature-engineering]
---

# Feature Selection

Keeping only the columns that are actually useful to the model. Removes irrelevant and redundant features.

**Why it matters:**
- Fewer features → less overfitting
- Faster training
- Simpler, more explainable models

Contrast with [[Feature Extraction - PCA]], which creates new columns. Selection keeps a subset of originals — so features remain interpretable.

## Method 1 — Drop obvious junk manually

```python
df = df.drop(columns=['row_id', 'timestamp_created', 'internal_ref'])
```

## Method 2 — Drop highly correlated columns

If two columns correlate > 0.95, they carry almost the same information. Keep one, drop the other.

```python
import numpy as np

corr = df.corr().abs()
upper = corr.where(np.triu(np.ones(corr.shape), k=1).astype(bool))
to_drop = [col for col in upper.columns if upper[col].max() > 0.95]
df = df.drop(columns=to_drop)
```

## Method 3 — Mutual information score

Measures how much each feature tells the model about the target. Works for both linear and non-linear relationships.

```python
from sklearn.feature_selection import mutual_info_classif
import pandas as pd

scores = mutual_info_classif(X_train, y_train)
feat_scores = pd.Series(scores, index=X_train.columns).sort_values(ascending=False)
print(feat_scores)
# credit_score    0.92  ← very useful
# income          0.78
# age             0.61
# city            0.34
# row_id          0.01  ← drop this
```

## Method 4 — Feature importance from a tree model

```python
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(n_estimators=100, random_state=42)
rf.fit(X_train, y_train)

importances = pd.Series(rf.feature_importances_, index=X_train.columns)
print(importances.sort_values(ascending=False))
```

## ⚠️ Key rule

Run feature selection using **training data only**. Never look at test data to decide which features to keep — that is [[Data Leakage]].

## Related
- [[Feature Extraction - PCA]]
- [[Feature Creation]]
- [[Data Leakage]]
