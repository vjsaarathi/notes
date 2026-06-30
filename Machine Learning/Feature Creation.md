---
tags: [ml, preprocessing, feature-engineering]
---

# Feature Creation

Combining or deriving new columns from existing ones using domain knowledge.

> This is the **highest-leverage activity in applied ML**. A well-chosen feature can outperform 10 raw columns — because it encodes a relationship a domain expert already knows is meaningful.

## Datetime features

```python
df['ts'] = pd.to_datetime(df['ts'])

df['hour']        = df['ts'].dt.hour
df['day_of_week'] = df['ts'].dt.dayofweek   # 0=Monday, 6=Sunday
df['month']       = df['ts'].dt.month
df['is_weekend']  = df['ts'].dt.dayofweek.isin([5, 6]).astype(int)
df['days_since']  = (pd.Timestamp.now() - df['ts']).dt.days
```

## Ratio features (often very powerful)

```python
# Debt-to-income: far more meaningful than raw loan amount alone
df['debt_to_income'] = df['loan_amount'] / df['income'].replace(0, np.nan)
# .replace(0, np.nan) prevents division by zero

df['income_per_dependent'] = df['income'] / (df['num_dependents'] + 1)
```

## Interaction terms

```python
# Multiply two columns to capture their combined effect
df['age_x_income'] = df['age'] * df['income']

# Polynomial features — all pairs + squares (useful for linear models)
from sklearn.preprocessing import PolynomialFeatures

poly = PolynomialFeatures(degree=2, include_bias=False)
X_poly = poly.fit_transform(df[['age', 'income']])
# Creates: age, income, age², income², age×income
```

## Target encoding

Replace each category with the mean of the target for that category. Useful for high-cardinality columns (many unique values).

```python
# Compute on training data ONLY, then map to test
city_means = y_train.groupby(X_train['city']).mean()

X_train['city_approval_rate'] = X_train['city'].map(city_means)
X_test['city_approval_rate']  = X_test['city'].map(city_means)

# Handle unseen cities in test set
X_test['city_approval_rate'] = X_test['city_approval_rate'].fillna(city_means.mean())
```

## Aggregation features

```python
# How many transactions did each customer make?
tx_counts = df.groupby('customer_id')['transaction_id'].count().rename('tx_count')
df = df.merge(tx_counts, on='customer_id', how='left')
```

## ⚠️ Target encoding leakage

If you compute group means using the full dataset (including test rows), you leak label information. Always compute on training data only. See [[Data Leakage]].

## Related
- [[Feature Selection]]
- [[Encoding Categorical Variables]]
- [[Data Leakage]]
