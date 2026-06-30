---
tags: [ml, preprocessing, encoding]
---

# Encoding Categorical Variables

ML models work with numbers, not strings. Encoding converts text categories ("Bengaluru", "F", "High") into numeric form.

Fix [[Inconsistent Labels]] before encoding.

---

## One-hot encoding (most common)

Creates a new binary (0/1) column for each unique category value.

Use when: categories have **no natural order** (city, gender, colour).

```python
# With pandas
df_encoded = pd.get_dummies(df, columns=['city', 'gender'])
# city_Bengaluru  city_Mumbai  city_Delhi  gender_F  gender_M
#       1               0           0          0         1
#       0               1           0          1         0

# With scikit-learn (preferred — fits on train, transforms test safely)
from sklearn.preprocessing import OneHotEncoder

ohe = OneHotEncoder(sparse_output=False, handle_unknown='ignore')
# handle_unknown='ignore' → unseen categories in test become all zeros

ohe.fit(X_train[['city', 'gender']])
encoded_train = ohe.transform(X_train[['city', 'gender']])
encoded_test  = ohe.transform(X_test[['city', 'gender']])
```

### ⚠️ Dummy variable trap (linear models only)

With linear models, drop one category to avoid multicollinearity. The dropped category becomes the "baseline."

```python
pd.get_dummies(df, columns=['city'], drop_first=True)
```

---

## Ordinal / label encoding

Assigns an integer to each category: Low → 0, Medium → 1, High → 2.

Use when: categories have a **natural order**.

```python
from sklearn.preprocessing import OrdinalEncoder

oe = OrdinalEncoder(categories=[['Low', 'Medium', 'High']])
df[['risk_encoded']] = oe.fit_transform(df[['risk_level']])
```

**Do NOT use label encoding for unordered categories.** The model will interpret the integers as implying an order that doesn't exist (e.g. "Delhi=2 > Mumbai=1" is meaningless).

---

## Target encoding

Replace each category with the mean of the target variable for that category.

Use when: column has **many unique values** (50+ cities) — one-hot would create too many columns.

```python
# Compute on training data ONLY
city_means = y_train.groupby(X_train['city']).mean()
X_train['city_enc'] = X_train['city'].map(city_means)
X_test['city_enc']  = X_test['city'].map(city_means)
# Fill unseen cities with global mean
X_test['city_enc']  = X_test['city_enc'].fillna(city_means.mean())
```

See [[Data Leakage]] — computing this on the full dataset leaks label information.

---

## Frequency encoding

Replace each category with how often it appears.

```python
freq = df['city'].value_counts(normalize=True)
df['city_freq'] = df['city'].map(freq)
```

---

## Which encoding to use

| Situation | Encoding |
|---|---|
| Nominal, few categories (<15) | One-hot |
| Ordered categories (Low/Med/High) | Ordinal |
| Nominal, many categories (50+) | Target or Frequency |
| Tree-based model | Any — trees don't care about numeric order |
| Linear model / neural net | One-hot (or target for high cardinality) |

## Related
- [[Inconsistent Labels]]
- [[Feature Creation]]
- [[Scaling Numeric Features]]
- [[Data Leakage]]
