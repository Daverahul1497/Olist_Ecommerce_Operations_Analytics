# Business Insights and Recommendations

This page separates **observed dataset results** from **potential follow-up actions**. The recommendations are investigation priorities, not proven causal remedies.

## 1. Delivery punctuality

**Observation:** Among 96,470 orders with a classified delivery outcome, 88,644 (91.89%) were on time and 7,826 (8.11%) were late.

**Suggested follow-up:** Segment late deliveries by purchase month, seller, product category and geography where available; investigate concentration before proposing operational interventions.

## 2. Monthly variability

**Observation:** March 2018 had a reported on-time rate of 78.64%, with 1,496 late deliveries. June 2018 reached 98.64% on-time. August 2018 averaged 7.66 delivery days but had an 89.61% on-time rate.

**Suggested follow-up:** Examine estimated delivery promises, seller handover timing and logistics patterns. Faster delivery does not automatically mean better punctuality against the promised date.

## 3. Category contribution

**Observation:** Health & Beauty (R$1,258,681.34), Watches & Gifts (R$1,205,005.68), and Bed, Bath & Table (R$1,036,988.68) were leading item-value categories.

**Suggested follow-up:** Compare category contribution with late-delivery exposure, seller concentration and order volumes. High item value alone does not establish profitability.

## 4. Demand seasonality

**Observation:** November 2017 recorded the highest monthly order count (7,544) and approximately R$1.01m in item value.

**Suggested follow-up:** Compare high-volume periods with delivery performance and fulfilment duration, controlling for changes in cohort composition.

## 5. Cancellation and data quality

**Observation:** Cancellation rates are based on current status within purchase-month cohorts. Eight delivered orders lack customer-delivery timestamps, and six cancelled orders have delivery timestamps.

**Suggested follow-up:** Track these exceptions explicitly in a data-quality monitoring view, and avoid using sparse tail months as evidence of sustained cancellation trends.

## Decision-use summary

The dataset supports prioritising deeper analysis of delivery exceptions, punctuality variation and category contribution. It does **not** establish the underlying reasons for delays, cancellation changes or category performance without further investigation.
