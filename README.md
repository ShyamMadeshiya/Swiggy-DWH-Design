# Swiggy Data Warehouse Design & Business Analytics

## 📌 Overview

This project presents a data warehouse design for a Swiggy-style food delivery platform. It models the complete order lifecycle, from customers browsing restaurants and viewing menus to adding items to the cart, placing orders, making payments, and receiving deliveries.

The design supports analytics across restaurant performance, customer behavior, menu item sales, delivery operations, rider utilization, order cancellations, refunds, and platform revenue.

The project focuses on dimensional data modeling, fact and dimension table relationships, Slowly Changing Dimension Type 2 (SCD Type 2), and SQL-based business metrics.

> **Disclaimer:** This is an independent educational portfolio project inspired by a food delivery platform. It is not Swiggy's official data model and does not represent its internal architecture.

## 🎯 Project Objectives

- Design a data warehouse model for food delivery analytics.
- Define fact tables, dimension tables, primary keys, and foreign key relationships.
- Model the customer journey from app sessions and menu impressions to cart activity and orders.
- Track restaurant acceptance, food preparation, rider assignment, and delivery performance.
- Apply SCD Type 2 to preserve historical changes in customer, restaurant, menu item, and delivery partner attributes.
- Analyze gross merchandise value (GMV), average order value (AOV), and platform take rate.
- Measure on-time delivery, order cancellation, refund, and repeat customer rates.
- Evaluate restaurant acceptance rates, rider utilization, and order preparation SLA performance.

## 📁 Repository Structure

```text
Swiggy-DWH-Design/
│
├── README.md
│
├── 01_Docs/
│   └── DWH_Data_Modelling.pdf
│
└── 02_Architecture/
    └── SwiggyDWH.png
```

### 📄 File Description

- **README.md** — Provides the project overview, objectives, data architecture, fact and dimension tables, and business metrics.
- **01_Docs/DWH_Data_Modelling.pdf** — Contains reference documentation for the Swiggy data warehouse design, table definitions, relationships, and SQL queries for business metrics.
- **02_Architecture/SwiggyDWH.png** — Contains the final Swiggy data warehouse ER diagram, illustrating table structures, attributes, primary keys, foreign keys, and relationships between entities.

## 🏗️ Data Architecture (Star Schema)

The data warehouse follows a dimensional modeling approach, separating descriptive business entities into dimension tables and measurable activities into fact tables.

The model covers the complete food delivery journey, including app sessions, menu interactions, order processing, restaurant acceptance, rider assignments, delivery trips, payments, cancellations, refunds, restaurant availability, and rider shifts.

### Dimension Tables (Context & Attributes)

Dimension tables store descriptive information about customers, restaurants, menu items, delivery partners, and other business entities.

| Dimension Table | Purpose |
|---|---|
| `dim_date` | Calendar attributes such as year, quarter, month, week, day, weekends, and period boundaries. |
| `dim_time` | Time attributes including hour, minute, second, AM/PM, 24-hour format, and day part. |
| `dim_geo` | Country, region, state, city, market, timezone, latitude, and longitude. |
| `dim_customer` | Customer details, loyalty tier, preferred currency and language, marketing preferences, segment, and risk flag. Uses SCD Type 2. |
| `dim_device` | Device type, operating system, app or web platform, browser, and version details. |
| `dim_channel` | Marketing source, medium, campaign, ad group, advertisement, UTM attributes, and partner information. |
| `dim_restaurant` | Restaurant name, brand, primary cuisine, rating, location, commission band, chain status, and operating hours. Uses SCD Type 2. |
| `dim_menu_item` | Menu item name, category, vegetarian flag, spiciness, base price, calories, and allergens. Uses SCD Type 2. |
| `dim_delivery_partner` | Rider details, vehicle type, license number, geographic information, and joining date. Uses SCD Type 2. |
| `dim_payment_method` | Payment method, card network, wallet type, issuer country, and 3-D Secure support. |
| `dim_promo` | Promotion code, discount type and value, validity dates, and targeting rules. |
| `dim_fee_policy` | Fee policy name, calculation type, configuration parameters, and validity period. |
| `dim_currency_rate` | Currency code, reporting exchange rate, exchange-rate source, and timestamp. |

### Fact Tables (Events & Metrics)

Fact tables capture measurable activities and transactions throughout the food delivery lifecycle.

| Fact Table | Purpose |
|---|---|
| `fact_app_session` | Records app sessions, page views, add-to-cart events, and checkout events. |
| `fact_menu_impression` | Tracks restaurant and menu item impressions, including the price shown to customers. |
| `fact_add_to_cart` | Records cart additions with customer, restaurant, menu item, quantity, and item price. |
| `fact_order_header` | Stores order-level information, customer and restaurant keys, GMV, discounts, fees, taxes, delivery timestamps, take rate, and order status. |
| `fact_order_item` | Records individual items within an order, including quantity, line price, discounts, add-ons, vegetarian flag, and category. |
| `fact_assignment` | Tracks restaurant order offers, acceptance status, and acceptance timestamps. |
| `fact_delivery_trip` | Records rider assignments, pickup and drop-off events, distance, duration, and surge multiplier. |
| `fact_cancellation` | Captures order cancellations, cancellation party, reason, cancellation fee, and timestamp. |
| `fact_payment` | Records payment authorizations, captures, refunds, transaction types, and payment timestamps. |
| `fact_refund` | Tracks refund amounts, chargeback flags, reasons, and refund timestamps. |
| `fact_restaurant_availability` | Measures restaurant open minutes, outage minutes, and uptime percentage. |
| `fact_rider_shift` | Tracks rider shift duration, active trip time, and idle time. |

### Key Data Modeling Concepts

- **Fact and dimension modeling:** Separates business events and numerical measures from descriptive attributes.
- **Primary and foreign keys:** Connect customers, restaurants, menu items, orders, riders, and related events.
- **SCD Type 2:** Maintains historical versions of customer, restaurant, menu item, and delivery partner records.
- **Order header and order item:** Separates order-level financial and operational information from individual menu items.
- **Operational lifecycle tracking:** Connects order acceptance, preparation, rider assignment, pickup, and drop-off events.
- **Role-playing date and time dimensions:** Supports analysis of order, cancellation, refund, and other event timestamps.

## 📊 SQL Implementations for Business Metrics

The project documentation defines 10 business metrics for analyzing food delivery operations and commercial performance.

### 1. App-to-Order Conversion Rate

**Definition:** Percentage of unique app sessions that result in an order during the reporting period.

**Formula:** Distinct orders ÷ distinct sessions × 100.

```sql
WITH s AS (
    SELECT COUNT(DISTINCT session_id) AS sessions
    FROM fact_app_session fs
    JOIN dim_date d ON fs.date_key = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
o AS (
    SELECT COUNT(DISTINCT oh.order_id_nk) AS orders
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
      AND oh.status IN ('DELIVERED', 'PLACED', 'ACCEPTED')
)
SELECT
    s.sessions,
    o.orders,
    ROUND(
        o.orders * 100.0 / NULLIF(s.sessions, 0), 2
    ) AS app_to_order_conversion_percent
FROM s
CROSS JOIN o;
```

### 2. Average Order Value (AOV)

**Definition:** Average gross merchandise value per delivered order.

**Formula:** Total GMV ÷ number of delivered orders.

```sql
SELECT
    COUNT(*) AS orders,
    SUM(oh.gmv_amount) AS total_gmv,
    ROUND(
        SUM(oh.gmv_amount) / NULLIF(COUNT(*), 0), 2
    ) AS average_order_value
FROM fact_order_header oh
JOIN dim_date d ON oh.date_key_ordered = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
  AND oh.status = 'DELIVERED';
```

### 3. Platform Take Rate Percentage

**Definition:** Commercial revenue captured by the platform relative to GMV, after accounting for refunds.

**Formula:** Net revenue ÷ GMV × 100.

```sql
WITH base AS (
    SELECT
        SUM(oh.gmv_amount) AS gmv,
        SUM(
            oh.gmv_amount * oh.take_rate_pct
            + oh.platform_fee
            + oh.delivery_fee
        ) - COALESCE(SUM(rf.refund_amount), 0) AS net_revenue
    FROM fact_order_header oh
    LEFT JOIN fact_refund rf
        ON rf.order_id_nk = oh.order_id_nk
    JOIN dim_date d
        ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
      AND oh.status = 'DELIVERED'
)
SELECT
    gmv,
    net_revenue,
    ROUND(
        net_revenue * 100.0 / NULLIF(gmv, 0), 2
    ) AS platform_take_rate_percent
FROM base;
```

### 4. On-Time Delivery Rate (OTD)

**Definition:** Percentage of delivered orders whose actual drop-off time is on or before the promised drop-off time.

**Formula:** On-time deliveries ÷ delivered orders × 100.

```sql
SELECT
    COUNT(*) AS delivered_orders,
    SUM(
        CASE
            WHEN oh.actual_dropoff_ts <= oh.promised_dropoff_ts
            THEN 1 ELSE 0
        END
    ) AS on_time_orders,
    ROUND(
        SUM(
            CASE
                WHEN oh.actual_dropoff_ts <= oh.promised_dropoff_ts
                THEN 1 ELSE 0
            END
        ) * 100.0 / NULLIF(COUNT(*), 0), 2
    ) AS on_time_delivery_percent
FROM fact_order_header oh
JOIN dim_date d ON oh.date_key_ordered = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
  AND oh.status = 'DELIVERED';
```

### 5. Order Cancellation Rate

**Definition:** Percentage of placed orders that end in cancellation by the restaurant, rider, or customer.

**Formula:** Cancelled orders ÷ placed orders × 100.

```sql
WITH placed AS (
    SELECT COUNT(*) AS placed_orders
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
cancelled AS (
    SELECT COUNT(DISTINCT c.order_id_nk) AS cancelled_orders
    FROM fact_cancellation c
    JOIN dim_date d ON c.date_key_cancel = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    placed_orders,
    cancelled_orders,
    ROUND(
        cancelled_orders * 100.0 / NULLIF(placed_orders, 0), 2
    ) AS cancellation_rate_percent
FROM placed
CROSS JOIN cancelled;
```

### 6. Refund Rate

**Definition:** Percentage of placed orders with at least one full or partial refund recorded during the reporting period.

**Formula:** Refunded orders ÷ placed orders × 100.

```sql
WITH placed AS (
    SELECT COUNT(DISTINCT oh.order_id_nk) AS placed_orders
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
ref AS (
    SELECT COUNT(DISTINCT rf.order_id_nk) AS refunded_orders
    FROM fact_refund rf
    JOIN dim_date d ON rf.date_key_refund = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    placed_orders,
    refunded_orders,
    ROUND(
        refunded_orders * 100.0 / NULLIF(placed_orders, 0), 2
    ) AS refund_rate_percent
FROM placed
CROSS JOIN ref;
```

### 7. Repeat Customer Rate (90-Day Lookback)

**Definition:** Percentage of customers with at least two delivered orders during the last 90 days.

**Formula:** Customers with two or more delivered orders ÷ customers with at least one delivered order × 100.

```sql
WITH cust_orders AS (
    SELECT
        customer_sk,
        COUNT(DISTINCT order_id_nk) AS orders_90d
    FROM fact_order_header oh
    JOIN dim_date d ON oh.date_key_ordered = d.date_key
    WHERE d.cal_date BETWEEN date_sub(current_date(), 90)
                         AND current_date()
      AND oh.status = 'DELIVERED'
    GROUP BY customer_sk
)
SELECT
    COUNT(*) AS active_customers_90d,
    SUM(
        CASE WHEN orders_90d >= 2 THEN 1 ELSE 0 END
    ) AS repeat_customers_90d,
    ROUND(
        SUM(CASE WHEN orders_90d >= 2 THEN 1 ELSE 0 END)
        * 100.0 / NULLIF(COUNT(*), 0), 2
    ) AS repeat_customer_rate_percent
FROM cust_orders;
```

### 8. Restaurant Acceptance Rate

**Definition:** Percentage of order offers accepted by restaurants.

**Formula:** Accepted offers ÷ total offers × 100.

```sql
SELECT
    r.restaurant_id_nk,
    r.brand,
    COUNT(*) AS total_offers,
    SUM(
        CASE WHEN a.accepted_flag = TRUE THEN 1 ELSE 0 END
    ) AS accepted_offers,
    ROUND(
        SUM(CASE WHEN a.accepted_flag THEN 1 ELSE 0 END)
        * 100.0 / NULLIF(COUNT(*), 0), 2
    ) AS acceptance_rate_percent
FROM fact_assignment a
JOIN dim_restaurant r
    ON a.restaurant_sk = r.restaurant_sk
   AND r.is_current = TRUE
JOIN dim_date d ON a.date_key = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
GROUP BY r.restaurant_id_nk, r.brand
ORDER BY acceptance_rate_percent DESC;
```

### 9. Rider Utilization

**Definition:** Percentage of rider shift time spent in active trip states, including travel to pickup, pickup, and travel to drop-off.

**Formula:** Total active duration ÷ total shift duration × 100.

```sql
SELECT
    COUNT(*) AS shifts,
    SUM(rs.shift_duration_sec) AS total_shift_sec,
    SUM(rs.active_duration_sec) AS total_active_sec,
    ROUND(
        SUM(rs.active_duration_sec) * 100.0
        / NULLIF(SUM(rs.shift_duration_sec), 0), 2
    ) AS rider_utilization_percent
FROM fact_rider_shift rs
JOIN dim_date d ON rs.date_key = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date;
```

### 10. Order Preparation SLA Hit Rate

**Definition:** Percentage of accepted or delivered orders that were marked ready by the promised preparation time.

**Formula:** Preparation SLA hits ÷ accepted orders × 100.

```sql
SELECT
    COUNT(*) AS accepted_orders,
    SUM(
        CASE
            WHEN oh.ready_ts IS NOT NULL
             AND oh.ready_ts <= oh.promised_ready_ts
            THEN 1 ELSE 0
        END
    ) AS prep_sla_hits,
    ROUND(
        SUM(
            CASE
                WHEN oh.ready_ts IS NOT NULL
                 AND oh.ready_ts <= oh.promised_ready_ts
                THEN 1 ELSE 0
            END
        ) * 100.0 / NULLIF(COUNT(*), 0), 2
    ) AS prep_sla_hit_percent
FROM fact_order_header oh
JOIN dim_date d ON oh.date_key_ordered = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
  AND oh.status IN ('ACCEPTED', 'DELIVERED');
```

## 🛠️ Tools & Technologies

- **SQL:** Joins, aggregations, Common Table Expressions (CTEs), conditional expressions, date functions, and analytical calculations.
- **Data Modeling:** Entity Relationship Diagrams (ERDs), dimensional modeling, fact and dimension tables.
- **Data Warehousing:** Primary keys, foreign keys, historical data tracking, and SCD Type 2.
- **Business Analytics:** GMV, AOV, platform take rate, delivery SLA, cancellations, refunds, and customer retention.
- **Version Control:** Git and GitHub for project documentation and version management.

*These tools and concepts describe the documented design. The repository does not imply that a production warehouse or live food delivery pipeline has been deployed.*

## 🚀 How to Use This Repository

1. Review `02_Architecture/SwiggyDWH.png` to understand the data model and table relationships.
2. Read the reference PDF in `01_Docs/` for detailed table definitions and SQL implementations.
3. Review the business metric definitions and their corresponding SQL queries.
4. Replace `:start_date` and `:end_date` with appropriate reporting parameters for your SQL environment.
5. Validate the queries against a database containing compatible tables and column definitions before using the results for reporting.

## 🔮 Future Enhancements

- Add SQL DDL scripts for creating fact and dimension tables.
- Organize the business metric queries into separate `.sql` files.
- Create sample datasets for customers, restaurants, menu items, orders, and delivery trips.
- Add a data dictionary documenting table grains, columns, keys, and relationships.
- Implement data quality checks and referential integrity validation.
- Build dashboards for order trends, restaurant performance, delivery efficiency, and revenue analysis.
- Validate financial calculations against multiple refunds per order to avoid double-counting.

## 👨‍💻 Author

**Shyam Madeshiya**

Aspiring Data Engineer interested in SQL, data warehousing, dimensional modeling, and analytical data solutions.
