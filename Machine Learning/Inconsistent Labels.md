---
tags: [ml, preprocessing, data-cleaning]
---

# Inconsistent Labels

The same category appears spelled or cased differently across rows. The model treats each unique string as a separate category — so "F", "Female", and "female" become three classes when they should be one.

This silently inflates the number of categories and produces wrong encodings.

## Audit first

```python
# Always run this on every categorical column before encoding
print(df['gender'].value_counts())
# M       3
# F       1
# Female  1   ← same as F but different string
# female  1   ← same again
```

## Fix 1 — Normalise case and whitespace

```python
df['city'] = df['city'].str.strip().str.title()
# "bengaluru " → "Bengaluru"
# "MUMBAI"     → "Mumbai"
```

## Fix 2 — Explicit mapping dictionary

```python
gender_map = {
    'F': 'F', 'Female': 'F', 'female': 'F', 'f': 'F',
    'M': 'M', 'Male':   'M', 'male':   'M', 'm': 'M',
}
df['gender'] = df['gender'].map(gender_map)

# Any value not in the map becomes NaN — check for leftovers
print(df['gender'].isnull().sum())
```

## Fix 3 — Replace specific values

```python
df['gender'] = df['gender'].replace({'Female': 'F', 'Male': 'M'})
```

## Habit to build

Run `df[col].value_counts()` on every categorical column immediately after loading data. This is where the most common and surprising problems hide.

## Related
- [[Raw Data]]
- [[Encoding Categorical Variables]]
- [[Duplicates]]
