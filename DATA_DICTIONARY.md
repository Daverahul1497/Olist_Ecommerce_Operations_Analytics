# Data Dictionary and KPI Definitions

## Source entities

| Table | Analytical role |
|---|---|
| `orders` | Order status, purchase date, delivery dates and estimated delivery |
| `order_items` | Item-grain price, freight, product and seller references |
| `products` | Product category and attributes |
| `product_category_name_translation` | Portuguese-to-English category labels |
| `customers` | Customer identifiers and attributes |
| `sellers` | Seller identifiers and attributes |
| `order_payments` | Payment records |
| `order_reviews` | Review records |
| `geolocation` | Geographic reference records |

## Analytical views

| View | Grain | Important columns |
|---|---|---|
| `vw_order_level_analytics` | One row per order | `order_id`, `customer_id`, `order_status`, `purchase_datetime`, `purchase_month`, `carrier_handover_datetime`, `customer_delivery_datetime`, `estimated_delivery_datetime`, `days_vs_estimated_delivery`, `delivery_performance` |
| `vw_order_item_analytics` | One row per order item | `order_id`, `order_item_id`, `product_id`, `seller_id`, `customer_id`, `order_status`, `purchase_datetime`, `purchase_month`, `delivery_performance`, `days_vs_estimated_delivery`, `product_category`, `item_price`, `freight_value`, `item_plus_freight_value` |

## Business metrics

| Metric | Operational meaning | Reference |
|---|---|---:|
| Total orders | Orders in the source population | 99,441 |
| Delivered-status orders | Orders whose current `order_status` is `delivered`, including missing-date exceptions | 96,478 |
| Delivered orders with classified punctuality | Delivered-status orders classified `On time` or `Late` | 96,470 |
| On-time delivered orders | Delivered and classified `On time` by the analytical view | 88,644 |
| Late delivered orders | Delivered and classified `Late` | 7,826 |
| On-time rate | On-time / (on-time + late) eligible delivered orders | 91.89% |
| Late rate | Late / (on-time + late) eligible delivered orders | 8.11% |
| Unknown delivery status | Delivered orders with missing required date | 8 |
| Not delivered | All non-delivered statuses | 2,963 |
| Item sales value | Sum of item prices, excluding freight | R$13,591,643.70 |
| Freight value | Sum of item freight values | R$2,251,909.54 |
| Average delivery days | Mean delivery duration for eligible delivered records | 12.50 |
| Cancellation rate by month | Current cancelled statuses within each purchase-month cohort | Depends on cohort |

**Delivery denominator:** The on-time and late rates refer to **96,470 classified delivered orders with known punctuality** (88,644 + 7,826). The additional eight delivered orders with missing dates are not counted in these two punctuality categories. The overall current delivered-status count is **96,478** when those eight unknowns are included. This distinction is essential when reconciling SQL view classifications.

**Date logic:** `days_vs_estimated_delivery` uses `DATEDIFF` (calendar-date difference); negative = earlier calendar date, zero = same calendar date, positive = later calendar date. The view's `delivery_performance` classification instead compares full parsed timestamps. An order can therefore have `days_vs_estimated_delivery = 0` but be classified `Late` if delivered after the estimated time on that date. Check the SQL for query-specific eligibility rules.

**Category fallback:** English translation where present, otherwise original category label, otherwise `unknown`.

**Do not interpret** item sales value as accounting revenue, profit or total payment value.
