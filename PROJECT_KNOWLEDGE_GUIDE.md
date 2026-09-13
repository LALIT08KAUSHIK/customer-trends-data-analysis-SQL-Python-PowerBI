# 🎓 Project Knowledge & Interview Master Guide
## *Customer Shopping Trends & Behavior Analysis*
**Author:** Lalit Kaushik ([@LALIT08KAUSHIK](https://github.com/LALIT08KAUSHIK))

> **Purpose of this Guide:**  
> This guide is your personal cheat sheet. It breaks down every component of the project—the business context, the Python data pipeline, the SQL queries, the Power BI dashboard, and the strategic recommendations. By reading this guide, you will be able to speak about this project with confidence, authority, and depth in technical and behavioral interviews.

---

## 📑 Table of Contents
1. [The 60-Second Elevator Pitch](#1-the-60-second-elevator-pitch)
2. [Business Context & Problem Statement](#2-business-context--problem-statement)
3. [Dataset Architecture & Features](#3-dataset-architecture--features)
4. [Python ETL & Data Preprocessing Deep-Dive](#4-python-etl--data-preprocessing-deep-dive)
5. [SQL Analytics & Query-by-Query Breakdown](#5-sql-analytics--query-by-query-breakdown)
6. [Power BI Dashboard Architecture & KPIs](#6-power-bi-dashboard-architecture--kpis)
7. [Strategic Business Recommendations](#7-strategic-business-recommendations)
8. [Top 10 Interview Questions & Model Answers](#8-top-10-interview-questions--model-answers)

---

## 1. The 60-Second Elevator Pitch

> *"When an interviewer says: **'Walk me through a project on your resume.'**, deliver this response:"*

> *"In this project, I built an end-to-end retail customer behavior analytics pipeline analyzing 3,900 customer purchasing transactions across 18 demographic and order attributes.*  
>  
> *I used **Python and Pandas** for exploratory data analysis, handling missing values through category-median imputation, creating age and frequency features, and eliminating redundant data. I then automated staging the cleaned data into **PostgreSQL** using SQLAlchemy.*  
>  
> *In SQL, I wrote complex queries using CTEs and window functions like `ROW_NUMBER()` to segment customers into loyalty tiers, evaluate subscription spend, and identify top-selling products per category.*  
>  
> *Finally, I designed an interactive **Power BI dashboard** delivering executive KPIs, category breakdowns, and cohort filters, enabling stakeholders to identify discount fatigue and craft targeted retention campaigns."*

---

## 2. Business Context & Problem Statement

### The Problem
A retail business noticed changing purchasing patterns across demographics, product categories, and sales channels. Management needed clear visibility into:
- Which factors (discounts, product ratings, season, shipping, payment methods) drive repeat purchases?
- Are discounts eroding margins without boosting cart sizes?
- Do subscribers generate significantly higher revenue than non-subscribers?
- How should customers be segmented to prioritize retention efforts?

### The Objective
Transform raw, uncleaned transactional data into a structured relational data model, answer core commercial questions using advanced SQL, and build an interactive BI dashboard for executive decision-making.

---

## 3. Dataset Architecture & Features

- **Total Records:** 3,900 transactions
- **Total Features:** 18 columns initially $\rightarrow$ cleaned to 18 features (after dropping 1 duplicate and adding 2 engineered features).
- **Core Dimensions:**
  1. **Customer Demographics:** `customer_id`, `age`, `gender`, `location`.
  2. **Product & Basket:** `item_purchased`, `category` (Clothing, Footwear, Outerwear, Accessories), `purchase_amount` (USD, range: \$20 to \$100, mean: ~\$59.76), `size` (S, M, L, XL), `color`, `season` (Spring, Summer, Fall, Winter).
  3. **Behavioral & Loyalty:** `review_rating` (2.5 to 5.0), `subscription_status` (Yes/No), `previous_purchases` (1 to 50), `frequency_of_purchases` (e.g., Weekly, Monthly, Annually).
  4. **Logistics & Financials:** `shipping_type` (Express, Free Shipping, Next Day Air, Standard, 2-Day Shipping, Store Pickup), `discount_applied` (Yes/No), `payment_method` (Credit Card, PayPal, Cash, Venmo, Bank Transfer, Debit Card).

---

## 4. Python ETL & Data Preprocessing Deep-Dive

All preprocessing steps take place in [`Customer_Shopping_Behavior_Analysis.ipynb`](Customer_Shopping_Behavior_Analysis.ipynb). Here is why each step was implemented:

### A. Missing Value Imputation
- **Observation:** `review_rating` had 37 null values (`Review Rating: 3863 non-null`).
- **Why NOT drop them?** Dropping rows causes loss of valid transactional and revenue data.
- **Why NOT use the global mean?** Different product categories have different baseline satisfaction levels (e.g., clothing sizing issues vs. outerwear durability).
- **Solution:** Impute with the **category median rating**:
  ```python
  df['Review Rating'] = df.groupby('Category')['Review Rating'].transform(lambda x: x.fillna(x.median()))
  ```
  *Median is robust against extreme outlier ratings.*

### B. Header Standardization
- Renamed headers to lowercase snake_case (e.g., `Purchase Amount (USD)` $\rightarrow$ `purchase_amount`). This prevents syntax bugs and quotes in PostgreSQL queries.

### C. Feature Engineering
1. **Age Cohort Segmentation (`age_group`):**
   - Used quantile-based binning (`pd.qcut(df['age'], q=4)`) into 4 equal-sized quartiles:
     - `Young Adult` (18–31)
     - `Adult` (31–44)
     - `Middle-aged` (44–57)
     - `Senior` (57–70)
   - *Why `qcut` over `cut`?* `qcut` ensures an equal number of customers in each cohort, avoiding sample size imbalance during comparative statistical analysis.
2. **Standardized Purchase Frequency (`purchase_frequency_days`):**
   - Mapped textual strings to numeric days: Weekly $\rightarrow$ 7, Fortnightly $\rightarrow$ 14, Monthly $\rightarrow$ 30, Quarterly $\rightarrow$ 90, Annually $\rightarrow$ 365.
   - Enables mathematical correlation and quantitative modeling of churn risk.

### D. Multicollinearity / Duplicate Feature Removal
- Verified whether `promo_code_used` and `discount_applied` were redundant:
  ```python
  (df['discount_applied'] == df['promo_code_used']).all() # Evaluated to True
  ```
- Because every customer who used a discount used a promo code, retaining both introduces perfect multicollinearity. `promo_code_used` was dropped.

### E. Database Ingestion Pipeline
- Used **SQLAlchemy** to connect to PostgreSQL and loaded the cleaned DataFrame:
  ```python
  engine = create_engine(f"postgresql+psycopg2://{username}:{password}@{host}:{port}/{database}")
  df.to_sql("customer", engine, if_exists="replace", index=False)
  ```

---

## 5. SQL Analytics & Query-by-Query Breakdown

Here is the technical explanation and business takeaway for all 10 SQL queries in [`customer_behavior_sql_queries.sql`](customer_behavior_sql_queries.sql):

### Q1: Total Revenue by Gender
- **SQL Concept:** Basic `GROUP BY` and `SUM()`.
- **Query:**
  ```sql
  SELECT gender, SUM(purchase_amount) AS revenue
  FROM customer
  GROUP BY gender;
  ```
- **Finding:** Male customers generated higher aggregate revenue (~67%) primarily due to higher transaction volume in the dataset (~2,652 males vs. 1,248 females).

### Q2: Discount Users Spending Above Average
- **SQL Concept:** Subquery in `WHERE` clause.
- **Query:**
  ```sql
  SELECT customer_id, purchase_amount 
  FROM customer 
  WHERE discount_applied = 'Yes' 
    AND purchase_amount >= (SELECT AVG(purchase_amount) FROM customer);
  ```
- **Business Insight:** Identifies high-value customers who still purchase large baskets despite discount usage—prime candidates for premium loyalty tiers rather than discount-heavy promotions.

### Q3: Top 5 Highest-Rated Products
- **SQL Concept:** `AVG()`, type casting (`review_rating::numeric`), `ORDER BY ... DESC LIMIT 5`.
- **Query:**
  ```sql
  SELECT item_purchased, ROUND(AVG(review_rating::numeric), 2) AS "Average Product Rating"
  FROM customer
  GROUP BY item_purchased
  ORDER BY AVG(review_rating) DESC
  LIMIT 5;
  ```
- **Business Insight:** Highlights top-performing hero items (e.g., Gloves, Boots) to feature in marketing campaigns and homepage recommendations.

### Q4: Average Spend: Standard vs. Express Shipping
- **SQL Concept:** Filtering with `IN ('Standard', 'Express')` and calculating average order value.
- **Query:**
  ```sql
  SELECT shipping_type, ROUND(AVG(purchase_amount), 2)
  FROM customer
  WHERE shipping_type IN ('Standard', 'Express')
  GROUP BY shipping_type;
  ```
- **Business Insight:** Compares basket size between delivery methods. Express shipping buyers typically have equal or slightly higher urgency and spend.

### Q5: Subscription Spend & Revenue Contribution
- **SQL Concept:** Multi-metric aggregation (`COUNT`, `AVG`, `SUM`).
- **Query:**
  ```sql
  SELECT subscription_status,
         COUNT(customer_id) AS total_customers,
         ROUND(AVG(purchase_amount), 2) AS avg_spend,
         ROUND(SUM(purchase_amount), 2) AS total_revenue
  FROM customer
  GROUP BY subscription_status
  ORDER BY total_revenue, avg_spend DESC;
  ```
- **Business Insight:** Average spend per transaction is nearly identical between subscribers and non-subscribers (~$59-$60), but non-subscribers represent ~73% of volume. Subscription conversion is the single biggest revenue expansion opportunity.

### Q6: Top 5 Products with Highest Discount Rate
- **SQL Concept:** Conditional aggregation (`CASE WHEN`) to compute a percentage.
- **Query:**
  ```sql
  SELECT item_purchased,
         ROUND(100.0 * SUM(CASE WHEN discount_applied = 'Yes' THEN 1 ELSE 0 END) / COUNT(*), 2) AS discount_rate
  FROM customer
  GROUP BY item_purchased
  ORDER BY discount_rate DESC
  LIMIT 5;
  ```
- **Business Insight:** Reveals products that customers rarely buy at full price. Identifies potential pricing weakness or over-reliance on markdowns.

### Q7: Customer Loyalty Segmentation (New, Returning, Loyal)
- **SQL Concept:** Common Table Expression (CTE) with `CASE` logic.
- **Query:**
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
- **Business Insight:** Loyal customers (>10 purchases) form the core backbone of repeat purchases, proving high customer lifetime value (LTV).

### Q8: Top 3 Most Purchased Products per Category (Window Function)
- **SQL Concept:** Analytical Window Function `ROW_NUMBER() OVER (PARTITION BY category ORDER BY COUNT(customer_id) DESC)`.
- **Query:**
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
- **Business Insight:** Vital for category managers to optimize warehouse stocking and bundle complementary products.

### Q9: Repeat Buyers vs. Subscription Propensity
- **SQL Concept:** Filtering on repeat purchases (`> 5`) and grouping by subscription.
- **Query:**
  ```sql
  SELECT subscription_status,
         COUNT(customer_id) AS repeat_buyers
  FROM customer
  WHERE previous_purchases > 5
  GROUP BY subscription_status;
  ```
- **Business Insight:** A significant proportion of repeat buyers are not yet subscribed. This segment represents low-hanging fruit for subscription onboarding.

### Q10: Revenue Contribution by Age Cohort
- **SQL Concept:** Aggregation across engineered feature `age_group`.
- **Query:**
  ```sql
  SELECT age_group,
         SUM(purchase_amount) AS total_revenue
  FROM customer
  GROUP BY age_group
  ORDER BY total_revenue DESC;
  ```
- **Business Insight:** Revenue is evenly distributed across age cohorts (~25% each), confirming the brand has broad multi-generational appeal rather than a niche demographic.

---

## 6. Power BI Dashboard Architecture & KPIs

The dashboard file [`customer_behavior_dashboard.pbix`](customer_behavior_dashboard.pbix) provides visual analytics across 4 quadrants:

1. **Executive KPI Header:**
   - Total Revenue (\$233K+)
   - Total Customers (3,900)
   - Average Order Value (\$59.76)
   - Average Rating (3.75 / 5.0)

2. **Category & Product Performance:**
   - Bar chart of Revenue by Category (Clothing dominates, followed by Accessories, Footwear, Outerwear).
   - Treemap of top items within categories.

3. **Customer Demographics & Channels:**
   - Donut charts for Gender split and Subscription status.
   - Clustered column charts comparing Age Groups across Seasons.

4. **Fulfillment & Payment Breakdown:**
   - Shipping preference distribution (Free Shipping and Standard are most popular).
   - Payment method breakdown (PayPal, Credit Card, Cash leading).

---

## 7. Strategic Business Recommendations

When interviewers ask *"What recommendations would you make to the business based on your analysis?"*, mention these 4 points:

1. **Launch a Subscription Conversion Campaign:**
   Since average order value is similar between subscribers and non-subscribers, the key is conversion. Target non-subscribers with $>5$ previous purchases with a "First Month Free" or "Free Shipping on All Orders" membership hook.
2. **Re-evaluate Discount Strategy on High-Demand Products:**
   Products with discount rates exceeding 45% should be tested at lower discount depths (e.g., 10% instead of 20%) to prevent unnecessary margin erosion.
3. **Incentivize Higher Cart Values via Shipping Thresholds:**
   Since Free Shipping is the most chosen method, set a dynamic minimum threshold (e.g., "Free Shipping on orders over \$75") to push the current \$59.76 average order value higher.
4. **Category Stock Optimization:**
   Focus inventory investments on category volume leaders (Blouses, Sweaters, Jeans) while reviewing low-rated accessories to reduce return rates.

---

## 8. Top 10 Interview Questions & Model Answers

### Q1: "Why did you impute review rating with category median instead of mean or 0?"
> **Answer:** *"Review ratings are bounded between 1 and 5 and often display negative or positive skewness. The median is resistant to extreme outliers. Furthermore, customer satisfaction differs across categories—for example, Footwear has higher variance due to fit issues compared to Accessories. Category-level median imputation preserves the natural distribution of the category."*

### Q2: "What is the difference between `pd.cut` and `pd.qcut` in Pandas, and why did you choose `pd.qcut` for age groups?"
> **Answer:** *"`pd.cut` divides the range into equal-width bins (e.g., 20-30, 30-40), which can result in bins with very few or too many data points if the distribution is skewed. `pd.qcut` divides the data based on sample quantiles, ensuring each of the 4 age groups has an approximately equal number of records (~975 customers each), which makes comparative statistical analysis balanced."*

### Q3: "Explain how you used a window function in SQL and why a standard GROUP BY wasn't enough."
> **Answer:** *"In Query 8, the business wanted the top 3 items within each category. A standard `GROUP BY category, item_purchased ORDER BY count DESC LIMIT 3` would return only 3 items across the entire company. To get top 3 per category, I used a CTE with `ROW_NUMBER() OVER (PARTITION BY category ORDER BY COUNT(customer_id) DESC) AS item_rank`, and then filtered for `WHERE item_rank <= 3`."*

### Q4: "What is the difference between `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()`?"
> **Answer:** *"`ROW_NUMBER()` always assigns unique sequential integers (1, 2, 3), regardless of ties. `RANK()` assigns identical ranks to ties and skips subsequent numbers (e.g., 1, 2, 2, 4). `DENSE_RANK()` assigns identical ranks to ties but does not skip numbers (e.g., 1, 2, 2, 3). For inventory ranking, `ROW_NUMBER()` guarantees a strict top 3 cutoff."*

### Q5: "Why did you drop the `promo_code_used` column in Python?"
> **Answer:** *"During EDA, I cross-tabulated `discount_applied` and `promo_code_used` and ran `(df['discount_applied'] == df['promo_code_used']).all()`. It returned `True`, indicating 100% duplicate information. Keeping duplicate features wastes storage and creates multicollinearity in machine learning models."*

### Q6: "How did you connect your Python pipeline to PostgreSQL?"
> **Answer:** *"I used the `SQLAlchemy` library paired with `psycopg2-binary`. I created a connection engine string specifying host, port, database, and credentials, then used `df.to_sql('customer', engine, if_exists='replace', index=False)` to automate table creation and batch insertion."*

### Q7: "What were the primary drivers of revenue in this dataset?"
> **Answer:** *"Clothing was the highest-grossing category (~44% of total revenue). Repeat customers (>10 purchases) accounted for the bulk of stable volume. While male customers contributed ~67% of total revenue, average spend per transaction between males (\$59.5) and females (\$60.2) was nearly identical—the revenue difference was driven by sample transaction count, not individual purchasing power."*

### Q8: "How does Power BI connect to this data, and what modeling did you apply?"
> **Answer:** *"Power BI connects directly to either the PostgreSQL database or the clean CSV export. In Power BI, I established explicit DAX measures for Total Revenue (`SUM`), AOV (`AVERAGE`), and Total Orders (`DISTINCTCOUNT`), and built hierarchical drill-downs from Category down to Item Purchased."*

### Q9: "If you had more time, what additional analytics would you perform?"
> **Answer:** *"I would implement an RFM (Recency, Frequency, Monetary) segmentation model to calculate dynamic churn probability scores, run Market Basket Analysis using the Apriori algorithm to uncover product cross-selling affinity (e.g., customers who buy jeans also buy belts), and build a predictive classification model to forecast subscription conversion."*

### Q10: "How did you ensure data quality during the pipeline?"
> **Answer:** *"I verified data integrity at each checkpoint: running `.info()` and `.isnull().sum()` to identify nulls, checking for unexpected categorical values with `.value_counts()`, confirming data types were appropriate (converting numeric ratings and string categories), and testing for feature collinearity before staging into PostgreSQL."*
