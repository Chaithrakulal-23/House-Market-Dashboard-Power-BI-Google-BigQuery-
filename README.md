# House Market Dashboard (Power BI + Google BigQuery)
### Dashboard Link : https://app.powerbi.com/links/j9UYhfCoyO?ctid=51697115-1ecd-42b5-b509-2d62c3919f76&pbi_source=linkShare&bookmarkGuid=82d4961f-b5a6-45bb-b12e-d12c6db551b4
### Dashboard Pages:
- **House Market Overview**
- **Sales Performance**
- **House Type Analysis**

---

## Problem Statement

This dashboard helps analysts and stakeholders understand the Danish housing market. It provides insights into regional sales trends, house type comparisons, pricing dynamics, and year-on-year sales growth across different sale types. By identifying patterns in offer vs. purchase prices, SQM pricing, and market yield/interest rates, stakeholders can make informed investment and policy decisions.

Since auctions show the highest YOY growth (+0.29) while family sales show the steepest decline (-0.75), stakeholders can use these insights to rebalance their strategy. The dashboard also highlights that Jutland leads in median sales price growth, while Zealand dominates total sales volume at 95bn.

---

## Data Source

- **Platform:** Google BigQuery (Cloud)
- **Dataset:** Danish Housing Data (`Housing Data.csv`)
- **Table:** `alert-district-461908-q5.1.Housing`
- **Size:** 100K rows · 17.23 MB
- **Loaded:** 9 March 2026

---

## Steps Followed

### Step 1 — Set Up Google BigQuery

- Created a new project on Google Cloud Platform.
- Created a **Dataset** (`1`) and two **Tables** (`Housing`, `Test`) inside the project `alert-district-461908-q5`.
- Uploaded `Housing Data.csv` via the **Create Table** dialog (Source: Upload, Format: CSV, Schema: Auto-detect).

![Create Table - BigQuery](https://github.com/Chaithrakulal-23/House-Market-Dashboard-Power-BI-Google-BigQuery-/blob/c9038c2063856f2797460260fe229bb7aae2d0d4/create%20table2.png)


- Used the **Test** table as a sandbox copy of Housing for SQL exploration.

---

### Step 2 — SQL Data Exploration in BigQuery

Ran the following SQL queries in BigQuery Studio to understand and validate the data:

```sql
-- Preview all data
SELECT * FROM `alert-district-461908-q5.1.Test` LIMIT 1000;

-- Average purchase price by sales type
SELECT sales_type, avg(purchase_price)
FROM `alert-district-461908-q5.1.Test`
GROUP BY sales_type;

-- Update SQM where no_rooms = 3
UPDATE `alert-district-461908-q5.1.Test`
SET sqm = 100 WHERE no_rooms = 3;

-- Verify distinct SQM values
SELECT DISTINCT sqm FROM `alert-district-461908-q5.1.Test` WHERE no_rooms = 3;
```

![SQL Queries in BigQuery](https://github.com/Chaithrakulal-23/House-Market-Dashboard-Power-BI-Google-BigQuery-/blob/f512a5eeb2f39daf80a8ed627f661df3e6eb9594/runquery.png)

---

### Step 3 — Connect BigQuery to Power BI

- In Power BI Desktop, used **Get Data → Google BigQuery** to connect.
- Navigated to project `alert-district-461908-q5 → Dataset 1 → Housing` table.
- Selected the **Housing** table (100K rows) and clicked **Transform Data** to open Power Query Editor.

![BigQuery Navigator in Power BI](https://user-images.githubusercontent.com/102996550/loadbigquerrydstopw_.png)

---

### Step 4 — Data Cleaning in Power Query Editor

- Enabled **Column Distribution**, **Column Quality**, and **Column Profile** (based on entire dataset) under the View tab.
- Observed the following data quality issues:

| Column | Issue | Fix Applied |
|---|---|---|
| `city` | <1% empty values | Replaced `null` → `"Unknown"` |
| `dk_ann_infl_rate%` | <1% null values | Replaced `null` → `1.85` (most frequent value) |
| `yield_on_mortgage_credit_bonds%` | null values | Replaced `null` → `1.47` (most frequent value) |

- Applied three **Replaced Value** steps visible in the Applied Steps panel.

![Power Query Column Profile](https://user-images.githubusercontent.com/102996550/columnprofile.png)

---

### Step 5 — DAX Measures & Calculated Columns

#### Calculated Columns

**Age** (years since built):
```dax
Age = ABS(YEAR(Housing[date]) - Housing[year_build])
```

**Offer Price** (back-calculated from purchase price and % change):
```dax
Offer Price = (100 * Housing[purchase_price]) / (100 - [%_change_between_offer_and_purchase])
```

---

#### DAX Measures

**Year-on-Year Sales Growth:**
```dax
YOY_Sales_Growth = 
    VAR CurrentYearSales =
        CALCULATE(SUM(Housing[purchase_price]),
            YEAR(Housing[date]) = YEAR(MAX(Housing[date])))
    VAR PreviousYearSales =
        CALCULATE(SUM(Housing[purchase_price]),
            YEAR(Housing[date]) = YEAR(MAX(Housing[date])) - 1)
    RETURN
        IF(PreviousYearSales <> 0,
            (CurrentYearSales - PreviousYearSales) / PreviousYearSales, BLANK())
```

**Median Sales Price Change:**
```dax
Median Sales Price Change = 
    VAR CurrentMedianPrice =
        MEDIANX(FILTER(Housing,
            YEAR(Housing[date]) = YEAR(MAX(Housing[date]))),
            Housing[purchase_price])
    VAR PreviousMedianPrice =
        MEDIANX(FILTER(Housing,
            YEAR(Housing[date]) = YEAR(MAX(Housing[date])) - 1),
            Housing[purchase_price])
    RETURN
        IF(PreviousMedianPrice <> 0,
            (CurrentMedianPrice - PreviousMedianPrice) / PreviousMedianPrice, BLANK())
```

**Units Sold in Latest Year and Quarter:**
```dax
Units Sold in Latest Year and Quarter = 
    CALCULATE(DISTINCTCOUNT(Housing[house_id]),
        YEAR(Housing[date]) = YEAR(MAX(Housing[date])) &&
        QUARTER(Housing[date]) = QUARTER(MAX(Housing[date])))
```

**Last 12 Month Sales:**
```dax
Last 12 Month Sales = 
    CALCULATE(SUM(Housing[purchase_price]),
        DATESINPERIOD(Housing[date], MAX(Housing[date]), -12, MONTH))
```

**Sales by Region:**
```dax
Sales by Region = 
    CALCULATE(SUM(Housing[purchase_price]),
        ALLEXCEPT(Housing, Housing[region]))
```

**Total YTD Sales:**
```dax
Total YTD Sales = TOTALYTD(SUM(Housing[purchase_price]), Housing[date].[Date])
```

**Average Price SQM:**
```dax
Average Price SQM = AVERAGE(Housing[sqm_price])
```

**Offer to SQM Ratio:**
```dax
Offer to SQM Ratio = DIVIDE(SUM(Housing[Offer Price]), SUM(Housing[sqm]))
```

---

### Step 6 — Report Pages

#### Page 1: House Market Overview

- **Bar Chart** — Median Sales Price Change by region
- **Card Visual** — Units Sold (Latest Year & Quarter): **77**
- **Card Visual** — 12 Month Sales: **13bn**
- **Scatter Plot** — Offer Price vs. Purchase Price
- **Line Chart** — Year on Year Sales Growth by Sales Type

![House Market Overview](https://user-images.githubusercontent.com/102996550/page1p.png)

---

#### Page 2: Sales Performance

- **Funnel/Bar Chart** — Sales by Region (Zealand: 95bn | Jutland: 81bn | Fyn & islands: 15bn | Bornholm: 1bn)
- **Key Influencers Visual** — Factors influencing `purchase_price` increase (Age 2-16 → +501.1K avg)
- **Table Visual** — Date, Total YTD Sales, Sum of Purchase Price (from 1992 onwards)
- **Stacked Bar Chart** — Offer to SQM Ratio by Sales Type
- **Donut Chart** — Average Price SQM by region (Zealand: 35%, Fyn & islands: 23.25%, Bornholm: 18.1%)

![Sales Performance](https://user-images.githubusercontent.com/102996550/page2p.png)

---

#### Page 3: House Type Analysis

- **Clustered Bar Chart** — Average Offer / Purchase Price by House Type
- **Clustered Bar Chart** — Average Inflation / Interest / Yield by House Type
- **Line and Stacked Column Chart** — Average SQM / SQM Price by House Type
- **Slicers** — Area, City, Sales Type, Region

![House Type Analysis](https://user-images.githubusercontent.com/102996550/page3p.png)

---

### Step 7 — Published to Power BI Service

The three report pages were published to Power BI Service for sharing and collaboration.

---

## Dashboard Snapshots

### House Market Overview
![House Market Overview Dashboard](https://user-images.githubusercontent.com/102996550/housemarketd.png)

### Sales Performance
![Sales Performance Dashboard](https://user-images.githubusercontent.com/102996550/salesperformanced.png)

### House Type Analysis
![House Type Analysis Dashboard](https://user-images.githubusercontent.com/102996550/housetypeanalysisd.png)

---

## Insights

### [1] Regional Sales

- **Zealand** leads total sales volume at **95bn**, accounting for approximately 35% of average SQM price share.
- **Jutland** follows at **81bn**, with the **highest median sales price growth** among all regions.
- **Fyn & Islands** and **Bornholm** contribute **15bn** and **1bn** respectively.
- Bornholm has the **highest average SQM price share** at 18.1% relative to its sales volume, indicating premium pricing per square metre.

### [2] YOY Sales Growth by Sales Type

| Sales Type | YOY Growth |
|---|---|
| Auction | +0.29 |
| Regular Sale | -0.21 |
| Other Sale | -0.21 |
| Family Sale | -0.75 |

> Auctions are the only sales type with positive YOY growth. Family sales are declining sharply.

### [3] Offer to SQM Ratio by Sales Type

| Sales Type | Ratio |
|---|---|
| Regular Sale | 15K |
| Other Sale | 14K |
| Family Sale | 12K |
| Auction | 11K |

### [4] Key Influencer: Purchase Price

- When **Age is 2–16**, the average `purchase_price` increases by **501.1K** — newer properties command significantly higher prices.
- When **Age is more than 75**, the increase is only **391K**.

### [5] House Type Pricing

| House Type | Avg Offer Price | Avg Purchase Price | Yield |
|---|---|---|---|
| Farm | 2.7M | 2.7M | 4.6% |
| Apartment | 2.4M | 2.4M | 3.9% |
| Townhouse | 2.1M | 2.1M | 3.9% |
| Villa | 1.7M | 1.8M | 4.2% |
| Summerhouse | 1.2M | 1.2M | 3.8% |

- **Farms** have the highest average price and yield.
- **Apartments** have the highest SQM price at **28.7K**, making them the most expensive per square metre.
- **Farms** have the highest total SQM at **196.32**, reflecting their larger size.

### [6] Units Sold & Recent Activity

- **77 units** sold in the latest year and quarter.
- **13bn** in total sales over the last 12 months.
