# FMCG Executive KPI & Supply Chain Reporting System

An end-to-end business intelligence and enterprise reporting platform built for Fast-Moving Consumer Goods (FMCG) operations. This project establishes an automated pipeline from a normalized MySQL relational database star schema to an executive Power BI dashboard featuring dynamic DAX financial tickers, conditional color-status formatting, and multi-dimensional supply chain tracking.

---

## Key Features & Architecture

* **Relational Database Design (MySQL):** 
  * Structured using a star schema (`fmcg_kpi_reporting`) consisting of dimension tables (`dim_date`, `dim_product`) and fact tables (`fact_sales`).
  * Tracks high-level operational metrics including order quantities, target thresholds, delivery statuses, and regional sales channels.
* **Advanced DAX Financial Tickers:**
  * Built dynamic string-concatenated KPI cards featuring real-time directional status arrows ($\blacktriangle$ / $\blacktriangledown$) and automated performance tracking.
  * Designed custom DAX measures for **Total Revenue**, **OTIF Rate %** (On-Time In-Full), and **Target Variance %**.
* **Dynamic Conditional Formatting:**
  * Implemented hex-code color status logic (`#107C10` for positive growth/targets met, `#A80000` for underperformance) mapped via Power BI's **Field value** conditional formatting rules.
* **Executive Matrix & Breakdown Analysis:**
  * Multi-dimensional pivot matrix displaying channel-wise and region-wise revenue distribution, variance percentages, and volume output.

---

## Tech Stack & Tools

* **Database:** MySQL / MySQL Workbench (Data Modeling, SQL Scripting, Relational Joins)
* **BI & Visualization:** Power BI Desktop (DAX Measures, Card Tickers, Matrix Visuals, Conditional Formatting)
* **Data Engineering:** Star schema modeling, primary/foreign key indexing, and custom metric modeling

---

## Database Setup Script (SQL)

```sql
-- Create Database
DROP DATABASE IF EXISTS fmcg_kpi_reporting;
CREATE DATABASE fmcg_kpi_reporting;
USE fmcg_kpi_reporting;

-- Create Dimension Tables
CREATE TABLE dim_date (
    date_key DATE PRIMARY KEY,
    day_of_week VARCHAR(15),
    month_name VARCHAR(15),
    month_num INT,
    quarter VARCHAR(5),
    year INT
);

CREATE TABLE dim_product (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    unit_price DECIMAL(10,2)
);

-- Create Fact Table
CREATE TABLE fact_sales (
    order_id INT PRIMARY KEY,
    date_key DATE,
    product_id INT,
    region VARCHAR(50),
    channel VARCHAR(50),
    quantity_sold INT,
    target_quantity INT,
    delivery_status VARCHAR(20),
    order_fulfilled_in_full TINYINT,
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (product_id) REFERENCES dim_product(product_id)
);

-- Seed Sample Data (DML)
INSERT INTO dim_date VALUES 
('2026-10-01', 'Thursday', 'October', 10, 'Q4', 2026),
('2026-10-02', 'Friday', 'October', 10, 'Q4', 2026);

INSERT INTO dim_product VALUES 
(101, 'Energy Drink 250ml', 'Beverages', 45.00),
(102, 'Packaged Snacks 100g', 'Foods', 30.00);
-- Total Revenue Measure
Total Revenue = 
SUMX(
    fact_sales, 
    fact_sales[quantity_sold] * RELATED(dim_product[unit_price])
)

-- Target Fulfillment Rate (OTIF %) Measure
OTIF Rate % = 
DIVIDE(
    CALCULATE(COUNTROWS(fact_sales), fact_sales[delivery_status] = "On-Time" && fact_sales[order_fulfilled_in_full] = 1),
    COUNTROWS(fact_sales),
    0
) * 100

-- Sales Target Variance % Measure
Target Variance % = 
DIVIDE(
    SUM(fact_sales[quantity_sold]) - SUM(fact_sales[target_quantity]),
    SUM(fact_sales[target_quantity]),
    0
) * 100

-- Revenue Ticker with Directional Indicators
Revenue_Ticker = 
VAR CurrentRev = [Total Revenue]
VAR TargetRev = SUM(fact_sales[target_quantity]) * 65
VAR VariancePct = DIVIDE(CurrentRev - TargetRev, TargetRev, 0)
VAR Arrow = IF(VariancePct >= 0, " ▲ (+", " ▼ (")
RETURN
    "₹ " & FORMAT(CurrentRev, "#,##0.00") & 
    " | Target Var: " & Arrow & FORMAT(VariancePct, "0.0%") & ")"

-- OTIF Fulfillment Ticker
OTIF_Ticker = 
VAR CurrentOTIF = [OTIF Rate %]
VAR Arrow = IF(CurrentOTIF >= 85, " ▲ (Target Met)", " ▼ (Below Target)")
RETURN
    "OTIF Rate: " & FORMAT(CurrentOTIF, "0.0") & "% | Fulfillment Health: " & Arrow

-- Dynamic Color Status Logic (Revenue)
Revenue_Color_Status = 
VAR VarianceVal = [Target Variance %]
RETURN
    IF(VarianceVal >= 0, "#107C10", "#A80000")

-- Dynamic Color Status Logic (OTIF)
OTIF_Color_Status = 
VAR CurrentOTIF = [OTIF Rate %]
RETURN
    IF(CurrentOTIF >= 85, "#107C10", "#A80000")

INSERT INTO fact_sales VALUES 
(5001, '2026-10-01', 101, 'East', 'E-commerce', 600, 550, 'On-Time', 1),
(5002, '2026-10-02', 102, 'North', 'Modern Trade', 400, 420, 'Delayed', 0);
