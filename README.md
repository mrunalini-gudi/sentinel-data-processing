# SENTINEL Data Processing

Data loading, quality checks, leakage-free feature engineering and reason codes for **Project SENTINEL**, PayVista's real-time payment fraud system.

The notebook [`SENTINEL_Data_Processing.ipynb`](SENTINEL_Data_Processing.ipynb) covers the five tasks of the project brief:

| # | Task | Output |
|---|---|---|
| 1 | Load customers, merchants, devices, transactions, payment_outcomes and labels into pandas | one DataFrame per table, read from Supabase page by page |
| 2 | Data quality report | nulls per column, duplicates, wrong types, outliers in amount, row count per split |
| 3 | Feature engineering, using only data from before each transaction | `amount_to_average_ratio`, `is_new_device`, `customer_merchant_distance_km`, `is_night`, `transaction_count_last_1_hour`, `transaction_amount_last_24_hours` |
| 4 | Reason codes | R01–R08 with the thresholds from class |
| 5 | Validation | match rate per feature against the provided `features` table, with every mismatch explained |

## Results

Run on the complete SENTINEL dataset (1,100,000 transactions, including the warmup split):

| Feature | Match rate with the provided table |
|---|---|
| amount_to_average_ratio | 100% |
| is_new_device | 100% |
| customer_merchant_distance_km | 100% |
| is_night | 100% |
| transaction_count_last_1_hour | 100% |
| transaction_amount_last_24_hours | 100% |
| failed attempts in last 24 hours (helper for R04) | 99.9999% (1 row: a failed outcome exactly on the 24-hour boundary) |

**Reason codes.** R01, R03, R04 and R05 match the provided `reason_codes`. R02, R06, R07 and R08 differ only because the provided table uses different thresholds from the class values:

| Code | Class threshold | Provided table |
|---|---|---|
| R02 GEO_MISMATCH | distance > 500 km | distance > 500 km **or a new city** |
| R06 NEW_ACCOUNT | account < 7 days | account < **30** days |
| R07 VELOCITY | > 5 transactions in 60 min | > **3** transactions in 60 min |
| R08 HIGH_RISK_MCC | travel, gaming, electronics | **gift_cards**, gaming, electronics |

Re-running the codes with the provided thresholds gives a 100% match for every code, so all differences are threshold choices, not calculation errors.

**Key data-quality findings**

- `customer_location` is null in 3.0% of rows, `average_transaction_value` in 2.6% and `device_id` in 1.0%. These rows are kept, and the affected features are treated as unknown.
- No duplicate primary keys and no fully duplicated rows.
- Timestamps and JSON locations arrive as text and are converted before feature engineering. Times are stored in UTC, but night-time is computed in IST.
- Amounts are heavily skewed: ₹2.17 to ₹79.9 lakh, with a median of ₹1,251.
- Splits: warmup 100,000 · train 714,745 · validation 144,803 · holdout 140,452.

> With a partial load or a sample, some customers' earlier transactions are missing, so a few feature mismatches are expected. Section 5 labels each one ("earlier history missing from the loaded data").

## Repository structure

```
sentinel-data-processing/
├── SENTINEL_Data_Processing.ipynb   # the analysis (sections 1–5)
├── requirements.txt                 # Python packages
├── .env.example                     # template for your own .env (no secrets)
├── .gitignore                       # keeps .env and data files out of Git
├── screenshots/                     # output screenshots used below
└── README.md
```

## Setup

1. **Clone the repository**

   ```bash
   git clone https://github.com/<your-username>/sentinel-data-processing.git
   cd sentinel-data-processing
   ```

2. **Create and activate a virtual environment** (Python 3.11 or newer)

   ```bash
   python -m venv sentinelenv
   sentinelenv\Scripts\activate        # Windows
   source sentinelenv/bin/activate     # macOS / Linux
   ```

3. **Install the packages**

   ```bash
   pip install -r requirements.txt
   ```

4. **Add your Supabase credentials.** Copy the template and fill in your project URL and key (Supabase → Project Settings → API):

   ```bash
   copy .env.example .env              # Windows
   cp .env.example .env                # macOS / Linux
   ```

   `.env` is listed in `.gitignore`, so your key is never committed.

5. **Load the SENTINEL data into Supabase** with the class scripts from Project Work 4 and 5 (`sql/create_table` then `python ingest.py all`). The notebook reads the tables `customers`, `merchants`, `devices`, `transactions`, `payment_outcomes`, `labels` and `features`.

   *Alternative:* to read the dataset's CSV files directly, set `SENTINEL_DATA_SOURCE=local` and `SENTINEL_DATA_DIR=<path to sentinel_data>` in `.env`.

## How to run

1. Open `SENTINEL_Data_Processing.ipynb` in VS Code or Jupyter and select the `sentinelenv` kernel.
2. Choose **Run All**.
3. Optional settings in the *Configuration* cell:
   - `MAX_ROWS_PER_TABLE`: limits rows per table for a quick test.
   - `SAMPLE_CUSTOMERS`: keeps N random customers with **all** their transactions. Sampling by customer keeps each customer's history complete, so the time-window features stay correct.

Runtime on the full 1.1M-row dataset from local files is about 2 minutes. Loading from Supabase takes longer, because tables are read 1,000 rows per request.

## Screenshots

### 1. Supabase connection and table load
![Supabase connection](screenshots/01_connection_and_load.png)

### 2. Data quality summary
![Data quality summary](screenshots/02_data_quality_summary.png)

### 3. Outliers in amount and rows per split
![Outliers and splits](screenshots/03_outliers_and_splits.png)

### 4. Engineered features and the no-leakage spot check
![Features](screenshots/04_features_and_leakage_check.png)

### 5. Reason codes and fraud rate per code
![Reason codes](screenshots/05_reason_codes.png)

### 6. Match rate per feature
![Feature match rate](screenshots/06_feature_match_rate.png)

### 7. Reason-code comparison and threshold differences
![Reason code comparison](screenshots/07_reason_code_comparison.png)

## Security

- Keys live only in `.env`, which Git ignores. `.env.example` contains empty placeholders only.
- The notebook reads with the key from the environment and only runs read-only queries.
- Data files (`*.csv.gz`, `*.parquet`, `*.jsonl`) are excluded by `.gitignore`. The dataset is synthetic, but it is still not published.
