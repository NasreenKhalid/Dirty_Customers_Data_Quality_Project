# Data Quality Assessment: Dirty Customer Dataset

A data quality assessment of a deliberately messy customer dataset (1,000 rows, 8 columns), done in Python (pandas). The aim was to practise real analyst work: **find the problems, quantify them, document the assumptions, and recommend next steps, before changing any data.**

> **Scope:** This project stops at assessment and recommendations. **No cleaning has been applied.** Several fixes (which duplicate record is correct, the intended date order, whether `$` and `USD` are the same currency) need a decision from the data owner first, so the report lists them as open questions.

## Contents

| File | Description |
|---|---|
| `report/Data_Quality_Report.pdf` | Full report: summary, findings table, detailed findings, assumptions, limitations, recommendations |
| `carousel/Data_Quality_Assessment_LinkedIn_Carousel.pdf` | 5-slide visual summary |
| `notebooks/data_quality_profiling.ipynb` | Profiling code (add your notebook here) |
| `data/dirty-customers.csv` | The dataset (see *Data source* below) |

## The dataset

| Column | Content |
|---|---|
| `customer_id` | Customer identifier (`CUST-1387` style) |
| `full_name` | Customer name |
| `email` | Email address |
| `signup_date` | Signup date, stored as text |
| `country` | Country (many spellings) |
| `spend` | Spend amount, stored as text |
| `segment` | Customer segment |
| `is_active` | Active status (true/false in many forms) |

Signups run from **1 January 2023 to 19 June 2025**.

**Data source:** [add where the dataset came from, and its licence if it has one]

## Key findings

Every column had at least one issue. Overall verdict: **poor**, since the data isn't ready for analysis without cleaning.

| Area | Finding | Count / % of 1,000 |
|---|---|---|
| Missing values | `signup_date` missing or placeholder | 54 (5.4%) |
| | `spend` missing or placeholder | 62 (6.2%) |
| | `email` blank | 31 (3.1%) |
| Placeholders | "Missing" written 7 ways: `NULL`, `none`, `N/A`, `n/a`, `-`, `-999`, blank | across `signup_date` and `spend` |
| Date formats | 5 formats in `signup_date`; not ISO | 756 (75.6%) |
| Spend formats | `$` prefix 205, `USD ` prefix 203 | 408 (40.8%) |
| Emails | Malformed (`[at]`, `@@`, `Name <email>` ...) | 63 (6.3%) |
| Duplicates | Exact duplicate rows | 1 (0.1%) |
| | Customer IDs appearing more than once | 42 IDs |
| | Rows in repeated-ID groups | 85 (8.5%) |
| | Extra records beyond one per ID | 43 (4.3%) |
| Categories | `country`: 14 spellings, 5 countries | |
| | `segment`: 15 spellings, 4 categories | |
| | `is_active`: 10 spellings, 2 values | |
| IDs | 4-digit vs 8-digit format | 720 / 280 |
| Names | Stray spaces / all-upper or all-lower | 266 / 152 |
| Outliers | `spend` above the IQR upper fence (about 1,381) | about 150 of 938 numeric values (about 16%) |

## Approach

1. **Read everything as text** (`dtype=str, keep_default_na=False`) so nothing is silently converted.
2. **Profile the raw data without modifying it.** Every check writes to a separate variable. The raw `df` is never overwritten.
3. **Classify every value into exactly one category** and check that the categories sum to 1,000 rows.
4. **Quantify:** count and percentage for every issue.
5. **Judge severity** and write a proposed action for each issue.
6. **Document assumptions and limitations** that need the data owner to confirm.

```python
import pandas as pd

df = pd.read_csv("data/dirty-customers.csv", dtype=str, keep_default_na=False)
```

## Challenges and how I solved them

### 1. The file wouldn't import into MySQL
**Problem:** A `DATE` column rejects anything that isn't `YYYY-MM-DD`, and `signup_date` mixed five date formats with placeholder text.
**Solution:** Read the file in pandas as text first, and profile it there. (In MySQL, the alternative is a staging table with every column as `VARCHAR`.)

### 2. pandas silently hid some missing values
**Problem:** By default, `read_csv` turns strings like `NULL`, `N/A` and blank cells into real `NaN`. My value counts and `isnull()` checks skipped those rows, so my totals didn't add up.
**Solution:** `keep_default_na=False` keeps every placeholder as visible text, so each spelling can be counted. I also report the difference as a finding in itself.

### 3. My counts didn't reconcile
**Problem:** Categories of valid dates, nulls and "neither" didn't sum to 1,000. Causes: null-token lists that differed between cells, masks calculated before I reloaded the DataFrame, and case-sensitive matching (`N/A` vs `n/a` missed 7 rows).
**Solution:** One cell defines a single null list (compared in lowercase) and a single set of date formats, then assigns every row to exactly one of three categories: valid, missing/placeholder, or invalid. A final check confirms the three counts equal the row count.

```python
null_tokens = {"", "null", "none", "-", "-999", "n/a"}
is_missing = s.str.lower().isin(null_tokens)
# ... valid / invalid computed in the same cell ...
assert is_valid.sum() + is_missing.sum() + is_invalid.sum() == len(df)
```

### 4. A date-shaped string isn't always a valid date
**Problem:** A regex accepts `31.02.2024`, which isn't a real date.
**Solution:** A value counts as valid only if it also parses with its explicit format. In this dataset, no date-shaped value failed to parse. I record that as a checked result.

### 5. pandas misread ambiguous dates
**Problem:** `pd.to_datetime(format="mixed")` read dotted dates like `08.01.2023` month-first, which pushed the latest signup date to November 2025, when the correct value is 19 June 2025.
**Solution:** Dates with a number above 12 revealed the order in each format: dotted dates are day-first, slashed dates are month-first. I parsed each format group with its own explicit format, and listed the inference as an assumption to be confirmed.

### 6. `df.duplicated()` found only one duplicate
**Problem:** By default it compares every column and flags only later copies, so rows that differ only by casing were missed.
**Solution:** Four separate metrics, which reconcile with each other:

```python
df.duplicated().sum()                                  # exact duplicate rows
(df["customer_id"].value_counts() > 1).sum()           # IDs that repeat
df.duplicated("customer_id", keep=False).sum()         # rows in repeated groups
df.duplicated("customer_id").sum()                     # extra records
# rows in repeated groups - repeated IDs == extra records
# total rows - extra records == distinct IDs
```

### 7. Standardisation mappings needed checking
**Problem:** Mapping dictionaries had case-sensitive keys, a missing spelling, and one wrong assumption (`DE` mapped to Denmark, when it is the code for Germany).
**Solution:** Lowercase the data before mapping, then check that every spelling is mapped and the mapped counts total 1,000. Group sizes helped catch the error: Germany looked too small until `DE` was added.

```python
set(c.unique()) - set(country_map.keys())      # must be empty
c.map(country_map).value_counts().sum()        # must equal 1000
```

### 8. I accidentally overwrote a raw column
**Problem:** Converting `spend` to float during profiling broke the text-based checks.
**Solution:** Reload the raw file, and from then on assign to new variables only (`num = ...`). The raw data stays untouched, so "before" numbers stay trustworthy.

### 9. Outliers weren't clearly errors
**Problem:** The IQR rule flagged about 16% of spend values, and the maximum was around 171,623 against a median of about 252. That is too many to be a few typos.
**Solution:** I reported them as an open question for the data owner instead of removing them, and listed it under decisions to confirm.

### 10. Deciding where to stop
**Problem:** Cleaning involves judgment calls (which duplicate to keep, whether `$` and `USD` are the same currency).
**Solution:** I stopped at assessment on purpose. The report ends with assumptions to confirm and recommendations, which is how it would go to a manager or data owner before any change is made.

## What I learned

- Finding and explaining data problems is harder than fixing them.
- Every number should reconcile to the row count. If it doesn't, something is hiding.
- Library defaults can hide data issues. Check what your read function converts.
- Keep the raw data untouched while profiling.
- Write assumptions down so a reader can challenge them.

## Next steps

- Confirm the open questions with the data owner.
- Clean a copy of the data using the agreed rules, record every change, and complete the *Before vs After* table in the report.
- Add validation rules at data entry (dropdowns, date format, unique IDs).

## Tools

Python, pandas, regular expressions.

## Author

**Nasreen Khalid**: [LinkedIn profile link]
