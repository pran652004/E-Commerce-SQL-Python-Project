---

# Project Insight ::

## Overview

This project shows an end-to-end analysis workflow on an e-commerce dataset.
Data is stored in **MySQL**, key questions are answered with **SQL** (saved in `Queries/`), and results are pulled into **Python (Jupyter)** for quick EDA and charts in the notebook **`Python+SQL-Ecommerce_Project.ipynb`**. There’s also a small helper **`csv_to_sql.py`** script for loading CSVs into MySQL. ([GitHub][1])

---

## Dataset

* **Source:** Public e-commerce CSVs (customers, orders, items, products, payments).
* **Example dataset link:** Kaggle “Target-dataset” (adjust schema as needed): [https://www.kaggle.com/datasets/devarajv88/target-dataset?select=products.csv](https://www.kaggle.com/datasets/devarajv88/target-dataset?select=products.csv)

> If you use different CSVs/columns, update table definitions and the load step accordingly.

---

## Repository Structure

```
E-Commerce-SQL-Python-Project/
├─ Queries/                        # Saved SQL scripts for KPIs, trends, cohorts, etc.
├─ Python+SQL-Ecommerce_Project.ipynb
├─ csv_to_sql.py                   # Minimal loader (CSV → MySQL)
└─ README.md
```
---

## What this project answers

1. **Core KPIs:** total orders, revenue, Average Order Value (AOV); monthly trend
2. **Products & Categories:** top products/categories by revenue and mix
3. **Customers:** new vs repeat; simple first-order cohort / retention view
4. **Payments:** payment method mix over time

---

## Getting Started

### 1) MySQL: create DB, tables, and load data

Create a database and user (adjust to your setup):

```sql
CREATE DATABASE ecom_insight;
CREATE USER 'ecom_user'@'%' IDENTIFIED BY 'strongpass';
GRANT ALL PRIVILEGES ON ecom_insight.* TO 'ecom_user'@'%';
FLUSH PRIVILEGES;
```

Create tables (example schema you can adapt):

```sql
USE ecom_insight;

CREATE TABLE customers (
  customer_id   INT PRIMARY KEY,
  customer_name VARCHAR(128),
  email         VARCHAR(128),
  city          VARCHAR(64),
  state         VARCHAR(64),
  country       VARCHAR(64),
  created_at    DATE
);

CREATE TABLE products (
  product_id   INT PRIMARY KEY,
  product_name VARCHAR(128),
  category     VARCHAR(64),
  sub_category VARCHAR(64),
  unit_price   DECIMAL(10,2)
);

CREATE TABLE fact_orders (
  order_id     INT PRIMARY KEY,
  customer_id  INT,
  order_date   DATE,
  order_status VARCHAR(32),
  order_amount DECIMAL(12,2),
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
  order_item_id INT PRIMARY KEY,
  order_id      INT,
  product_id    INT,
  quantity      INT,
  unit_price    DECIMAL(10,2),
  FOREIGN KEY (order_id)  REFERENCES fact_orders(order_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);

CREATE TABLE payments (
  payment_id     INT PRIMARY KEY,
  order_id       INT,
  payment_method VARCHAR(32),
  amount         DECIMAL(12,2),
  paid_at        DATETIME,
  FOREIGN KEY (order_id) REFERENCES fact_orders(order_id)
);
```

Load CSVs (choose one):

**Option A – use `csv_to_sql.py`** (edit file paths/cols inside if needed):

```bash
python csv_to_sql.py
```

(See the script in the repo.) ([GitHub][2])

**Option B – MySQL `LOAD DATA`** (sample):

```sql
SET GLOBAL local_infile = 1;

LOAD DATA LOCAL INFILE '/path/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
IGNORE 1 LINES
(customer_id, customer_name, email, city, state, country, @created_at)
SET created_at = STR_TO_DATE(@created_at, '%Y-%m-%d');
-- Repeat for products, fact_orders, order_items, payments...
```

### 2) Python environment

```bash
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install pandas matplotlib seaborn mysql-connector-python python-dotenv
```

Create a `.env` file (if notebook expects it):

```
DB_HOST=localhost
DB_PORT=3306
DB_NAME=ecom_insight
DB_USER=ecom_user
DB_PASS=strongpass
```

Open the notebook:

```bash
jupyter notebook "Python+SQL-Ecommerce_Project.ipynb"
```

(Notebook filename verified from repo.) ([GitHub][1])

---

## Query samples

**Monthly orders & revenue**

```sql
SELECT
  DATE_FORMAT(order_date, '%Y-%m') AS ym,
  COUNT(DISTINCT order_id)         AS orders,
  SUM(order_amount)                AS revenue
FROM fact_orders
GROUP BY ym
ORDER BY ym;
```

**Category revenue mix**

```sql
SELECT
  p.category,
  SUM(oi.quantity * oi.unit_price) AS revenue,
  ROUND(
    100 * SUM(oi.quantity * oi.unit_price)
    / SUM(SUM(oi.quantity * oi.unit_price)) OVER (), 2
  ) AS pct_share
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY revenue DESC;
```

**Top 3 customers per year**

```sql
SELECT *
FROM (
  SELECT
    YEAR(o.order_date) AS yr,
    o.customer_id,
    SUM(o.order_amount) AS revenue,
    RANK() OVER (
      PARTITION BY YEAR(o.order_date)
      ORDER BY SUM(o.order_amount) DESC
    ) AS rnk
  FROM fact_orders o
  GROUP BY yr, o.customer_id
) t
WHERE rnk <= 3
ORDER BY yr, rnk;
```

**First-order cohort (simplified)**

```sql
WITH first_order AS (
  SELECT customer_id, MIN(order_date) AS first_dt
  FROM fact_orders
  GROUP BY customer_id
),
orders_with_cohort AS (
  SELECT
    o.customer_id,
    o.order_date,
    DATE_FORMAT(f.first_dt, '%Y-%m') AS cohort_ym,
    TIMESTAMPDIFF(MONTH, f.first_dt, o.order_date) AS months_since
  FROM fact_orders o
  JOIN first_order f USING (customer_id)
)
SELECT cohort_ym, months_since, COUNT(DISTINCT customer_id) AS active_customers
FROM orders_with_cohort
GROUP BY cohort_ym, months_since
ORDER BY cohort_ym, months_since;
```
---

## Results ::

* Which **categories/products** lead revenue and by what share
* **AOV** trend and a short hypothesis for changes
* Seasonal **peaks/dips** in monthly orders/revenue
* Basic pattern in **repeat behavior** from the cohort table
* One concrete **next step** (e.g., a promotion for a slow category or a bundle test)

Keep exact numbers in the notebook so the README stays dataset-agnostic.

---
