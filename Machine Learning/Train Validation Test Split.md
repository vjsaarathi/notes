---
tags: [ml, preprocessing]
---

# Train / Validation / Test Split

You cannot evaluate your model on data it learned from — it has already "seen" that data and will appear to perform better than it actually is. Splitting keeps honest evaluation data completely separate.

## The three sets

| Set | Purpose | Typical size |
|---|---|---|
| **Training** | Model learns from this | 70–80% |
| **Validation** | Tune hyperparameters, compare models | 10–15% |
| **Test** | Final honest evaluation — touch only once | 10–15% |

## Code

```python
from sklearn.model_selection import train_test_split

# Step 1: Split off the test set first — set it aside and don't touch it
X_trainval, X_test, y_trainval, y_test = train_test_split(
    X, y,
    test_size=0.15,
    random_state=42,
    stratify=y         # preserves class ratio in each split
)

# Step 2: Split the rest into train and validation
X_train, X_val, y_train, y_val = train_test_split(
    X_trainval, y_trainval,
    test_size=0.15/0.85,
    random_state=42,
    stratify=y_trainval
)

print(f"Train: {len(X_train)}, Val: {len(X_val)}, Test: {len(X_test)}")
```

## `stratify=y` — always use for classification

Ensures the proportion of each class is the same in all three splits. Without it, by chance your test set might have very few examples of one class.

```python
# Example: 30% approved, 70% not approved
# stratify=y guarantees this 30/70 ratio is preserved in train, val, and test
```

## K-fold cross-validation (for small datasets)

When you don't have enough data for a dedicated validation set, k-fold trains the model k times, each time using a different fold as the validation set. The final score is the average.

```python
from sklearn.model_selection import cross_val_score
from sklearn.linear_model import LogisticRegression

model = LogisticRegression()
scores = cross_val_score(model, X_train, y_train, cv=5, scoring='accuracy')

print(f"Scores per fold: {scores}")
print(f"Mean: {scores.mean():.3f} ± {scores.std():.3f}")
```

With 5-fold: your training data is split into 5 equal parts. The model trains on 4, validates on 1, repeats 5 times rotating the validation fold.

## ⚠️ Critical: split BEFORE fitting anything

Scalers, imputers, encoders must be fitted on training data only. The split must happen first. See [[Data Leakage]].

## Related
- [[Data Leakage]]
- [[Preprocessing Pipeline - Correct Order]]
