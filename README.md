# Online Retail Analytics — Data Cleaning & Transformation Pipeline

A comprehensive data cleaning, auditing, and feature engineering pipeline for the **Online Retail** dataset using **Python, Pandas, and Regular Expressions**.

This project processes raw transactional retail data (541,909 entries) into a verified, analysis-ready dataset by systematically eliminating duplicates, handling missing data, segregating anomalous returns/damaged goods, standardizing date-time fields, sanitizing customer identifiers, and engineering revenue metrics.

---

## 📊 Dataset Overview

| Metric | Raw Dataset | Cleaned Dataset (`cleaned_data.csv`) |
| :--- | :--- | :--- |
| **Total Rows** | 541,909 | **525,373** |
| **Total Columns** | 8 | **10** (added `InvoiceTime`, `TotalCost`) |
| **Duplicates** | 5,268 rows | **0** |
| **Missing Descriptions** | 1,454 rows | **0** (dropped) |
| **Missing Customer IDs** | 135,037 rows | **0** (filled as `'Not Available'`) |
| **Rows Removed** | — | **9,814 rows** (invalid/return/negative/duplicate) |

---

## 🧹 Step-by-Step Cleaning & Transformation

Based on the audit report and workflow in `online_retail.ipynb`:

### 1. Duplicate Rows Removal
* **Issue:** 5,268 records were exact duplicates across all 8 columns.
* **Action:** Removed using `df.drop_duplicates()`, retaining 536,641 unique records.

### 2. Handling Missing Values
* **`Description` (1,454 nulls):**
  * *Reasoning:* Items missing descriptions cannot be classified or analyzed for product performance (<0.3% of the dataset).
  * *Action:* Dropped using `df.dropna(subset=['Description'])`.
* **`CustomerID` (135,037 nulls):**
  * *Reasoning:* Dropping missing customer IDs would discard ~25% of total sales revenue from guest checkouts.
  * *Action:* Converted column to `object` and imputed missing values with the category `'Not Available'`.

### 3. Invoice Number (`InvoiceNo`) Audit & Pattern Validation
* **Findings:**
  * 510,741 duplicate invoice IDs were observed. These are legitimate because an invoice groups multiple line items in a single transaction.
  * Regex pattern audit revealed two distinct format classes:
    1. **Standard purchases (6 numeric digits, e.g., `536365`):** 525,933 records.
    2. **Cancellations / Adjustments (7 characters starting with a letter, e.g., `C536365`):** 9,254 records.
  * No placeholder text (such as `"n/a"`, `"null"`, `"none"`) was present.
* **Action:** No records altered; structural validity verified.

### 4. Stock Code (`StockCode`) Audit
* **Findings:**
  * Contains standard product codes (e.g., `85123A`) and service/special adjustment codes (`POST`, `DOT`, `M`, `BANK CHARGES`, `AMAZONFEE`, `D`).
* **Action:** Retained as valid domain-specific transaction identifiers.

### 5. Description & Return/Damaged Product Segregation
* **Issue:** Negative quantities occurred for genuine customer returns, but also for bookkeeping anomalies, test orders, and damages (e.g. `'check'`, `'damages'`, `'Discount'`, `'lost'`, `'wet boxes'`, `'????damages????'`).
* **Multi-Step Resolution:**
  1. **Product Aggregate Quantity:** Computed net cumulative quantity for each of the 4,223 unique descriptions and exported to `data/clean/total_quantity.csv`.
  2. **Audit Negative Products:** Isolated 160 products with a net negative aggregate quantity into `data/clean/return_data.csv` for independent returns analysis.
  3. **Cleaned Active Sales:** Retained only products with net positive total quantities and transactions with individual `Quantity > 0`.

### 6. Quantity Validation
* **Audit:** Verified for fractional/decimal quantities (`Quantity % 1 != 0`).
* **Result:** 0 decimal values found; all quantities are clean discrete integers.

### 7. Date & Time Separation (`InvoiceDate`)
* **Issue:** Excel date-time values were combined into single timestamps (e.g., `2010-12-01 08:26:00`).
* **Action:**
  * Extracted time of purchase into a separate column: `df_cleaned['InvoiceTime'] = df_cleaned['InvoiceDate'].dt.time`
  * Formatted date to standard string convention: `df_cleaned['InvoiceDate'] = df_cleaned['InvoiceDate'].dt.strftime('%d-%m-%Y')`

### 8. Unit Price (`UnitPrice`) Sanitization
* **Issue:** Negative prices represent accounting debt adjustments or erroneous entries.
* **Action:** Filtered out negative prices using `df_cleaned = df_cleaned[~df_cleaned['UnitPrice'].astype(str).str.match(r'^[-]')]`.

### 9. Feature Engineering: `TotalCost`
* **Action:** Computed total transaction value per line item:
  $$\text{TotalCost} = \text{Quantity} \times \text{UnitPrice}$$

### 10. Customer ID (`CustomerID`) Sanitization
* **Issue:** Because raw customer IDs contained missing values, Pandas initially parsed them as floating-point numbers (`17850.0`).
* **Action:** Stripped trailing `.0` using regular expressions:
  ```python
  df_cleaned['CustomerID'] = (
      df_cleaned['CustomerID'].astype(str)
      .str.replace(r"\.[0-9]+$", "", regex=True)
  )
  ```
  Resulting IDs are clean integer strings (`'17850'`) while preserving `'Not Available'`.

---

## 📁 Repository & Data Structure

```text
ONLINE RETAIL/
│
├── data/
│   ├── raw/
│   │   └── retail.xlsx                 # Original raw dataset (541,909 rows)
│   │
│   └── clean/
│       ├── cleaned_data.csv            # Cleaned & transformed dataset (525,373 rows)
│       ├── return_data.csv             # Segregated returns & damaged items (160 items)
│       └── total_quantity.csv          # Aggregate sales volume per product description
│
├── online_retail.ipynb                 # Jupyter Notebook containing full code and execution outputs
└── README.md                           # Project documentation & cleaning report
```

---

## 🛠️ Tech Stack
* **Language:** Python 3.14
* **Libraries:**
  * `pandas` — Data wrangling, type transformations, aggregations
  * `numpy` — Numeric operations & vectorization
  * `matplotlib` — Data inspection and visualization
  * `re` — Regular expression pattern matching & data audit

---

## 🚀 Key Takeaways for Downstream Analytics
1. **Clean Baseline for RFM Analysis:** The segregation of guest users (`'Not Available'`) allows precise customer retention and RFM scoring without losing general revenue trends.
2. **Returns Segregated:** Return and damaged goods data are kept in a separate file (`return_data.csv`), ensuring core sales analysis is not skewed by negative quantities.
3. **Temporal Split:** Dedicated `InvoiceDate` and `InvoiceTime` columns facilitate peak shopping hour discovery and time-series seasonality modeling.
