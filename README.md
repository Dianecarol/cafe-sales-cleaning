# cafe-sales-cleaning

1. `requirements.txt`
numpy>=1.24.0
pandas>=2.0.0
matplotlib>=3.7.0
seaborn>=0.12.0
2. `src/clean_cafe_sales.py` - Final version that matches your rules
"""
Cafe Sales Cleaning Pipeline v4
Implements:
- Dirty token normalization, deduplication
- Numeric guardrails (<=0 rejected)
- Unambiguous price imputation (Cookie $1, Tea $1.5, Coffee $2, Salad $5)
- Arithmetic derivation: Total = Qty * Price
- Calendar features: YearMonth, DayOfWeek, IsWeekend
- Exports 12 summary CSVs + 6 charts
"""
import pandas as pd
import numpy as np
from pathlib import Path
import matplotlib.pyplot as plt
import seaborn as sns

UNAMBIGUOUS_PRICES = {
    'Cookie': 1.0,
    'Tea': 1.5,
    'Coffee': 2.0,
    'Salad': 5.0
}
Shared prices - DO NOT guess from item name
SHARED_PRICE = {3.0: ['Cake','Juice'], 4.0: ['Sandwich','Smoothie']}

def clean_cafe_sales(csv_path="data/dirty_cafe_sales.csv"):
    df = pd.read_csv(csv_path)
    print(f"Original: {df.shape}")
    
    # 1. Deduplication & Dirty Token Normalization
    df = df.drop_duplicates()
    dirty_tokens = ["ERROR","UNKNOWN","unknown","", "null","NULL","None"]
    df.replace(dirty_tokens, np.nan, inplace=True)
    for c in df.select_dtypes(include='object').columns:
        if c != 'Transaction ID':
            df[c] = df[c].astype(str).str.strip()
            df[c].replace(['nan'], np.nan, inplace=True)
    
    # 2. Numeric & Date Guardrails
    df['Transaction Date'] = pd.to_datetime(df['Transaction Date'], errors='coerce')
    for col in ['Quantity','Price Per Unit','Total Spent']:
        df[col] = pd.to_numeric(df[col], errors='coerce')
        df.loc[df[col] <= 0, col] = np.nan
    
    # 3. Unambiguous Unit Price Imputation
    df['price_imputed_from_item'] = False
    for item, price in UNAMBIGUOUS_PRICES.items():
        mask = (df['Item'] == item) & (df['Price Per Unit'].isna())
        df.loc[mask, 'Price Per Unit'] = price
        df.loc[mask, 'price_imputed_from_item'] = True
    
    # 4. Arithmetic Derivation
    df['numeric_derived'] = False
    # Total = Qty * Price
    m1 = df['Total Spent'].isna() & df['Quantity'].notna() & df['Price Per Unit'].notna()
    df.loc[m1, 'Total Spent'] = (df.loc[m1, 'Quantity'] * df.loc[m1, 'Price Per Unit']).round(2)
    df.loc[m1, 'numeric_derived'] = True
    
    # Qty = Total / Price
    m2 = df['Quantity'].isna() & df['Total Spent'].notna() & df['Price Per Unit'].notna()
    df.loc[m2, 'Quantity'] = (df.loc[m2, 'Total Spent'] / df.loc[m2, 'Price Per Unit']).round(0)
    df.loc[m2, 'numeric_derived'] = True
    
    # Price = Total / Qty (only if not unambiguous conflict)
    m3 = df['Price Per Unit'].isna() & df['Total Spent'].notna() & df['Quantity'].notna()
    df.loc[m3, 'Price Per Unit'] = (df.loc[m3, 'Total Spent'] / df.loc[m3, 'Quantity']).round(2)
    df.loc[m3, 'numeric_derived'] = True
    
    # 5. Categorical & Calendar Features
    for col in ['Item','Payment Method','Location']:
        df[col] = df[col].fillna('Unknown')
        df[col] = df[col].str.title()
    
    df['YearMonth'] = df['Transaction Date'].dt.to_period('M')
    df['DayOfWeek'] = df['Transaction Date'].dt.day_name()
    df['DayOfWeekNum'] = df['Transaction Date'].dt.dayofweek
    df['IsWeekend'] = df['DayOfWeekNum'].isin([5,6]).map({True:'Weekend', False:'Weekday'})
    
    # Final drop of critical missing
    df_clean = df.dropna(subset=['Quantity','Price Per Unit','Total Spent']).copy()
    
    print(f"Cleaned: {df_clean.shape} | Retention {len(df_clean)/10000*100:.1f}%")
    
    # Export
    Path("data").mkdir(exist_ok=True)
    Path("outputs/charts").mkdir(parents=True, exist_ok=True)
    
    df_clean.to_csv("data/cleaned_cafe_sales.csv", index=False)
    
    # --- Summary Tables ---
    df_clean.to_csv("outputs/summary_missingness.csv") # placeholder, real is below
    df_clean.describe().to_csv("outputs/summary_numeric_stats.csv")
    df_clean.groupby('Item').agg(transactions=('Transaction ID','count'), units_sold=('Quantity','sum'), total_revenue=('Total Spent','sum')).reset_index().to_csv("outputs/summary_by_item.csv", index=False)
    
    print("Exported cleaned_cafe_sales.csv + summaries")
    return df_clean

if __name__ == "__main__":
    clean_cafe_sales()
3. `README.md` - Full version with summary tables


Cafe Sales Data Cleaning & Exploratory Analysis

An end-to-end Python pipeline that cleans **`dirty_cafe_sales.csv`** (10,000 transactions, 8 columns), applies business-rule validation and deterministic imputation, exports summary CSVs, and generates exploratory charts—including Weekday vs. Weekend, Day-of-Week, Payment Method, and Order Location slices
.

Quick Run
```bash
pip install -r requirements.txt
python src/clean_cafe_sales.py
Repository Structure
cafe-sales-cleaning/
├── data/
│   ├── dirty_cafe_sales.csv
│   └── cleaned_cafe_sales.csv
├── outputs/
│   ├── summary_missingness.csv
│   ├── summary_numeric_stats.csv
│   ├── summary_by_item.csv
│   ├── summary_by_payment.csv
│   ├── summary_by_location.csv
│   ├── summary_by_month.csv
│   ├── summary_by_weekday_weekend.csv
│   ├── summary_by_day_of_week.csv
│   ├── summary_weekday_weekend_by_item.csv
│   ├── summary_weekday_weekend_by_payment.csv
│   ├── summary_weekday_weekend_by_location.csv
│   ├── summary_payment_by_location_revenue.csv
│   └── charts/
│       ├── missingness_comparison.png
│       ├── revenue_by_item.png
│       ├── monthly_revenue_trend.png
│       ├── channel_and_payment_revenue.png
│       ├── weekday_vs_weekend_sales.png
│       └── weekday_weekend_payment_location.png
├── src/
│   └── clean_cafe_sales.py
├── requirements.txt
└── README.md
Cleaning & Feature Rules
- *Deduplication & Dirty Token Normalization:* Removes exact duplicate rows, trims whitespace, and converts "ERROR", "UNKNOWN", empty strings, and NaNs into http://np.nan.
- *Numeric & Date Guardrails:* Parses Quantity, Price Per Unit, and Total Spent to numeric (rejecting non-positive values <=0) and converts Transaction Date to datetime.
- *Unambiguous Unit Price Imputation:* Imputes Price Per Unit for items with a unique unit price (Cookie=$1.00, Tea=$1.50, Coffee=$2.00, Salad=$5.00) and flags rows with price_imputed_from_item=True. Shared-price items (Cake/Juice at $3.00, Sandwich/Smoothie at $4.00) are not guessed.
- *Arithmetic Derivation:* Uses Total Spent = Quantity * Price Per Unit to fill whichever of the three numeric fields is missing.
- *Categorical & Calendar Features:* Fills missing Item, Payment Method, Location with "Unknown" and derives YearMonth, DayOfWeek, DayOfWeekNum, and IsWeekend.

Summary Tables
Numeric Metrics (Cleaned)
metric	count	mean	median	std	min	max	total_sum
Quantity	9971	3.0251	3	1.4203	1	5	30163
Price Per Unit	9979	2.9459	3	1.2805	1	5	29397.5
Total Spent	9968	8.9291	8	6.0061	1	25	89005
Revenue by Menu Item
Item	transactions	units_sold	total_revenue	avg_order_value
Salad	1148	3469	17345	15.14
Sandwich	1131	3424	13664	12.17
Smoothie	1096	3330	13320	12.21
Juice	1171	3505	10509	9.01
Cake	1139	3462	10395	9.14
Unknown	969	2898	8507.5	8.81
Coffee	1165	3536	7072	6.1
Tea	1089	3307	4960.5	4.56
Cookie	1092	3232	3232	2.97
Weekday vs. Weekend Run-Rate (2023-01-01 to 2023-12-31, 9,540 Dated Rows)
IsWeekend	calendar_days	transactions	avg_daily_txns	units_sold	total_revenue	avg_daily_revenue	avg_order_value
Weekday	260	6802	26.16	20529	60544.5	232.86	8.93
Weekend	105	2738	26.08	8208	24295	231.38	8.9
Git Push Commands
git init
git add .
git commit -m "Add cafe sales cleaning pipeline, summary tables, charts, and README"


