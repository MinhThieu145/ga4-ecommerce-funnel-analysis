
# **GA4 Ecommerce Data Introduction**

## **Dataset Overview**

This dataset contains GA4 **event-level** data from the Google Merchandise Store (Google-branded ecommerce site), exported to BigQuery as a public sample dataset.

* **Source Website (official reference):** `shop.googlemerchandisestore.com`
* **Dataset:** `bigquery-public-data.ga4_obfuscated_sample_ecommerce`
* **Date Range:** **2020-11-01 to 2021-01-31** (3 months / 92 days)
* **Table Pattern:** Daily tables named `events_YYYYMMDD` (for example, `events_20201101`)
* **Implementation Context:** Web ecommerce implementation + enhanced measurement sample



## **Data Model**

GA4 uses an **event-based schema** in BigQuery export. In practice, analysis is done at event-record level, with many attributes stored in nested/repeated fields (especially `event_params`, `items`, `user_properties`).

A single user journey (session start → product view → add to cart → purchase) appears as multiple event records, which must be aggregated for session/funnel analysis.

Key modeling implication: GA4 is **not** session-table-first; you build sessions/funnels from events.

## **Obfuscation and Sample Constraints**

This is an **obfuscated** public sample. Some fields include placeholder values like:

* `<Other>`
* `NULL`
* `''` (empty string)

Because of obfuscation, internal consistency is somewhat limited, and this dataset should not be expected to match the GA demo property exactly.



## **Core Fields (High-Value for Analysis)**

### **Event Identification**

* `event_date` (STRING, `YYYYMMDD`)
  Date the event was logged (property timezone basis).
* `event_timestamp` (INTEGER, microseconds UTC)
  Time the event was **received by GA**.
* `event_name` (STRING)
  Event type (e.g., `session_start`, `page_view`, `view_item`, `add_to_cart`, `begin_checkout`, `purchase`, etc.).



### **User Identification**

* `user_pseudo_id` (STRING)
  Pseudonymous user identifier (core ID for user-level analysis). Most likely based on browser cookies. So it depends on the devide / browser
* `user_id` (STRING)
  Optional authenticated/custom ID for login user. For this data the field is always NULL (deleted by Google)
* `user_first_touch_timestamp` (INTEGER)
  Microsecond timestamp for first touch/open/visit.



### **Device Information**

Inside `device`:

* `device.category` (mobile/tablet/desktop)
* `device.operating_system`
* `device.operating_system_version`
* `device.language`
* `device.web_info.browser`
* `device.web_info.browser_version`
* `device.web_info.hostname`



### **Geography**

Inside `geo`:

* `geo.continent`
* `geo.sub_continent`
* `geo.country`
* `geo.region`
* `geo.metro`
* `geo.city`

Derived from IP-based location context in export.



### **Stream / Platform**

* `stream_id` (STRING): GA4 data stream ID
* `platform` (STRING): stream platform (`Web`, `IOS`, `Android`)

For this specific sample (web ecommerce implementation), analysis is generally web-focused.




## **Traffic Source: Important Scope Differences**


### 1) **User-level first acquisition**

* `traffic_source.name`
* `traffic_source.medium`
* `traffic_source.source`

These represent the traffic source that **first acquired the user** and do not update for later campaigns.
(First-touch user attribution scope.)

### 2) **Event-collected source context**

* `collected_traffic_source.*`
  Contains source/medium/campaign and click IDs captured with collected events (manual params/referrer parsing, plus IDs like `gclid`, `dclid`, `srsltid`).

### 3) **Session last-click context**

* `session_traffic_source_last_click.*`
  Session-level last-click attribution context (manual + ads integrations where available).

So:

* Use `traffic_source.*` for **user acquisition** perspective
* Use session/event traffic fields for **session acquisition / channeling** perspective




## **Ecommerce Fields**

Inside `ecommerce`:

* `ecommerce.total_item_quantity`
* `ecommerce.purchase_revenue_in_usd`
* `ecommerce.purchase_revenue`
* `ecommerce.refund_value_in_usd`
* `ecommerce.refund_value`
* `ecommerce.shipping_value_in_usd`
* `ecommerce.shipping_value`
* `ecommerce.tax_value_in_usd`
* `ecommerce.tax_value`
* `ecommerce.transaction_id`
* `ecommerce.unique_items`

And item-level details live in repeated `items` (e.g., `items.item_id`, `items.item_name`, `items.item_brand`, categories, quantity, price, etc.).




## **Privacy / Consent Fields**

Inside `privacy_info`:

* `privacy_info.ads_storage`
* `privacy_info.analytics_storage`
* `privacy_info.uses_transient_token`

Possible values are **Yes / No / Unset**.




## **Nested Field Access**

GA4 export uses nested + repeated records, so dot notation and `UNNEST()` are standard.

```sql
SELECT
  device.category,
  geo.country,
  ecommerce.purchase_revenue
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_20201101`;
```

For repeated fields like `event_params` and `items`, use `UNNEST(...)`.




## **Session Identification**

GA4 does not provide a top-level `session_id` column in the event table.
Session ID is typically extracted from `event_params` key `ga_session_id`.

A practical unique session key is:

* `user_pseudo_id` + `ga_session_id`

All events with that combination belong to the same session window (subject to GA sessionization behavior).




## **Known Limitations (Non-Completeness)**

1. **Obfuscation artifacts** can affect raw field fidelity (`<Other>`, `NULL`, empty string).
2. **Traffic-source interpretation depends on scope** (user first-touch vs session/event context).
3. **Nested/repeated schema** requires careful SQL (`UNNEST`, dedup/session logic).
4. **Sample dataset behavior is representative but not identical to production property outputs**.
5. **Not intended as a perfect mirror** of the GA demo account.


```
| # | field_path | data_type |
|---|------------|-----------|
| 1 | user_pseudo_id | STRING |
| 2 | user_properties.value.string_value | INT64 |
| 3 | user_properties.value.set_timestamp_micros | INT64 |
| 4 | user_properties.value.int_value | INT64 |
| 5 | user_properties.value.float_value | INT64 |
| 6 | user_properties.value.double_value | INT64 |
| 7 | user_properties.value | STRUCT<string_value INT64, int_value INT64, float_value INT64, double_value INT64, set_timestamp_micros INT64> |
| 8 | user_properties.key | INT64 |
| 9 | user_properties | ARRAY<STRUCT<key INT64, value STRUCT<string_value INT64, int_value INT64, float_value INT64, double_value INT64, set_timestamp_micros INT64>>> |
| 10 | user_ltv.revenue | FLOAT64 |
| 11 | user_ltv.currency | STRING |
| 12 | user_ltv | STRUCT<revenue FLOAT64, currency STRING> |
| 13 | user_id | STRING |
| 14 | user_first_touch_timestamp | INT64 |
| 15 | traffic_source.source | STRING |
| 16 | traffic_source.name | STRING |
| 17 | traffic_source.medium | STRING |
| 18 | traffic_source | STRUCT<medium STRING, name STRING, source STRING> |
| 19 | stream_id | INT64 |
| 20 | privacy_info.uses_transient_token | STRING |
| 21 | privacy_info.analytics_storage | INT64 |
| 22 | privacy_info.ads_storage | INT64 |
| 23 | privacy_info | STRUCT<analytics_storage INT64, ads_storage INT64, uses_transient_token STRING> |
| 24 | platform | STRING |
| 25 | items.quantity | INT64 |
| 26 | items.promotion_name | STRING |
| 27 | items.promotion_id | STRING |
| 28 | items.price_in_usd | FLOAT64 |
| 29 | items.price | FLOAT64 |
| 30 | items.location_id | STRING |
| 31 | items.item_variant | STRING |
| 32 | items.item_revenue_in_usd | FLOAT64 |
| 33 | items.item_revenue | FLOAT64 |
| 34 | items.item_refund_in_usd | FLOAT64 |
| 77 | ecommerce.total_item_quantity | INT64 |
| 78 | ecommerce.tax_value_in_usd | FLOAT64 |
| 79 | ecommerce.tax_value | FLOAT64 |
| 80 | ecommerce.shipping_value_in_usd | FLOAT64 |
| 81 | ecommerce.shipping_value | FLOAT64 |
| 82 | ecommerce.refund_value_in_usd | FLOAT64 |
| 83 | ecommerce.refund_value | FLOAT64 |
| 84 | ecommerce.purchase_revenue_in_usd | FLOAT64 |
| 85 | ecommerce.purchase_revenue | FLOAT64 |
| 86 | ecommerce | STRUCT<total_item_quantity INT64, purchase_revenue_in_usd FLOAT64, purchase_revenue FLOAT64, refund_value_in_usd FLOAT64, refund_value FLOAT64, shipping_value_in_usd FLOAT64, shipping_value FLOAT64, tax_value_in_usd FLOAT64, tax_value FLOAT64, unique_items INT64, transaction_id STRING> |
| 87 | device.web_info.browser_version | STRING |
| 88 | device.web_info.browser | STRING |
| 89 | device.web_info | STRUCT<browser STRING, browser_version STRING> |
| 90 | device.vendor_id | INT64 |
| 91 | device.time_zone_offset_seconds | INT64 |
| 92 | device.operating_system_version | STRING |
| 93 | device.operating_system | STRING |
| 94 | device.mobile_os_hardware_model | INT64 |
| 95 | device.mobile_model_name | STRING |
| 96 | device.mobile_marketing_name | STRING |
| 97 | device.mobile_brand_name | STRING |
| 98 | device.language | STRING |
| 99 | device.is_limited_ad_tracking | STRING |
| 100 | device.category | STRING |
| 101 | device.advertising_id | INT64 |
| 102 | device | STRUCT<category STRING, mobile_brand_name STRING, mobile_model_name STRING, mobile_marketing_name STRING, mobile_os_hardware_model INT64, operating_system STRING, operating_system_version STRING, vendor_id INT64, advertising_id INT64, language STRING, is_limited_ad_tracking STRING, time_zone_offset_seconds INT64, web_info STRUCT<browser STRING, browser_version STRING>> |
| 103 | app_info.version | STRING |
| 104 | app_info.install_store | STRING |
| 105 | app_info.install_source | STRING |
| 106 | app_info.id | STRING |
| 107 | app_info.firebase_app_id | STRING |
| 108 | app_info | STRUCT<id STRING, version STRING, install_store STRING, firebase_app_id STRING, install_source STRING> |
```