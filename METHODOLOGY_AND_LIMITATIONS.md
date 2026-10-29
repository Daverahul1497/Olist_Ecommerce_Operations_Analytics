# Methodology and Limitations

## Analytical approach

MySQL was used to inspect source tables, validate raw fields, compute descriptive aggregations, reconcile delivery populations, and create order-level and item-level analytical views. Power BI imports those views and relates them one-to-many on `order_id`.

## Delivery-status populations

The validated view breakdown is:

| Delivery performance | Orders |
|---|---:|
| On time | 88,644 |
| Late | 7,826 |
| Not delivered | 2,963 |
| Unknown - missing date | 8 |
| **Total** | **99,441** |

`On time` + `Late` = **96,470** orders eligible for punctuality reporting. Eight additional orders have status `delivered` but lack a customer-delivery date. Thus, the number of orders whose **current status** is `delivered` is 96,478; the **classified delivery baseline** is 96,470. The punctuality rates are 88,644 / 96,470 = 91.89% and 7,826 / 96,470 = 8.11%.

The `Not delivered` classification includes all non-delivered order statuses; it does not measure lateness.

## Timestamp exceptions

- Eight delivered-status orders have no customer-delivery timestamp.
- Six cancelled-status orders contain carrier/customer delivery timestamps.
- Raw date fields may be text and require `STR_TO_DATE` or casts.
- A missing date must not be silently imputed or treated as an on-time/late delivery.

## Value definitions

**Item sales value** sums item prices (R$13,591,643.70) and **excludes freight** (R$2,251,909.54). Neither amount is necessarily recognised accounting revenue or profit. Care is needed when joining payments, reviews and items because their grains can differ.

## Temporal analysis

Monthly cancellation analysis groups the **current** status of each order by its purchase month. It is not a month-end historical snapshot. September and October 2018 contain only 16 and 4 orders respectively in the status query, so they should not be used as representative full months. Other late-tail-month patterns require context.

## Delivery duration and punctuality

Average delivery duration (12.50 days) measures elapsed time for the eligible population used by the analysis. Delivery speed and punctuality are distinct: a shorter average delivery time does not necessarily imply a higher on-time rate, because promised/estimated delivery dates vary.

## Grain and duplication risk

`vw_order_level_analytics` has one row per order. `vw_order_item_analytics` has one row per order item. Summing order-level attributes over item-level rows can duplicate orders. Use distinct-order counts or the order-level view for order KPIs.

## Execution and reproducibility

The consolidated SQL assumes that the nine raw tables have been imported into MySQL schema `ecommerce_analytics` using the import wizard. It runs diagnostic queries and replaces two views using `CREATE OR REPLACE VIEW`. It does not import raw CSVs, create raw table schemas, install Power BI, or configure credentials.

Reference metrics are previously validated project outputs, **not newly executed results** from this repository package. Recheck them after changing input data, SQL or dashboard filters.

## Interpretation limits

Findings are descriptive. They identify periods, categories and operational exceptions worth investigating, but they do not prove root causes or support causal claims.
