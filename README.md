# 🛒 Customer Shopping Trends & Behavior Analysis
### *End-to-End Retail Data Analytics & Business Intelligence Pipeline*

[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Power BI](https://img.shields.io/badge/Power_BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)](https://powerbi.microsoft.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

---

## 📌 Executive Summary

In the modern retail landscape, understanding consumer buying patterns, churn drivers, and customer lifetime value is essential for sustainable growth. This project delivers a comprehensive, production-grade data analytics pipeline analyzing **3,900 customer shopping transactions** across demographics, product categories, and purchasing channels.

Using **Python (Pandas)** for exploratory data analysis (EDA) and automated data modeling, **PostgreSQL** for relational schema staging and advanced analytical queries (CTEs, Window Functions), and **Power BI** for executive-level interactive visualization, this project transforms raw transactional data into actionable commercial strategies.

---

## 🏗️ Analytics Pipeline Architecture

```
┌─────────────────────────┐
│   Raw Transaction Data  │ ──> customer_shopping_behavior.csv (3,900 records, 18 features)
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│    Python ETL & EDA     │ ──> Missing value imputation (category-median rating)
│    (Pandas / NumPy)     │ ──> Feature Engineering: Age Cohorts (qcut), Frequency mapping
└───────────┬─────────────┘ ──> Collinearity pruning & snake_case standardization
            │
            ▼
┌─────────────────────────┐
│  Relational DB Staging  │ ──> Automated ingestion via SQLAlchemy & psycopg2
│      (PostgreSQL)       │ ──> Staging table: customer
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Deep-Dive Analytics   │ ──> 10 Business Queries: Window Functions (ROW_NUMBER),
│          (SQL)          │     CTEs, Cohort Segmentation (New / Returning / Loyal)
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Interactive BI & Visuals│ ──> Executive Dashboard: customer_behavior_dashboard.pbix
│       (Power BI)        │ ──> KPIs, Category breakdown, Shipping & Subscription ROI
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Stakeholder Deliverable │ ──> Executive Summary Report & Presentation Deck (PDF / PPTX)
└─────────────────────────┘
```

---

## 📊 Dataset Overview

The dataset contains transactional records capturing customer demographics, purchasing behavior, product attributes, and marketing engagement:

| Dimension | Key Features |
| :--- | :--- |
| **Demographics** | `customer_id`, `age`, `gender`, `location` |
| **Product & Order** | `item_purchased`, `category`, `purchase_amount`, `size`, `color`, `season` |
| **Behavioral & Loyalty** | `review_rating`, `subscription_status`, `previous_purchases`, `frequency_of_purchases` |
| **Marketing & Logistics** | `shipping_type`, `discount_applied`, `promo_code_used`, `payment_method` |

---

## 🛠️ Data Preparation & Feature Engineering (Python)

All preprocessing and automated pipeline steps are documented in [`Customer_Shopping_Behavior_Analysis.ipynb`](Customer_Shopping_Behavior_Analysis.ipynb):

1. **Missing Data Imputation:**
   - Identified 37 missing values in `Review Rating`.
   - Rather than dropping records or using a global mean, missing values were imputed using the **median rating per product category**, preserving category-level variance:
     ```python
     df['Review Rating'] = df.groupby('Category')['Review Rating'].transform(lambda x: x.fillna(x.median()))
     ```

2. **Column Standardization:**
   - Normalized all headers to lowercase snake_case (e.g., `Purchase Amount (USD)` $\rightarrow$ `purchase_amount`).

3. **Feature Engineering:**
   - **Age Cohorts (`age_group`):** Applied quantile-based discretization (`pd.qcut`) into 4 balanced segments: `Young Adult`, `Adult`, `Middle-aged`, and `Senior`.
   - **Purchase Cycle (`purchase_frequency_days`):** Mapped qualitative frequencies to standard cycle days (e.g., Weekly = 7, Fortnightly = 14, Monthly = 30, Annually = 365).

4. **Multicollinearity Pruning:**
   - Evaluated `discount_applied` vs. `promo_code_used`. Confirmed 100% equivalence (`(df['discount_applied'] == df['promo_code_used']).all() == True`) and removed redundant feature to eliminate collinearity.

5. **Database Ingestion Engine:**
   - Built a scalable database connection with **SQLAlchemy** to automatically stage data into PostgreSQL, MySQL, or SQL Server.

---

## 💡 SQL Business Intelligence & Key Findings

All production queries are organized in [`customer_behavior_sql_queries.sql`](customer_behavior_sql_queries.sql). Highlights include:

### 1. Customer Loyalty & Retention Segmentation (CTE)
```sql
WITH customer_type AS (
    SELECT customer_id, previous_purchases,
    CASE 
        WHEN previous_purchases = 1 THEN 'New'
        WHEN previous_purchases BETWEEN 2 AND 10 THEN 'Returning'
        ELSE 'Loyal'
    END AS customer_segment
    FROM customer
)
SELECT customer_segment, COUNT(*) AS "Number of Customers" 
FROM customer_type 
GROUP BY customer_segment;
```
> **Insight:** Returning and loyal customers account for the majority of transaction volume, demonstrating high repeat engagement and the need for retention-focused rewards.

### 2. Top 3 Highest-Demand Products by Category (Window Function)
```sql
WITH item_counts AS (
    SELECT category,
           item_purchased,
           COUNT(customer_id) AS total_orders,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY COUNT(customer_id) DESC) AS item_rank
    FROM customer
    GROUP BY category, item_purchased
)
SELECT item_rank, category, item_purchased, total_orders
FROM item_counts
WHERE item_rank <= 3;
```
> **Insight:** Accurately isolates category leaders (e.g., Blouse and Sweater in Clothing; Sandals and Sneakers in Footwear), guiding inventory stocking priorities.

### 3. Subscription Economics & Spend Comparison
```sql
SELECT subscription_status,
       COUNT(customer_id) AS total_customers,
       ROUND(AVG(purchase_amount), 2) AS avg_spend,
       ROUND(SUM(purchase_amount), 2) AS total_revenue
FROM customer
GROUP BY subscription_status
ORDER BY total_revenue DESC;
```
> **Insight:** Subscribed customers display consistent order values and loyalty, but represent a smaller share of overall customer count—highlighting a key growth lever for membership conversions.

---

## 📈 Power BI Executive Dashboard

The interactive dashboard file [`customer_behavior_dashboard.pbix`](customer_behavior_dashboard.pbix) delivers dynamic visual analytics for decision-makers:

- **Executive KPI Cards:** Total Revenue, Average Order Value (AOV), Total Orders, and Customer Satisfaction Index (Avg Rating).
- **Category & Product Performance:** Revenue breakdown and rating distribution across Clothing, Footwear, Outerwear, and Accessories.
- **Logistics & Payment Analysis:** Order volume distribution across Express vs. Free Shipping and payment preferences (PayPal, Credit Card, Cash, Venmo).
- **Cohort Drill-downs:** Slicers by Age Group, Gender, Season, and Subscription Status.

---

## 🎯 Strategic Business Recommendations

1. **Targeted Loyalty Rewards:** Customers with $>10$ previous purchases drive baseline stability; offer tier-based loyalty perks to prevent churn.
2. **Subscription Upsell Strategy:** Because average basket size is healthy across subscribers, introduce entry incentives (e.g., first-order shipping perks) to convert frequent non-subscribers.
3. **Discount Optimization:** Analyze margin impact on high-demand items where discount usage is high but organic demand is strong, reducing unnecessary margin erosion.
4. **Logistics Alignment:** Express shipping users demonstrate slightly higher order value—bundle expedited delivery thresholds with minimum cart values to increase AOV.

---

## 📂 Repository Structure

```
├── Business Problem  Document.pdf        # Problem statement & business deliverables
├── Customer Shopping Behavior Analysis.pdf# Executive presentation report (PDF)
├── Customer-Shopping-Behavior-Analysis.pptx# Stakeholder presentation slide deck
├── Customer_Shopping_Behavior_Analysis.ipynb# Python ETL, data cleaning, & DB loading
├── customer_behavior_dashboard.pbix     # Interactive Power BI dashboard file
├── customer_behavior_sql_queries.sql    # 10 business problem SQL queries
├── customer_shopping_behavior.csv       # Raw transactional dataset (3,900 rows)
├── PROJECT_KNOWLEDGE_GUIDE.md           # In-depth project breakdown & interview prep guide
├── RESUME_AND_GITHUB_GUIDE.md           # Resume bullet points & GitHub upload guide
├── .gitignore                           # Git ignore rules
├── LICENSE                              # MIT License
└── README.md                            # Comprehensive project documentation
```

---

## 🚀 How to Replicate This Project Locally

### 1. Prerequisites
- Python 3.9+ (`pandas`, `sqlalchemy`, `psycopg2-binary`)
- PostgreSQL / MySQL / SQL Server
- Power BI Desktop (Windows)

### 2. Setup Steps
1. **Clone the Repository:**
   ```bash
   git clone https://github.com/LALIT08KAUSHIK/customer-trends-data-analysis-SQL-Python-PowerBI.git
   cd customer-trends-data-analysis-SQL-Python-PowerBI
   ```

2. **Run Data Cleaning & Database Load:**
   - Open and execute [`Customer_Shopping_Behavior_Analysis.ipynb`](Customer_Shopping_Behavior_Analysis.ipynb).
   - Enter your local database credentials to automatically stage tables.

3. **Run SQL Queries:**
   - Open your SQL client (e.g., pgAdmin / DBeaver) and run [`customer_behavior_sql_queries.sql`](customer_behavior_sql_queries.sql).

4. **Launch Power BI Dashboard:**
   - Open [`customer_behavior_dashboard.pbix`](customer_behavior_dashboard.pbix) in Power BI Desktop to explore and interact with visual reports.

---

## 👨‍💻 Author

**Lalit Kaushik**  
- **GitHub:** [@LALIT08KAUSHIK](https://github.com/LALIT08KAUSHIK)  
- **Role:** Data Analyst / Business Intelligence Enthusiast  

*If you found this project helpful, feel free to give it a ⭐ on GitHub!*
