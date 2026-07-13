# **Loss Analysis: Discounts Eroded 38.5% of Superstore Profit**
An end-to-end analysis of discount-driven profit erosion on the Global Superstore dataset — from EDA and database design to SQL analysis and an interactive dashboard with a what-if simulation.

🔗  [View the Interactive Dashboard on Looker Studio](https://datastudio.google.com/reporting/7524752c-899b-4ed1-b9ab-838741477c33)

## Dashboard Preview


### 📌 Executive Summary
Global Superstore posted a **net profit of $1.47M** — but hidden beneath that figure lies a **cumulative loss of $920.7K** across 11.3K loss-making transactions, equivalent to **38.6% of profit eroded**.

This analysis demonstrates that discounting is the primary driver of losses: 
- **every discount band above 20% consistently produces negative profit**
- **A what-if simulation** shows that **capping discounts at 20% would recover ~70% of lost profit**, lifting projected profit from **$1.47M to $2.50M (+70.3%)**.

## **Metric	Value**
- Total Actual Profit	$1,467,456.55
- Total Cumulative Loss	−$920.7K
- Profit Erosion Percent	38.6%
- Loss Transactions	11.3K
- Simulated Profit (20% discount cap)	$2.50M
- Simulated Profit Increase	+70.3%


## 🎯 Business Context & Questions
Retail management often uses discounts to drive sales volume without monitoring the impact on per-transaction profitability. This project answers four questions:

- How much profit is being eroded by loss-making transactions?
- At what discount level do transactions consistently start losing money?
- Which product categories are most affected?
- How much profit could be recovered if discounts were capped?

## 🗂️ Dataset
Source: Global Superstore Dataset (public, ~51K retail transaction rows)
Scope: Cross-region sales transactions with order, product, customer, shipping, discount, sales, and profit attributes
Granularity: One row = one order line item

## 🛠️ Tools & Technologies
### Stage	Tools
- Exploratory Data Analysis	Python (pandas), 
- Database & Data Modeling	PostgreSQL (hosted on Aiven), pgAdmin
- Analysis	SQL (CTEs, window functions, CASE WHEN bucketing)
- Visualization	Looker Studio (live connection to PostgreSQL)
  
### 🔄 Methodology
#### 1. Exploratory Data Analysis (Python)
Profiled data structure, column types, missing values, and duplicates in a Jupyter Notebook
Analyzed the distributions of discount, sales, and profit to identify early loss patterns
Validated the initial hypothesis: a negative relationship between discount level and profit
#### 2. Schema Design & Import into PostgreSQL
Designed the table schema in pgAdmin with a surrogate key to preserve row integrity
Resolved CSV import issues (encoding, date formats, delimiters)
Deduplicated the data to guarantee unique order lines — critical to prevent inflated aggregations downstream
#### 3. SQL Analysis: Discount Bands & Profit Erosion
Transactions were grouped into discount bands using CASE WHEN with explicit, non-overlapping bounds:

````SELECT
  CASE
    WHEN discount = 0                          THEN '0%'
    WHEN discount > 0    AND discount <= 0.20  THEN '0–20%'
    WHEN discount > 0.20 AND discount <= 0.40  THEN '20–40%'
    WHEN discount > 0.40 AND discount <= 0.60  THEN '40–60%'
    ELSE '60–85%'
  END AS discount_band,
  COUNT(DISTINCT order_id)                            AS transactions,
  SUM(profit)                                         AS total_profit,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY profit) AS median_profit
FROM transactions
GROUP BY 1
ORDER BY 1;

The Profit Erosion Percent metric is defined as the ratio of total losses to gross positive profit:

SELECT
  ABS(SUM(profit) FILTER (WHERE profit < 0))
  / SUM(profit) FILTER (WHERE profit > 0) * 100 AS profit_erosion_percent
FROM transactions;
````
#### 4. What-If Simulation: Capping Discounts at 20%
The simulation recomputes profit under the assumption that every transaction discounted above 20% is re-priced at a maximum 20% discount, with margin re-estimated proportionally:
```
WITH simulation AS (
  SELECT
    profit AS actual_profit,
    CASE
      WHEN discount > 0.20
        THEN (sales / (1 - discount)) * (1 - 0.20)
             * (profit / NULLIF(sales, 0))  -- margin re-estimated at the new price
      ELSE profit
    END AS simulated_profit
  FROM transactions
)
SELECT
  SUM(actual_profit)    AS total_actual_profit,
  SUM(simulated_profit) AS total_simulated_profit,
  (SUM(simulated_profit) - SUM(actual_profit))
    / SUM(actual_profit) * 100 AS increase_percent
FROM simulation;
⚠️ Note: adapt the queries above to match the final version used in the project — the logic is identical, but the margin estimation formula may differ.
`````

#### 5. Looker Studio Dashboard
- Live connection to PostgreSQL on Aiven via custom queries
- KPI scorecards: Total Profit, Total Loss, Profit Erosion Percent, Loss Transactions
- Combo chart: profit (bars) vs. transaction count (line) per discount band, with annotation highlighting the loss zone
- What-if panel comparing actual vs. simulated profit
- Interactive filters: region, product category
- Percentage display types multiply values by 100 — values must be stored as decimals
- Cross-chart reconciliation: every profit chart was validated to sum back to $1,467,456.55
<img width="1339" height="1009" alt="dashboard_loss analysis" src="https://github.com/user-attachments/assets/a3c065a8-9b1d-4cf5-b806-4a8e6f65fd1d" />

#### 💡 Key Findings
- 38.6% of profit is eroded. $920.7K in gross profit is lost through 11.3K loss-making transactions.
- Discounts above 20% are the tipping point. The 0% and 0–20% bands generate positive profit ($1.8M and $511.4K). Every band above 20% is negative: −$187.3K (20–40%), −$388.3K (40–60%), −$239.2K (60–85%) — and their median profit is negative too, meaning the losses are systemic, not caused by a handful of outliers.
- Volume doesn't compensate. High-discount bands contribute far fewer transactions (2.8K and 1.3K vs. 15.2K in the 0% band) — deep discounts do not generate enough volume to justify the losses.
- Furniture leads loss contribution ($370.2K), followed by technology ($286.5K) and office supplies ($264K).
- Capping discounts at 20% recovers ~70% of lost profit, lifting total profit to $2.50M.

#### ✅ Business Recommendations
-Set a default maximum discount of 20%; anything above requires special approval with a documented margin justification.
- Audit the 40–60% discount band — the largest loss contributor (−$388.3K) — to identify the most problematic products and regions.
- Track Profit Erosion Percent as a monthly KPI, not just total profit, so hidden losses are detected early.
  

## 👤 About Me
**Indah Permatasari** — Office Administration Lecturer expanding into data analytics. Experienced in building end-to-end data pipelines: from database design and SQL analysis to visualization.


