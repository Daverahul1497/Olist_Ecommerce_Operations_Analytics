# Olist E-commerce Operations Analytics — Consolidated SQL Reference

This reference reproduces the source SQL with explanatory commentary. 

## 01. Database setup and environment checks

Create the database if absent, select the schema, and inspect the active database and local_infile setting. The source tables have already been imported using the import wizard.

```sql
CREATE DATABASE IF NOT EXISTS ecommerce_analytics;

USE ecommerce_analytics;

SELECT DATABASE();

SHOW GLOBAL VARIABLES LIKE 'local_infile';
```

## 02. Raw source-table row counts

Count records in all nine imported Olist tables. 

```sql
SELECT 'customers' AS table_name, COUNT(*) AS row_count FROM customers
UNION ALL
SELECT 'geolocation', COUNT(*) FROM geolocation
UNION ALL
SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL
SELECT 'order_payments', COUNT(*) FROM order_payments
UNION ALL
SELECT 'order_reviews', COUNT(*) FROM order_reviews
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'products', COUNT(*) FROM products
UNION ALL
SELECT 'sellers', COUNT(*) FROM sellers
UNION ALL
SELECT 'product_category_name_translation', COUNT(*)
FROM product_category_name_translation;
```

## 03. Source-table schema inspection

Inspect data types and field names before interpreting date and numeric strings.

```sql
DESCRIBE orders;
DESCRIBE order_items;
DESCRIBE order_payments;
DESCRIBE order_reviews;
DESCRIBE products;
```

## 04. Sample records and initial missing-date checks

Sample core transactional tables and identify missing or blank purchase, customer-delivery, and estimated-delivery timestamps.

```sql
-- 1. Inspect sample order records
SELECT *
FROM orders
LIMIT 5;

-- 2. Inspect sample order-item records
SELECT *
FROM order_items
LIMIT 5;

-- 3. Inspect sample payment records
SELECT *
FROM order_payments
LIMIT 5;

-- 4. Inspect sample review records
SELECT *
FROM order_reviews
LIMIT 5;

-- 5. Check missing or blank order dates
SELECT
    COUNT(*) AS total_orders,
    SUM(order_purchase_timestamp IS NULL
        OR TRIM(order_purchase_timestamp) = '') AS missing_purchase_dates,
    SUM(order_delivered_customer_date IS NULL
        OR TRIM(order_delivered_customer_date) = '') AS missing_delivery_dates,
    SUM(order_estimated_delivery_date IS NULL
        OR TRIM(order_estimated_delivery_date) = '') AS missing_estimated_dates
FROM orders;
```

## 05. Delivery-date completeness by order status

Compare timestamp completeness across statuses; undelivered orders can legitimately lack a customer delivery date.

```sql
SELECT
    order_status,
    COUNT(*) AS total_orders,
    SUM(
        order_delivered_customer_date IS NULL
        OR TRIM(order_delivered_customer_date) = ''
    ) AS missing_delivery_dates
FROM orders
GROUP BY order_status
ORDER BY total_orders DESC;
```

## 06. Delivered-order punctuality baseline

Measure on-time and late deliveries from delivered orders. 

```sql
SELECT
    COUNT(*) AS delivered_orders,
    SUM(
        CAST(order_delivered_customer_date AS DATETIME)
        <= CAST(order_estimated_delivery_date AS DATETIME)
    ) AS delivered_on_or_before_estimate,
    SUM(
        CAST(order_delivered_customer_date AS DATETIME)
        > CAST(order_estimated_delivery_date AS DATETIME)
    ) AS delivered_late,
    ROUND(
        100.0 * SUM(
            CAST(order_delivered_customer_date AS DATETIME)
            <= CAST(order_estimated_delivery_date AS DATETIME)
        ) / COUNT(*),
        2
    ) AS on_time_delivery_pct
FROM orders
WHERE order_status = 'delivered'
  AND order_delivered_customer_date IS NOT NULL
  AND TRIM(order_delivered_customer_date) <> ''
  AND order_estimated_delivery_date IS NOT NULL
  AND TRIM(order_estimated_delivery_date) <> '';
```

## 07. Delivery-status anomaly investigations

Review delivered orders missing customer delivery dates and cancelled orders with delivery timestamps. These are investigations, not instructions to recode records.

```sql
SELECT
    order_id,
    order_status,
    order_purchase_timestamp,
    order_approved_at,
    order_delivered_carrier_date,
    order_delivered_customer_date,
    order_estimated_delivery_date
FROM orders
WHERE order_status = 'delivered'
  AND (
      order_delivered_customer_date IS NULL
      OR TRIM(order_delivered_customer_date) = ''
  )
ORDER BY order_purchase_timestamp;

SELECT
    order_id,
    order_status,
    order_purchase_timestamp,
    order_delivered_carrier_date,
    order_delivered_customer_date,
    order_estimated_delivery_date
FROM orders
WHERE order_status = 'canceled'
  AND order_delivered_customer_date IS NOT NULL
  AND TRIM(order_delivered_customer_date) <> ''
ORDER BY order_purchase_timestamp;
```

## 08. Order-item numeric data-quality checks

Check missing or invalid price/freight values and calculate item-value and freight totals. Item value excludes freight and is not recognised accounting revenue.

```sql
SELECT
    COUNT(*) AS total_order_items,

    SUM(
        price IS NULL OR TRIM(price) = ''
    ) AS missing_prices,

    SUM(
        freight_value IS NULL OR TRIM(freight_value) = ''
    ) AS missing_freight_values,

    SUM(
        CASE
            WHEN TRIM(price) REGEXP '^[0-9]+(\\.[0-9]+)?$'
            THEN 0
            ELSE 1
        END
    ) AS invalid_price_formats,

    SUM(
        CASE
            WHEN TRIM(freight_value) REGEXP '^[0-9]+(\\.[0-9]+)?$'
            THEN 0
            ELSE 1
        END
    ) AS invalid_freight_formats,

    ROUND(SUM(CAST(price AS DECIMAL(12,2))), 2)
        AS total_item_value,

    ROUND(SUM(CAST(freight_value AS DECIMAL(12,2))), 2)
        AS total_freight_value

FROM order_items;
```

## 09. Item sales value by product category

Aggregate item price by product category, using English translation where available.

```sql
SELECT
    COALESCE(
        translation.product_category_name_english,
        products.product_category_name,
        'unknown'
    ) AS product_category,

    COUNT(*) AS total_order_items,

    COUNT(DISTINCT order_items.order_id) AS total_orders,

    ROUND(SUM(CAST(order_items.price AS DECIMAL(12,2))), 2)
        AS total_item_value,

    ROUND(SUM(CAST(order_items.freight_value AS DECIMAL(12,2))), 2)
        AS total_freight_value,

    ROUND(AVG(CAST(order_items.price AS DECIMAL(12,2))), 2)
        AS average_item_price

FROM order_items

LEFT JOIN products
    ON order_items.product_id = products.product_id

LEFT JOIN product_category_name_translation AS translation
    ON products.product_category_name =
       translation.product_category_name

GROUP BY
    COALESCE(
        translation.product_category_name_english,
        products.product_category_name,
        'unknown'
    )

ORDER BY total_item_value DESC

LIMIT 15;
```

## 10. Product/category join-cardinality validation

Check whether joins to products and translations duplicate or lose order-item records.

```sql
SELECT
    (SELECT COUNT(*)
     FROM order_items) AS original_order_items,

    (SELECT COUNT(*)
     FROM order_items AS oi
     LEFT JOIN products AS p
        ON oi.product_id = p.product_id) AS items_after_products_join,

    (SELECT COUNT(*)
     FROM order_items AS oi
     LEFT JOIN products AS p
        ON oi.product_id = p.product_id
     LEFT JOIN product_category_name_translation AS t
        ON p.product_category_name = t.product_category_name
    ) AS items_after_translation_join,

    (SELECT COUNT(DISTINCT product_id)
     FROM products) AS distinct_product_ids,

    (SELECT COUNT(*)
     FROM products) AS total_product_rows,

    (SELECT COUNT(DISTINCT product_category_name)
     FROM product_category_name_translation)
        AS distinct_translated_categories,

    (SELECT COUNT(*)
     FROM product_category_name_translation)
        AS total_translation_rows;
```

## 11. Monthly item sales and freight trends

Summarise item prices and freight by purchase month; consider sparse/incomplete tail months when interpreting trends.

```sql
SELECT
    DATE_FORMAT(
        STR_TO_DATE(order_purchase_timestamp, '%Y-%m-%d %H:%i:%s'),
        '%Y-%m'
    ) AS purchase_month,

    COUNT(DISTINCT order_id) AS total_orders,

    COUNT(*) AS total_order_items,

    ROUND(
        SUM(CAST(price AS DECIMAL(12,2))),
        2
    ) AS total_item_value,

    ROUND(
        SUM(CAST(freight_value AS DECIMAL(12,2))),
        2
    ) AS total_freight_value

FROM orders
JOIN order_items USING (order_id)

WHERE order_purchase_timestamp IS NOT NULL
  AND TRIM(order_purchase_timestamp) <> ''

GROUP BY purchase_month
ORDER BY purchase_month;
```

## 12. Monthly order-status analysis

Group current order status by purchase month; these are purchase cohorts, not historical month-end snapshots.

```sql
SELECT
    DATE_FORMAT(
        STR_TO_DATE(order_purchase_timestamp, '%Y-%m-%d %H:%i:%s'),
        '%Y-%m'
    ) AS purchase_month,

    COUNT(*) AS total_orders,

    SUM(order_status = 'delivered') AS delivered_orders,

    SUM(order_status = 'canceled') AS canceled_orders,

    SUM(order_status = 'shipped') AS shipped_orders,

    SUM(order_status NOT IN ('delivered', 'canceled', 'shipped'))
        AS other_status_orders

FROM orders

WHERE order_purchase_timestamp IS NOT NULL
  AND TRIM(order_purchase_timestamp) <> ''

GROUP BY purchase_month
ORDER BY purchase_month;
```

## 13. Monthly cancellation rates

Calculate cancellations by purchase cohort. 

```sql
SELECT
    DATE_FORMAT(
        STR_TO_DATE(order_purchase_timestamp, '%Y-%m-%d %H:%i:%s'),
        '%Y-%m'
    ) AS purchase_month,

    COUNT(*) AS total_orders,

    SUM(order_status = 'delivered') AS delivered_orders,

    SUM(order_status = 'canceled') AS canceled_orders,

    ROUND(
        100.0 * SUM(order_status = 'delivered') / COUNT(*),
        2
    ) AS delivered_status_pct,

    ROUND(
        100.0 * SUM(order_status = 'canceled') / COUNT(*),
        2
    ) AS cancellation_rate_pct

FROM orders

WHERE order_purchase_timestamp IS NOT NULL
  AND TRIM(order_purchase_timestamp) <> ''

GROUP BY purchase_month
ORDER BY purchase_month;
```

## 14. Monthly delivery performance and duration

Analyse delivery timeliness and duration by purchase month.

```sql
SELECT
    DATE_FORMAT(
        STR_TO_DATE(order_purchase_timestamp, '%Y-%m-%d %H:%i:%s'),
        '%Y-%m'
    ) AS purchase_month,

    COUNT(*) AS eligible_delivered_orders,

    SUM(
        STR_TO_DATE(order_delivered_customer_date, '%Y-%m-%d %H:%i:%s')
        <= STR_TO_DATE(order_estimated_delivery_date, '%Y-%m-%d %H:%i:%s')
    ) AS on_time_orders,

    SUM(
        STR_TO_DATE(order_delivered_customer_date, '%Y-%m-%d %H:%i:%s')
        > STR_TO_DATE(order_estimated_delivery_date, '%Y-%m-%d %H:%i:%s')
    ) AS late_orders,

    ROUND(
        100.0 * SUM(
            STR_TO_DATE(order_delivered_customer_date, '%Y-%m-%d %H:%i:%s')
            <= STR_TO_DATE(order_estimated_delivery_date, '%Y-%m-%d %H:%i:%s')
        ) / COUNT(*),
        2
    ) AS on_time_delivery_pct,

    ROUND(
        AVG(
            DATEDIFF(
                STR_TO_DATE(order_delivered_customer_date, '%Y-%m-%d %H:%i:%s'),
                STR_TO_DATE(order_purchase_timestamp, '%Y-%m-%d %H:%i:%s')
            )
        ),
        2
    ) AS avg_days_to_deliver,

    ROUND(
        AVG(
            CASE
                WHEN STR_TO_DATE(order_delivered_customer_date, '%Y-%m-%d %H:%i:%s')
                     > STR_TO_DATE(order_estimated_delivery_date, '%Y-%m-%d %H:%i:%s')
                THEN DATEDIFF(
                    STR_TO_DATE(order_delivered_customer_date, '%Y-%m-%d %H:%i:%s'),
                    STR_TO_DATE(order_estimated_delivery_date, '%Y-%m-%d %H:%i:%s')
                )
            END
        ),
        2
    ) AS avg_days_late_when_late

FROM orders

WHERE order_status = 'delivered'
  AND order_purchase_timestamp IS NOT NULL
  AND TRIM(order_purchase_timestamp) <> ''
  AND order_delivered_customer_date IS NOT NULL
  AND TRIM(order_delivered_customer_date) <> ''
  AND order_estimated_delivery_date IS NOT NULL
  AND TRIM(order_estimated_delivery_date) <> ''

GROUP BY purchase_month
ORDER BY purchase_month;
```

## 15. Delivery eligibility and monthly reconciliation

Check date completeness, date parsing, and whether monthly eligible-order counts reconcile to the overall eligible population.

```sql
-- Overall delivery population
SELECT
    COUNT(*) AS delivered_status_orders,

    SUM(
        order_delivered_customer_date IS NOT NULL
        AND TRIM(order_delivered_customer_date) <> ''
    ) AS orders_with_customer_delivery_date,

    SUM(
        order_estimated_delivery_date IS NOT NULL
        AND TRIM(order_estimated_delivery_date) <> ''
    ) AS orders_with_estimated_delivery_date,

    SUM(
        order_purchase_timestamp IS NOT NULL
        AND TRIM(order_purchase_timestamp) <> ''
        AND order_delivered_customer_date IS NOT NULL
        AND TRIM(order_delivered_customer_date) <> ''
        AND order_estimated_delivery_date IS NOT NULL
        AND TRIM(order_estimated_delivery_date) <> ''
    ) AS eligible_by_nonblank_dates

FROM orders
WHERE order_status = 'delivered';


-- Check whether any required date fails to parse
SELECT
    COUNT(*) AS eligible_by_nonblank_dates,

    SUM(
        STR_TO_DATE(
            order_purchase_timestamp, '%Y-%m-%d %H:%i:%s'
        ) IS NULL
    ) AS invalid_purchase_timestamps,

    SUM(
        STR_TO_DATE(
            order_delivered_customer_date, '%Y-%m-%d %H:%i:%s'
        ) IS NULL
    ) AS invalid_delivery_timestamps,

    SUM(
        STR_TO_DATE(
            order_estimated_delivery_date, '%Y-%m-%d %H:%i:%s'
        ) IS NULL
    ) AS invalid_estimated_timestamps

FROM orders
WHERE order_status = 'delivered'
  AND order_purchase_timestamp IS NOT NULL
  AND TRIM(order_purchase_timestamp) <> ''
  AND order_delivered_customer_date IS NOT NULL
  AND TRIM(order_delivered_customer_date) <> ''
  AND order_estimated_delivery_date IS NOT NULL
  AND TRIM(order_estimated_delivery_date) <> '';

WITH monthly_counts AS (
    SELECT
        DATE_FORMAT(
            STR_TO_DATE(
                order_purchase_timestamp, '%Y-%m-%d %H:%i:%s'
            ),
            '%Y-%m'
        ) AS purchase_month,
        COUNT(*) AS eligible_orders
    FROM orders
    WHERE order_status = 'delivered'
      AND order_purchase_timestamp IS NOT NULL
      AND TRIM(order_purchase_timestamp) <> ''
      AND order_delivered_customer_date IS NOT NULL
      AND TRIM(order_delivered_customer_date) <> ''
      AND order_estimated_delivery_date IS NOT NULL
      AND TRIM(order_estimated_delivery_date) <> ''
    GROUP BY purchase_month
)

SELECT
    (SELECT COUNT(*)
     FROM orders
     WHERE order_status = 'delivered'
       AND order_purchase_timestamp IS NOT NULL
       AND TRIM(order_purchase_timestamp) <> ''
       AND order_delivered_customer_date IS NOT NULL
       AND TRIM(order_delivered_customer_date) <> ''
       AND order_estimated_delivery_date IS NOT NULL
       AND TRIM(order_estimated_delivery_date) <> ''
    ) AS overall_eligible_orders,

    SUM(eligible_orders) AS sum_of_monthly_eligible_orders,

    (SELECT COUNT(*)
     FROM monthly_counts
     WHERE purchase_month IS NULL
    ) AS null_month_groups

FROM monthly_counts;
```

## 16. Create and validate order-level analytical view

Create the order-grain Power BI view, with parsed dates and delivery-performance classifications; validate category counts.

```sql
CREATE OR REPLACE VIEW vw_order_level_analytics AS
SELECT
    order_id,
    customer_id,
    order_status,

    STR_TO_DATE(
        order_purchase_timestamp,
        '%Y-%m-%d %H:%i:%s'
    ) AS purchase_datetime,

    DATE_FORMAT(
        STR_TO_DATE(
            order_purchase_timestamp,
            '%Y-%m-%d %H:%i:%s'
        ),
        '%Y-%m'
    ) AS purchase_month,

    STR_TO_DATE(
        order_delivered_carrier_date,
        '%Y-%m-%d %H:%i:%s'
    ) AS carrier_handover_datetime,

    STR_TO_DATE(
        order_delivered_customer_date,
        '%Y-%m-%d %H:%i:%s'
    ) AS customer_delivery_datetime,

    STR_TO_DATE(
        order_estimated_delivery_date,
        '%Y-%m-%d %H:%i:%s'
    ) AS estimated_delivery_datetime,

    CASE
        WHEN order_status = 'delivered'
         AND TRIM(COALESCE(order_delivered_customer_date, '')) <> ''
         AND TRIM(COALESCE(order_estimated_delivery_date, '')) <> ''
        THEN DATEDIFF(
            STR_TO_DATE(
                order_delivered_customer_date,
                '%Y-%m-%d %H:%i:%s'
            ),
            STR_TO_DATE(
                order_estimated_delivery_date,
                '%Y-%m-%d %H:%i:%s'
            )
        )
        ELSE NULL
    END AS days_vs_estimated_delivery,

    CASE
        WHEN order_status <> 'delivered'
            THEN 'Not delivered'
        WHEN TRIM(COALESCE(order_delivered_customer_date, '')) = ''
          OR TRIM(COALESCE(order_estimated_delivery_date, '')) = ''
            THEN 'Unknown - missing date'
        WHEN STR_TO_DATE(
            order_delivered_customer_date,
            '%Y-%m-%d %H:%i:%s'
        ) <= STR_TO_DATE(
            order_estimated_delivery_date,
            '%Y-%m-%d %H:%i:%s'
        )
            THEN 'On time'
        ELSE 'Late'
    END AS delivery_performance

FROM orders;

SELECT
    delivery_performance,
    COUNT(*) AS order_count
FROM vw_order_level_analytics
GROUP BY delivery_performance
ORDER BY order_count DESC;
```

## 17. Create and validate order-item analytical view

Create the item-grain Power BI view with category enrichment, item price, and freight; validate its totals.

```sql
CREATE OR REPLACE VIEW vw_order_item_analytics AS
SELECT
    oi.order_id,
    oi.order_item_id,
    oi.product_id,
    oi.seller_id,

    o.customer_id,
    o.order_status,
    o.purchase_datetime,
    o.purchase_month,
    o.delivery_performance,
    o.days_vs_estimated_delivery,

    COALESCE(
        pct.product_category_name_english,
        p.product_category_name,
        'unknown'
    ) AS product_category,

    CAST(oi.price AS DECIMAL(12,2)) AS item_price,
    CAST(oi.freight_value AS DECIMAL(12,2)) AS freight_value,

    CAST(oi.price AS DECIMAL(12,2))
      + CAST(oi.freight_value AS DECIMAL(12,2))
      AS item_plus_freight_value

FROM order_items oi

JOIN vw_order_level_analytics o
    ON oi.order_id = o.order_id

LEFT JOIN products p
    ON oi.product_id = p.product_id

LEFT JOIN product_category_name_translation pct
    ON p.product_category_name = pct.product_category_name;


SELECT
    COUNT(*) AS total_order_items,
    COUNT(DISTINCT order_id) AS distinct_orders,
    COUNT(DISTINCT product_id) AS distinct_products,
    ROUND(SUM(item_price), 2) AS total_item_value,
    ROUND(SUM(freight_value), 2) AS total_freight_value
FROM vw_order_item_analytics;
```

## 18. Final join-quality checks

Check order-item links to order, product, and category data. 

```sql
SELECT
    COUNT(*) AS total_order_items,

    SUM(o.order_id IS NULL) AS missing_order_matches,

    SUM(p.product_id IS NULL) AS missing_product_matches,

    SUM(
        oi.product_id IS NOT NULL
        AND p.product_id IS NOT NULL
        AND TRIM(COALESCE(p.product_category_name, '')) <> ''
        AND TRIM(COALESCE(pct.product_category_name_english, '')) = ''
    ) AS products_without_english_category,

    SUM(
        TRIM(COALESCE(oi.price, '')) = ''
    ) AS missing_prices,

    SUM(
        TRIM(COALESCE(oi.freight_value, '')) = ''
    ) AS missing_freight_values

FROM order_items oi

LEFT JOIN vw_order_level_analytics o
    ON oi.order_id = o.order_id

LEFT JOIN products p
    ON oi.product_id = p.product_id

LEFT JOIN product_category_name_translation pct
    ON p.product_category_name = pct.product_category_name;
```
