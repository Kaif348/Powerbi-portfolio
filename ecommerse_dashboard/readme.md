# Olist Brazilian E-Commerce — Sales & Operations Dashboard

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Data Modelling](https://img.shields.io/badge/Data%20Modelling-Star%20Schema-green?style=for-the-badge)
![Dataset](https://img.shields.io/badge/Dataset-Kaggle-20BEFF?style=for-the-badge&logo=kaggle&logoColor=white)

---

## Project Overview

This project is an end-to-end **Business Intelligence dashboard** built in **Power BI** using the real-world [Olist Brazilian E-Commerce dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) from Kaggle.

Olist is the largest department store in Brazilian marketplaces — operating similarly to Amazon. The dataset contains **100,000+ real commercial transactions** from 2016 to 2018 across multiple Brazilian states, covering everything from order placement to customer reviews.

The goal of this project was to transform 9 raw CSV files into a fully interactive, multi-page business intelligence report that answers critical business questions across sales, product performance, delivery operations, and customer satisfaction.

---

## Business Questions Answered

### Sales & Revenue
- What is the total Gross Merchandise Value (GMV) of the platform?
- How did revenue grow year over year from 2016 to 2018?
- Which months drive peak revenue and why?
- What is the average order value across the platform?

### Product & Category Performance
- Which product categories generate the most revenue?
- Which categories have the highest customer satisfaction?
- Which categories have the worst review scores — and why?
- Is there a relationship between freight cost and customer satisfaction?
- Which categories are trending upward in 2018?

### Delivery & Operations
- What is the average time from order placement to delivery?
- Which Brazilian states have the slowest delivery performance?
- What percentage of orders arrive early, on time, or late?
- Are delivery times improving or deteriorating over time?
- Which months experience the most delivery delays?

### Customer & Seller Analysis
- How many unique customers does Olist have?
- What percentage of customers are repeat buyers?
- Which states generate the most customers and revenue?
- How does payment method affect customer satisfaction?

---

## Dashboard Pages

### Page 1 — Executive Overview
High-level KPIs and platform-wide performance metrics for business stakeholders.

**Visuals:**
- 5 KPI Cards — Total GMV, Total Orders, Total Customers, Avg Review Score, On Time Delivery Rate
- Multi-line chart — Monthly Revenue Trend by Year (2016, 2017, 2018)
- Column chart — Total Orders by Year showing platform growth
- Donut chart — Order Status Breakdown (Delivered, Shipped, Canceled, Others)
- Filled Map — Revenue by Brazilian State
- Year slicer — filter entire dashboard by year

**Key Insight:** Olist grew from 329 orders in Q4 2016 to 54,011 orders in 2018 — a 180x growth in just 2 years.

---

### Page 2 — Product & Category Analysis
Deep dive into product category performance, freight costs, and customer satisfaction by category.

**Visuals:**
- Horizontal bar chart — Top 10 Categories by Revenue
- Horizontal bar chart — Categories with Lowest Customer Satisfaction (conditional formatting)
- Horizontal bar chart — Categories with Highest Freight Value
- Area chart — Category Revenue Trends over Months (top 5 categories)
- 3 KPI Cards — Top Revenue Category, Worst Rated Category, Highest Freight Category

**Key Insight:** Health & Beauty is the #1 revenue category. Security & Services has the lowest customer satisfaction score. Computers have the highest average freight cost at R$48 per order.

---

### Page 3 — Delivery Performance
Operational analysis of delivery times, delays, and regional fulfillment performance.

**Visuals:**
- 5 KPI Cards — Avg Delivery Delay, Early Orders, Late Orders, On Time Rate, Avg Fulfillment Days
- Donut chart — Delivery Status Distribution (Early/On Time/Late)
- Line chart — Monthly Delivery Delay Trend
- Bar chart — Average Delivery Days by State (all 27 states)
- Bar chart — States with Lowest On Time Delivery Rate (conditional formatting)

**Key Insight:** Remote northern states like RR (Roraima) and AP (Amapá) take 27+ days for delivery while SP (São Paulo) averages just 8 days — a 19 day gap highlighting a major logistics inequality across Brazil.

---

## Data Model

### Dataset Source
[Kaggle — Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

### Schema Type
**Hybrid Star-Snowflake Schema** — `order_items` serves as the central fact table surrounded by dimension tables, with some dimensions further normalized (snowflaked).

### Tables & Relationships

```
                    CUSTOMERS
                        |
                        | Many to 1
                        |
ORDER_REVIEWS ——— ORDERS ——— ORDER_PAYMENTS
     (1:M)          |              (1:M)
                    | 1:M
                    |
               ORDER_ITEMS ——— SELLERS
               (FACT TABLE)       |
                    |         geolocation
                    | M:1
                    |
                PRODUCTS
                    |
                    | M:1
                    |
          CATEGORY_TRANSLATION
```

### Table Descriptions

| Table | Type | Rows | Key Columns |
|---|---|---|---|
| `orders` | Fact (core) | 99,441 | order_id (PK), customer_id (FK), order_status, timestamps |
| `order_items` | Fact (transactions) | 112,650 | order_id (FK), product_id (FK), seller_id (FK), price, freight_value |
| `order_payments` | Fact (payments) | 103,886 | order_id (FK), payment_type, payment_value |
| `order_reviews` | Fact (satisfaction) | 99,224 | order_id (FK), review_score, review_creation_date |
| `customers` | Dimension | 99,441 | customer_id (PK), customer_unique_id, customer_state |
| `products` | Dimension | 32,951 | product_id (PK), product_category_name (FK) |
| `sellers` | Dimension | 3,095 | seller_id (PK), seller_state |
| `category_translation` | Lookup | 71 | product_category_name (PK), product_category_name_english |
| `geolocation` | Lookup | 1,000,163 | geolocation_zip_code_prefix, lat, lng, state |

### Key Data Modelling Decisions

1. **customer_id vs customer_unique_id** — Identified that `customer_id` is a per-order identifier while `customer_unique_id` represents the actual person. Used `customer_unique_id` for all customer count measures to avoid overcounting repeat buyers.

2. **Geolocation table deduplication** — The geolocation table contained multiple lat/lng readings per zip code prefix. Removed duplicates in Power Query to enable clean Many-to-1 relationships.

3. **Category translation join** — Connected `products[product_category_name]` to `category_translation[product_category_name]` to translate all Portuguese category names to English throughout the dashboard.

4. **Bidirectional cross-filtering** — Implemented bidirectional filtering on selected relationships to enable category-level analysis across the full table chain from `product_category` through to `order_reviews`.

5. **Archive table removal** — Identified and removed an accidentally imported Windows metadata table (`archive`) that was creating a corrupting Many-to-Many relationship with the products table.

---

## Data Cleaning (Power Query)

### Orders Table
- Converted all timestamp columns to correct Date/Time data types
- Created `delivery_delay` calculated column:
  ```
  = Duration.Days([order_estimated_delivery_date] - [order_delivered_customer_date])
  ```
  Positive = arrived early, Negative = arrived late

- Created `total_order_completion_time` calculated column:
  ```
  = Duration.Days([order_delivered_customer_date] - [order_purchase_timestamp])
  ```

### Customers Table
- Identified that `customer_city` values are all lowercase — noted for future normalization

### Products Table
- Left null `product_category_name` values as null — Power BI excludes them from calculations automatically, avoiding misleading "Unknown" category inflation

### Geolocation Table
- Removed duplicates on `geolocation_zip_code_prefix` to enable clean relationships

### Order Reviews Table
- Removed `review_comment_title` and `review_comment_message` columns — Portuguese text not needed for quantitative analysis

### Category Translation Table
- Used as a lookup to replace Portuguese category names with English equivalents across all visuals

---

## DAX Measures

### Sales & Revenue Measures

```dax
Total GMV = 
SUM(order_items[price]) + SUM(order_items[freight_value])

Total Revenue = 
SUM(order_items[price])

Total Freight Cost = 
SUM(order_items[freight_value])

Avg Order Value = 
DIVIDE([Total GMV], [Total Orders])

Freight Rate = 
DIVIDE(SUM(order_items[freight_value]), SUM(order_items[price]))
```

### Order Measures

```dax
Total Orders = 
DISTINCTCOUNT(orders[order_id])

Delivered Orders = 
CALCULATE([Total Orders], orders[order_status] = "delivered")

Cancelled Orders = 
CALCULATE([Total Orders], orders[order_status] = "canceled")

Cancellation Rate = 
DIVIDE([Cancelled Orders], [Total Orders])
```

### Customer Measures

```dax
Total Customers = 
DISTINCTCOUNT(customers[customer_unique_id])

Revenue per Customer = 
DIVIDE([Total GMV], [Total Customers])
```

### Delivery Performance Measures

```dax
Avg Fulfillment Days = 
AVERAGE(orders[total_order_completion_time])

Avg Delivery Delay = 
AVERAGE(orders[delivery_delay])

On Time Delivery Rate = 
DIVIDE(
    CALCULATE(
        [Total Orders],
        orders[Delivery Status] = "Early" 
            || orders[Delivery Status] = "On Time"
    ),
    [Total Orders]
)

Late Orders = 
CALCULATE([Total Orders], orders[Delivery Status] = "Late")

Early Orders = 
CALCULATE([Total Orders], orders[Delivery Status] = "Early")

On Time Rate % = 
ROUND([On Time Delivery Rate] * 100, 1)
```

### Customer Satisfaction Measures

```dax
Avg Review Score = 
AVERAGE(order_reviews[review_score])

Avg Freight Cost = 
AVERAGE(order_items[freight_value])
```

### Category Intelligence Measures

```dax
Top Category = 
FIRSTNONBLANK(
    TOPN(
        1,
        VALUES(product_category[product_category_name_english]),
        CALCULATE([Total GMV]),
        DESC
    ),
    1
)

Worst Rated Category = 
FIRSTNONBLANK(
    TOPN(
        1,
        VALUES(product_category[product_category_name_english]),
        CALCULATE(AVERAGE(order_reviews[review_score])),
        ASC
    ),
    1
)

Highest Freight Category = 
FIRSTNONBLANK(
    TOPN(
        1,
        VALUES(product_category[product_category_name_english]),
        CALCULATE(AVERAGE(order_items[freight_value])),
        DESC
    ),
    1
)
```

### DAX Calculated Columns

```dax
-- Delivery Status Classification
Delivery Status = 
IF(
    VALUE(orders[delivery_delay]) > 0, "Early",
    IF(
        VALUE(orders[delivery_delay]) >= -3, "On Time",
        "Late"
    )
)

-- Order Status Grouping
Order Status Grouped = 
SWITCH(
    TRUE(),
    orders[order_status] = "delivered", "Delivered",
    orders[order_status] = "shipped", "Shipped",
    orders[order_status] = "canceled", "Canceled",
    "Others"
)
```

---

## Key Business Insights

### 1. Extraordinary Platform Growth
Olist grew from **329 orders in Q4 2016** to **54,011 orders in 2018** — a 180x increase in just over 2 years. Note: 2016 data represents only September-December, so YoY comparisons require careful interpretation.

### 2. Health & Beauty Dominates
Health & Beauty is the #1 revenue category generating over **R$1.4M** in GMV — nearly double the second place category (Watches & Gifts at R$1.3M).

### 3. Delivery Inequality Across Brazil
There is a **19 day gap** between the slowest states (RR, AP — remote north, 27+ days) and fastest states (SP, PR — southeast, ~8 days). This represents a significant customer experience inequality requiring regional logistics investment.

### 4. 87.68% of Orders Arrive Early
The vast majority of Olist orders arrive **before** the estimated delivery date — demonstrating conservative delivery estimates that create positive customer surprises. Only 7.63% of orders arrive late.

### 5. Security & Services Has Critical Satisfaction Issues
With an average review score significantly below the platform average, the Security & Services category requires immediate product quality investigation.

### 6. Freight Costs Hurt Heavy Categories
Computers and home appliances have average freight costs of R$45-50 per order — nearly 15-20% of product price — which likely contributes to lower repeat purchase rates in these categories.

### 7. November Peak Revenue
Monthly trend analysis reveals a consistent revenue spike in **November** across 2017 and 2018 — driven by Brazilian Black Friday — suggesting Olist should prioritize seller inventory and logistics capacity in Q4 every year.

### 8. 96,096 Unique Real Customers
While the dataset shows 99,441 customer IDs, there are only 96,096 unique customers — meaning approximately **3,345 customers placed more than one order**, representing a repeat purchase rate of ~3.5%.

---

## Tools & Technologies

| Tool | Usage |
|---|---|
| **Power BI Desktop** | Dashboard development, visualizations |
| **Power Query (M Language)** | Data cleaning, transformation, calculated columns |
| **DAX** | Measures, calculated columns, KPI logic |
| **Kaggle** | Dataset source |
| **GitHub** | Portfolio hosting |

---

## Project Structure

```
olist-ecommerce-dashboard/
│
├── README.md                          ← This file
├── screenshots/
│   ├── 01_executive_overview.png
│   ├── 02_product_category.png
│   ├── 03_delivery_performance.png
|    
└── Dataset/
    └── all nine csv files          
```

---

## How to Use This Dashboard

1. Download the dataset from [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) or from Dataset folder of this repository.
2. Extract all 9 CSV files
3. Open the `.pbix` file in Power BI Desktop
4. Update data source paths to point to your local CSV files
5. Refresh the data
6. Use the Year and State slicers to filter all visuals interactively

---

## Dashboard Screenshots
Screenshots of all pages of the power bi dashboard are uploaded to this repo in screenshots folders.

---

## What I Learned

- **Multi-table data modelling** — building star and snowflake schemas from raw CSV files
- **Relationship management** — handling Many-to-Many relationships, bidirectional filtering, and ambiguous filter paths
- **DAX fundamentals** — CALCULATE, DIVIDE, SWITCH, FIRSTNONBLANK, TOPN, AVERAGEX
- **Power Query data cleaning** — Duration.Days calculations, deduplication, data type corrections
- **Business intelligence thinking** — translating raw data into actionable business insights
- **Dashboard design principles** — consistent color themes, visual hierarchy, conditional formatting
- **Real-world data challenges** — handling Portuguese language data, incomplete 2016 data, null values

---

## Author

**Kaif ul Wara**

- LinkedIn: linkedin.com/in/kaifulwara19
- Email: kaifulwara348@gmail.com

---

## License

This project uses publicly available data from Kaggle under their dataset terms of use. The dashboard and analysis are original work.

---

## ⭐ If you found this project useful, please star the repository!

---

*Dataset: Brazilian E-Commerce Public Dataset by Olist — 100,000 orders, 2016-2018, Brazil*
