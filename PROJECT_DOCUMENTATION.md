# Project Documentation

## Overview

This portfolio project translates raw Brazilian e-commerce transactions into operational performance indicators. It combines source-level SQL validation, two reusable analytical views, and a Power BI report with **Executive Overview** and **Delivery & Fulfilment** pages.

## Objectives and stakeholders

The report is intended for operations managers and business analysts evaluating order activity, fulfilment reliability, item sales value and cancellations. It supports descriptive performance monitoring and identification of areas for further investigation.

## Data sources

The public Olist Brazilian E-Commerce dataset contains nine tables: `customers`, `geolocation`, `order_items`, `order_payments`, `order_reviews`, `orders`, `products`, `sellers`, and `product_category_name_translation`.

| Source table | Validated rows |
|---|---:|
| customers | 99,441 |
| geolocation | 1,000,163 |
| order_items | 112,650 |
| order_payments | 103,886 |
| order_reviews | 99,224 |
| orders | 99,441 |
| products | 32,951 |
| sellers | 3,095 |
| product_category_name_translation | 71 |

## SQL analytical workflow

The [consolidated SQL script](SQL/Olist_Consolidated_SQL_Reference.sql) is arranged in **18 logical sections**, from database setup through join-quality validation. It covers schema inspection, source counts, sample data, timestamp completeness, delivery baseline, status anomalies, numeric checks, category and monthly aggregations, date-eligibility reconciliation, two analytical view definitions, and final join checks.

The script preserves exploratory and reconciliation queries that answer distinct questions. An identical duplicate row-count query was removed. The raw Olist tables are not created by this script; they must already be imported.

## Reporting model

### `vw_order_level_analytics`

One row per order, including identifiers, current order status, parsed purchase and delivery dates, purchase month, `days_vs_estimated_delivery`, and `delivery_performance`. Classification includes `On time`, `Late`, `Not delivered`, and `Unknown - missing date`.

### `vw_order_item_analytics`

One row per order item, enriched with order-level attributes and translated product category where available. It includes `item_price`, `freight_value`, and `item_plus_freight_value`. The view joins to the order-level view and product/category lookup tables.

**Grain control:** Counts of orders must come from the order-level grain or use distinct order identifiers; joining orders to multiple items can inflate order-level totals.

## Dashboard pages

**Executive Overview:** KPI summary and high-level trends for orders, item sales value, delivery and order health.

**Delivery & Fulfilment:** punctuality, delivery duration and monthly fulfilment performance.

## Validated outcomes

- 99,441 orders, including 96,470 delivered orders with classified punctuality (plus eight delivered-status orders with missing delivery dates).
- 88,644 on-time and 7,826 late delivered orders in the validated delivery baseline.
- 91.89% on-time and 8.11% late.
- R$13,591,643.70 item sales value, excluding R$2,251,909.54 freight.
- Average delivery time: 12.50 days.
- November 2017: peak monthly order count of 7,544 and approximately R$1.01m item sales value.

### Top product categories by item sales value

| Category | Item sales value |
|---|---:|
| Health & Beauty | R$1,258,681.34 |
| Watches & Gifts | R$1,205,005.68 |
| Bed, Bath & Table | R$1,036,988.68 |
| Sports & Leisure | R$988,048.97 |
| Computers & Accessories | R$911,954.32 |

## Quality and interpretation

The order-level classification identifies **8 delivered orders with missing customer-delivery timestamps** and **6 cancelled orders with carrier/customer delivery timestamps**. Neither anomaly should be silently recoded.

The monthly cancellation query groups present-day status by purchase month, so it is not a historical status snapshot. Sparse late-2018 months must be interpreted cautiously. For details, see [Methodology and Limitations](METHODOLOGY_AND_LIMITATIONS.md).

## Reproducibility and scope

The SQL is intended for MySQL 8.0 and a previously populated `ecommerce_analytics` schema. Power BI connects to the two analytical views. Credentials and raw data are intentionally excluded. SQL and PBIX should be refreshed and reconciled when the source extract changes.

## Conclusion

The project demonstrates end-to-end operations analytics: validation, business metric definition, reusable SQL views, model-grain awareness, Power BI visualisation, and transparent documentation of limitations.
