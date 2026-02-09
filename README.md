# GA4 Ecommerce Funnel Analysis

A comprehensive SQL-based funnel analysis using BigQuery on the GA4 sample ecommerce dataset, revealing dual-path revenue models and sequential conversion patterns.

## 📦 About the Data

**Dataset:** [GA4 Obfuscated Sample Ecommerce](https://console.cloud.google.com/marketplace/product/obfuscated-ga360-data/obfuscated-ga4-ecommerce-data-set) (BigQuery Public Dataset)

- **Source Website:** [Google Merchandise Store](https://www.googlemerchandisestore.com/) - Google-branded apparel and merchandise
- **BigQuery Table:** `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
- **Analysis Period:** November 1-30, 2020 (30 days)
- **Data Format:** GA4 event-level export (each row = one event)
- **Access:** Free to query with a Google Cloud account - no download required

> **Note:** This is obfuscated sample data. Some fields (like traffic source) are partially redacted for privacy.


## 🎯 Key Findings

- **Primary Opportunity:** Non-cart checkout abandonment (2,538 sessions, 68% drop rate) represents ~$20K monthly revenue opportunity
- **Dual Revenue Paths:** Non-cart path drives 71% of revenue; cart-assisted path has 12% higher AOV
- **User Behavior:** Returning users convert 2x better at checkout (57% vs 29%) than new users
- **Device Performance:** Minimal differences between mobile (8.36%) and desktop (8.52%) conversion

## 📊 Project Overview

This analysis examines 108,401 sessions from November 2020 to understand:

1. **Sequential funnel leakage** - Where users drop off in timestamp-validated order
2. **Segment-level performance** - Device, geography, user type, and traffic source patterns  
3. **Path economics** - Revenue impact of cart-assisted vs non-cart purchase journeys

**Dataset:** `bigquery-public-data.ga4_obfuscated_sample_ecommerce`  
**Time Period:** November 1-30, 2020  
**Tools:** BigQuery SQL, Python (pandas, matplotlib, seaborn)

## 📁 Repository Structure
```
📁 ga4-ecommerce-funnel-analysis/
├── README.md                                    # This file
├── GA4 Funnel Analysis Executive Summary.md     # Executive summary with findings
├── Data Introduction.md                         # Dataset documentation
├── Step 1 - EDA.ipynb                          # Exploratory data analysis
├── Step 2 - Analysis.ipynb                     # Sequential funnel & segmentation
├── Step 3 - Visualization.ipynb                # Chart generation code
├── 📁 exported_data_visualization/              # CSV exports from BigQuery
└── 📁 exported_visualization/                   # Generated charts (PNG)
```

## 🚀 Quick Start

### View the Analysis

1. **Executive Summary:** [`GA4 Funnel Analysis Executive Summary.md`](GA4%20Funnel%20Analysis%20Executive%20Summary.md)
2. **Data Documentation:** [`Data Introduction.md`](Data%20Introduction.md)

### Run the Analysis

**Prerequisites:**
- Google Cloud account with BigQuery access
- Python 3.8+ with pandas, matplotlib, seaborn


## 📈 Key Visualizations

### 1. Checkout Conversion by Path
![Chart 1](exported_visualization/chart1_checkout_conversion.png)

### 2. Revenue Distribution
![Chart 2](exported_visualization/chart2_revenue_share.png)

### 3. Average Order Value
![Chart 3](exported_visualization/chart3_aov.png)

### 4. Funnel Performance
![Chart 4](exported_visualization/chart4_reach_vs_conversion.png)

### 5. User Segmentation
![Chart 5](exported_visualization/chart5_new_vs_returning.png)

## 🔍 Methodology

**Session Reconstruction:**
- Used `user_pseudo_id + ga_session_id` as session key
- Timestamp-validated sequential transitions (not just reach)

**Revenue Deduplication:**
- Transaction-level dedup by `(session_key, transaction_id)`
- Handles duplicate purchase events

**Path Classification:**
- Mutually exclusive path families
- Cart-assisted vs non-cart distinction based on timestamp order

## 💡 Business Recommendations

**Priority 1: Fix Non-Cart Checkout Abandonment**
- Target: 2,538 abandoned sessions (68% drop rate)
- Impact: Improving to 40% conversion = +$20K monthly revenue
- Hypothesis: Authentication friction, unexpected costs, form complexity

**Priority 2: Dual-Path Reporting**
- Separate KPIs for cart-assisted vs non-cart paths
- Avoid forcing single linear funnel narrative

**Priority 3: New User Experience**
- Focus checkout optimization on first-time buyers (29% conversion vs 57% returning)

## 🛠️ Technical Stack

- **Query Language:** SQL (BigQuery Standard SQL)
- **Data Processing:** Python (pandas, numpy)
- **Visualization:** matplotlib, seaborn
- **Dataset:** GA4 ecommerce event export (Nov 2020)

