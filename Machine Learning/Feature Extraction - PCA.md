---
tags: [ml, preprocessing, feature-engineering]
---

# Feature Extraction — PCA

Instead of selecting original columns (see [[Feature Selection]]), extraction creates brand new columns that are mathematical combinations of the originals. The goal: represent the same information in fewer dimensions.

**PCA (Principal Component Analysis)** finds the directions in your data with the most variation and projects everything onto those directions.

Think of it like this: 100 slightly different measurement columns → 10 new "principal components" that together capture 90% of the original information.

## When to use PCA

- You have many columns (dozens to hundreds) and training is slow
- You want to visualise high-dimensional data in 2D or 3D
- Interpretability doesn't matter (you don't need to explain which features the model uses)

## When NOT to use PCA

- You need to explain the model (e.g. banking, healthcare, regulation) — PCA components are uninterpretable
- Your dataset is small

## Code

```python
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler

# Step 1: MUST scale first — PCA is sensitive to feature magnitude
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)

# Step 2: Compress to 10 components
pca = PCA(n_components=10)
X_pca = pca.fit_transform(X_scaled)

# Step 3: Check how much variance each component explains
print(pca.explained_variance_ratio_.cumsum())
# [0.42, 0.61, 0.74, 0.84, 0.90, ...]
# → First 5 components capture 90% of the information

# Or: let PCA decide how many components to keep
pca_auto = PCA(n_components=0.90)  # keep enough for 90% variance
X_pca_auto = pca_auto.fit_transform(X_scaled)
print(f"Components kept: {pca_auto.n_components_}")
```

## ⚠️ Key gotcha

PCA components are NOT interpretable. You lose the ability to say "the model heavily uses credit_score." For anything requiring explainability, use [[Feature Selection]] instead.

## Other extraction methods

| Method | Best for |
|---|---|
| PCA | Linear compression, general use |
| t-SNE / UMAP | Visualisation only — do NOT use as model features |
| Autoencoder | Non-linear structure, more powerful but slower |
| SVD / NMF | NLP, recommendation systems |

## Related
- [[Feature Selection]]
- [[Scaling Numeric Features]]
