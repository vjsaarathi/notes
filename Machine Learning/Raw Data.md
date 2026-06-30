---
tags: [ml, preprocessing]
---

# Raw Data

What a dataset typically looks like the moment you load it — before any cleaning.

## First thing to always run

```python
print(df.shape)           # (rows, columns)
print(df.dtypes)          # data type of each column
print(df.isnull().sum())  # missing value count per column
print(df.describe())      # mean, std, min, max for numeric columns
print(df.nunique())       # unique value count — helps identify categoricals
```

## Common column types

| Type | Example columns | Notes |
|---|---|---|
| Numeric | age, income, credit_score | Can be fed to models after scaling |
| Categorical | city, gender | Must be encoded into numbers |
| Target / label | loan_approved (0/1) | What the model is trying to predict |
| Datetime | transaction_ts | Must be decomposed into numeric features |

## Problems raw data almost always has

| Problem | Example | Effect |
|---|---|---|
| [[Missing Values\|Missing values]] | `age = NaN` | Model crashes or silently wrong |
| [[Outliers]] | `income = 9999999` | Distorts statistics and scaling |
| [[Duplicates]] | Row 5 = Row 1 exactly | Model overtrained on that example |
| [[Inconsistent Labels]] | "F" vs "Female" in same column | Treated as different categories |

## Related
- [[What is ML Preprocessing]]
- [[Missing Values]]
- [[Outliers]]
- [[Duplicates]]
- [[Inconsistent Labels]]
