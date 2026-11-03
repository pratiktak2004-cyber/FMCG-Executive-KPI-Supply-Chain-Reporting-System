# FMCG Executive KPI & Supply Chain Reporting System

An end-to-end business intelligence and enterprise reporting platform built for Fast-Moving Consumer Goods (FMCG) operations. This project establishes an automated pipeline from a normalized MySQL relational database star schema to an executive Power BI dashboard featuring dynamic DAX financial tickers, conditional color-status formatting, and multi-dimensional supply chain tracking.

---

## Key Features & Architecture

* **Relational Database Design (MySQL):** 
  * Structured using a clean star schema (`fmcg_kpi_reporting`) consisting of dimension tables (`Dim_Date`, `Dim_Product`) and fact tables (`Fact_Sales`).
  * Tracks high-level operational metrics including order quantities, target thresholds, delivery statuses, fulfillment flags, and regional sales channels.
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
-- 1. Drop existing tables if they exist to start fresh
DROP TABLE IF EXISTS Fact_Sales;
DROP TABLE IF EXISTS Dim_Product;
DROP TABLE IF EXISTS Dim_Date;

-- 2. Create Database
CREATE DATABASE IF NOT EXISTS fmcg_kpi_reporting;
USE fmcg_kpi_reporting;

-- 3. Create Date Dimension
CREATE TABLE Dim_Date (
    date_key DATE PRIMARY KEY,
    year INT,
    quarter VARCHAR(5),
    month_name VARCHAR(15),
    month_num INT,
    day_of_week VARCHAR(15)
);

-- 4. Create Product Dimension
CREATE TABLE Dim_Product (
    product_id VARCHAR(20) PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    sub_category VARCHAR(50),
    unit_cost DECIMAL(10,2),
    unit_price DECIMAL(10,2)
);

-- 5. Create Fact Sales Table
CREATE TABLE Fact_Sales (
    order_id VARCHAR(20) PRIMARY KEY,
    order_date DATE,
    delivery_date DATE,
    product_id VARCHAR(20),
    channel VARCHAR(50),            -- Modern Trade, General Trade, E-commerce
    region VARCHAR(50),             -- North, South, East, West
    quantity_sold INT,
    target_quantity INT,
    delivery_status VARCHAR(30),    -- On-Time, Delayed, Cancelled
    order_fulfilled_in_full TINYINT(1), -- 1 for Full, 0 for Partial
    FOREIGN KEY (order_date) REFERENCES Dim_Date(date_key),
    FOREIGN KEY (product_id) REFERENCES Dim_Product(product_id)
);

-- 6. Insert Product Data
INSERT INTO Dim_Product (product_id, product_name, category, sub_category, unit_cost, unit_price) VALUES
('PROD001', 'A2 Pure Banni Buffalo Milk 1L', 'Dairy & Beverage', 'Fresh Milk', 45.00, 65.00),
('PROD002', 'Sprouted Oat Muesli 500g', 'Packaged Foods', 'Breakfast Cereals', 120.00, 180.00),
('PROD003', 'Probiotic Greek Yogurt 200g', 'Dairy & Beverage', 'Yogurt', 35.00, 55.00),
('PROD004', 'Freeze-Dried Strawberry Mix 250g', 'Packaged Foods', 'Snacks', 150.00, 240.00);

-- 7. Insert Date Data
INSERT INTO Dim_Date (date_key, year, quarter, month_name, month_num, day_of_week) VALUES
('2026-10-01', 2026, 'Q4', 'October', 10, 'Thursday'),
('2026-10-02', 2026, 'Q4', 'October', 10, 'Friday'),
('2026-10-03', 2026, 'Q4', 'October', 10, 'Saturday'),
('2026-10-04', 2026, 'Q4', 'October', 10, 'Sunday'),
('2026-10-05', 2026, 'Q4', 'October', 10, 'Monday');

-- 8. Insert Sales Data
INSERT INTO Fact_Sales (order_id, order_date, delivery_date, product_id, channel, region, quantity_sold, target_quantity, delivery_status, order_fulfilled_in_full) VALUES
('ORD1001', '2026-10-01', '2026-10-02', 'PROD001', 'Modern Trade', 'North', 500, 480, 'On-Time', 1),
('ORD1002', '2026-10-01', '2026-10-03', 'PROD002', 'General Trade', 'West', 350, 400, 'Delayed', 0),
-- Total Revenue Measure
Total Revenue = 
SUMX(
    Fact_Sales, 
    Fact_Sales[quantity_sold] * RELATED(Dim_Product[unit_price])
)

-- Target Fulfillment Rate (OTIF %) Measure
OTIF Rate % = 
DIVIDE(
    CALCULATE(COUNTROWS(Fact_Sales), Fact_Sales[delivery_status] = "On-Time" && Fact_Sales[order_fulfilled_in_full] = 1),
    COUNTROWS(Fact_Sales),
    0
) * 100

-- Sales Target Variance % Measure
Target Variance % = 
DIVIDE(
    SUM(Fact_Sales[quantity_sold]) - SUM(Fact_Sales[target_quantity]),
    SUM(Fact_Sales[target_quantity]),
    0
) * 100

-- Revenue Ticker with Directional Indicators
Revenue_Ticker = 
VAR CurrentRev = [Total Revenue]
VAR TargetRev = SUM(Fact_Sales[target_quantity]) * 65
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
('ORD1003', '2026-10-02', '2026-10-03', 'PROD003', 'E-commerce', 'East', 600, 550, 'On-Time', 1),
('ORD1004', '2026-10-03', '2026-10-05', 'PROD004', 'Modern Trade', 'South', 200, 250, 'Delayed', 1),
('ORD1005', '2026-10-04', '2026-10-05', 'PROD001', 'E-commerce', 'West', 800, 750, 'On-Time', 1);
## ⚙️ Setup & Deployment Instructions

Follow these steps to set up the database and deploy the executive reporting dashboard:

### 1. Database Initialization
* Open **MySQL Workbench** or your preferred MySQL client.
* Create a new SQL tab, paste the complete **Database Setup Script** provided above, and execute it.
* This will automatically drop existing tables, recreate the `fmcg_kpi_reporting` database, build the star schema tables (`Dim_Date`, `Dim_Product`, `Fact_Sales`), and populate them with seed data.

### 2. Power BI Integration
* Open **Power BI Desktop**.
* Navigate to **Get Data > MySQL database**.
* Enter your server instance credentials and import the three core relational tables:
  * `Dim_Date`
  * `Dim_Product`
  * `Fact_Sales`

### 3. Dashboard & Report Configuration
* **Model Relationships:** Verify that `Fact_Sales` is correctly linked to `Dim_Date` (`order_date` $\rightarrow$ `date_key`) and `Dim_Product` (`product_id` $\rightarrow$ `product_id`).
* **DAX Measures:** Right-click the `Fact_Sales` table in Power BI, select **New measure**, and paste the custom DAX formulas for revenue tickers, OTIF fulfillment health, and dynamic color status logic.
* **Visual Mapping:** Drop the ticker measures into modern Card visuals and map the matrix view using `channel` rows, `region` columns, and your performance metrics.
