## Data Introduction

This analysis looks at **108,401 sessions** from the GA4 sample ecommerce dataset (**Nov 1–30, 2020**) to understand:

1. where users drop in the conversion journey,
2. which segments drive most of the loss, and
3. which journey type drives revenue.

Main result: the biggest problem is **early commitment** (after product view), and most revenue volume comes from **non-cart purchase journeys**. Cart-assisted journeys are smaller, but their orders are larger on average.

---

## Data at a Glance

* Dataset: `bigquery-public-data.ga4_obfuscated_sample_ecommerce`
* Period: 2020-11-01 to 2020-11-30
* Total events: **1,472,712**
* Users: **79,421**
* Reconstructed sessions: **108,401**
* Event types: **17**

Core fields used:

* `user_pseudo_id`
* `event_params.ga_session_id`
* `event_name`
* `event_timestamp`
* `ecommerce.transaction_id`
* `ecommerce.purchase_revenue`, `ecommerce.purchase_revenue_in_usd`
* `device.category`, `geo.country`
* `traffic_source.source`, `traffic_source.medium`
* `user_first_touch_timestamp`

For deeper schema and field notes, see **Data Introduction.md**.

---

## Simplified Important Info

### What we analyzed and why it matters

We modeled a simple shopping journey with 4 stages:

1. `view_item`
2. `add_to_cart`
3. `begin_checkout`
4. `purchase`

Then we checked two things:

* **Reach:** how many sessions touched each stage
* **Order:** how many sessions moved stage-to-stage in the expected order

Why this matters: this tells us where conversion is really breaking and where optimization will have the biggest business impact.

---

### 1) Biggest drop is right after product view

* Sessions with `view_item`: **25,943** (**23.93%** of all sessions)
* Ordered `view → cart`: **2,193**
* Step conversion `view → cart`: **8.45%**
* Leak after view: **91.55%**

What this means:

* The main issue is not “can users finish payment.”
* The main issue is “can we get product viewers to commit at all.”

Why leaders should care:

* This is the largest recoverable volume in the funnel.

**Chart Placeholder**
`[Chart 1: Reach vs Ordered Funnel Conversion]`

* Left: reach by stage
* Right: ordered conversion by step
* Goal: clearly show where the biggest loss happens

---

### 2) Cart is not a required step for most buyers

* Checkout without cart: **3,542** sessions (**77.25%** of checkout sessions)
* Purchase without cart: **1,188** sessions (**73.47%** of purchase sessions)
* Purchase without checkout: **2** sessions (**0.12%**)

What this means:

* Checkout is still the core gate.
* Cart is often skipped.
* A cart-first story misses how most purchases happen in this dataset.

Why leaders should care:

* If reporting or roadmap assumes “everyone goes through cart,” decisions will be off.

**Chart Placeholder**
`[Chart 2: Cart-Assisted vs Non-Cart Purchase Paths]`

* Type: Sankey or split-flow bars
* Goal: show that non-cart purchase is a major path, not a small exception

---

### 3) Segmenting helps prioritize effort

Pattern is consistent: high early leakage appears across device, country, and source groups.

Largest leakage volume drivers:

* `new_in_period` users: **80.51%** of total view-stage leakage
* United States: **43.88%** of total view-stage leakage

What this means:

* Root problem is broad.
* Prioritization should follow impact volume.

Why leaders should care:

* This tells teams where to focus first for maximum lift.

**Chart Placeholder**
`[Chart 3: Leakage Rate vs Leakage Volume by Segment]`

* X: leak rate after view
* Y: share of total leak volume
* Bubble size: sessions
* Goal: show “high-rate small segment” vs “medium-rate big-impact segment”

---

### 4) Revenue is driven by non-cart paths, value is stronger in cart-assisted paths

**Non-cart purchase path**

* Transactions: **1,240** (**73.20%** share)
* Deduped revenue: **86,029** (**71.08%** share)
* AOV: **69.38**

**Cart-assisted purchase path**

* Transactions: **452** (**26.68%** share)
* Deduped revenue: **35,005** (**28.92%** share)
* AOV: **77.44**

What this means:

* Non-cart path drives revenue volume.
* Cart-assisted path has higher order value.

Why leaders should care:

* You need two optimization tracks, not one:

  * non-cart flow for scale
  * cart-assisted flow for value per order

**Chart Placeholder**
`[Chart 4: Transaction Share and Revenue Share by Path]`
`[Chart 5: AOV by Path]`

---

### Practical reporting rule

Track performance with a **dual-path model** every week:

1. Cart-assisted conversion path
2. Non-cart conversion path

This avoids forcing all behavior into one linear funnel.

**Chart Placeholder**
`[Chart 6: Weekly Dual-Path KPI Scorecard]`

---

## More Detailed Analysis

### Event-level data snapshot

GA4 export is event-level, so one user session appears across multiple rows.

| event_date |  event_timestamp | user_pseudo_id | ga_session_id | event_name     | transaction_id | purchase_revenue |
| ---------- | ---------------: | -------------- | ------------: | -------------- | -------------- | ---------------: |
| 20201112   | 1605172200123456 | U_001          |    1605172200 | session_start  | null           |             null |
| 20201112   | 1605172210456789 | U_001          |    1605172200 | view_item      | null           |             null |
| 20201112   | 1605172240123456 | U_001          |    1605172200 | begin_checkout | null           |             null |
| 20201112   | 1605172260789012 | U_001          |    1605172200 | purchase       | T_98765        |            85.00 |
| 20201112   | 1605175500123000 | U_001          |    1605175500 | session_start  | null           |             null |

---

### Session reconstruction and funnel order logic

There is no top-level session id column, so sessions were rebuilt using:

* `ga_session_id` extracted from `event_params`
* `user_pseudo_id`
* `session_key = user_pseudo_id || '-' || ga_session_id`

For each session, first timestamp was taken for each funnel event:

* `ts_view`, `ts_cart`, `ts_checkout`, `ts_purchase`

Ordered transitions were counted only when time order was correct:

* `ts_cart >= ts_view`
* `ts_checkout >= ts_cart`
* `ts_purchase >= ts_checkout`

This avoids counting sessions that touched events but not in usable order.

---

### Sequential leakage result details

**Coverage**

* Total sessions: **108,401**
* Sessions with any funnel event: **26,006**
* View: **25,943**
* Cart: **2,201**
* Checkout: **4,585**
* Purchase: **1,617**

**Ordered transition performance**

* View → Cart: **2,193** (**8.45%**)
* Cart → Checkout: **849** (**38.57%**)
* Checkout → Purchase: **1,615** (**35.22%**)

**Bypass signals**

* Checkout without cart: **3,542** (**77.25% of checkout sessions**)
* Purchase without cart: **1,188** (**73.47% of purchase sessions**)
* Purchase without checkout: **2** (**0.12%**)

Interpretation:

* Early commitment is the dominant issue.
* Checkout is usually present before purchase.
* Cart is optional in many successful journeys.

---

### Segment-level findings

The same funnel logic was cut by:

* device
* top countries
* user type (`new_in_period`, `returning`)
* coarse first-touch source groups

Highlights:

* High view-stage leakage is consistent across segments.
* `new_in_period` users contribute most leakage volume (**80.51%**) due to larger base.
* US contributes the largest country share of leakage volume (**43.88%**).
* Device differences exist, but this is not a “mobile-only” or “desktop-only” issue.

---

### Path economics breakdown

Sessions were labeled into path families:

* `purchase_non_cart`
* `purchase_cart_assisted`
* `checkout_no_purchase_non_cart`
* `checkout_no_purchase_cart_assisted`
* `cart_no_checkout`
* `view_only`
* `other_or_no_funnel`

Revenue was measured two ways:

* event-level purchase revenue
* transaction-deduped revenue (used for core economics)

Why deduped revenue was used:

* purchase events can be duplicated
* transaction-level dedup gives a more stable financial view

Key read:

* non-cart purchase paths drive most revenue volume
* cart-assisted paths bring higher AOV
* biggest checkout-loss pool is non-cart checkout without purchase

---

### Interpretation limits and use rules

* Source-based cuts are coarse in this dataset; use them for direction, not precise attribution.
* Reach metrics and ordered transition metrics serve different purposes and should be read together.
* Path labels define some outcomes by construction; use conversion and revenue comparisons across families for decisions.
