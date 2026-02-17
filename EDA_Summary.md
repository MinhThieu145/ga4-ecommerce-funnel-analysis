# GA4 EDA Summary — Query 1 to 6

## What the EDA was for

Before building any funnel analysis, needed to answer three questions:
- Is the dataset big enough and clean enough to analyze?
- Can sessions be reconstructed from raw event data?
- Which fields are trustworthy enough to build KPIs on?

Ran 6 focused queries to work through these systematically before touching
any in-depth analysis.

---

## Query 1 — Big Picture Health Check

**Question:** Is this dataset worth analyzing at all?

**What it checked:**
- Dataset scale (events, users, sessions)
- Whether session reconstruction is viable
- Whether funnel events exist
- Whether revenue data is present
- Baseline conversion rate

**What we found:**

| Metric | Value |
|--------|-------|
| Total events | 1,472,712 |
| Total users | 79,421 |
| Sessions (reconstructed) | 108,401 |
| Session conversion rate | 1.49% |
| AOV | $114.58 |
| Purchase events | 2,054 |
| Distinct transactions | 1,259 |

**Two immediate flags:**

1. purchase_events (2,054) > transactions (1,259)
   - Same transaction_id fired multiple times
   - Happens when confirmation page refreshes and event fires again
   - Decision: always use COUNT(DISTINCT transaction_id) for financial metrics,
     never raw purchase event count

2. begin_checkout > add_to_cart
   - More sessions reaching checkout than cart
   - Logically backwards — users shouldn't reach checkout without cart
   - Flagged for deeper investigation → eventually became the dual-path finding

**Verdict:** Dataset is healthy. Proceed with analysis.

---

## Query 2 — Funnel Coverage and Drop-off

**Question:** How far down the funnel do users actually get?

**What it checked:**
- What % of sessions and users touched each funnel step
- Where the biggest volume drop happens
- Whether all core funnel events exist in the data

**Important note:** This is reach-based, not sequential.
Means it shows who TOUCHED each step at least once — not who followed
steps in strict order. A session that did purchase → view → cart would
still show up at all three steps.

**What we found:**

| Step | Sessions Reached | % of Sessions |
|------|-----------------|---------------|
| session_start | 106,550 | 98.29% |
| page_view | 98,555 | 90.92% |
| view_item | 25,943 | 23.93% |
| add_to_cart | 2,201 | 2.03% |
| begin_checkout | 4,585 | 4.23% |
| purchase | 1,617 | 1.49% |

**Key insights:**

1. Biggest leakage is view_item → add_to_cart (23.93% down to 2.03%)
   - 91.55% drop sounds alarming but is normal ecommerce browsing behavior
   - Most sessions are window shopping, not purchase intent

2. begin_checkout (4.23%) > add_to_cart (2.03%) confirmed
   - More than twice as many sessions reach checkout vs cart
   - Strongly suggests a non-cart path to checkout exists
   - This single observation motivated the entire dual-path framework in L1/L2/L3

**Verdict:** Funnel coverage is usable. Non-linear behavior confirmed and flagged
for deeper analysis.

---

## Query 3 — Core Field Null Audit

**Question:** Can I trust these fields to build analysis on?

**What it checked:**
- Null rates across all core structural fields
- Two versions of source/medium: user-level (traffic_source) vs
  event-level (event_params)

**What we found:**

| Field | Null Rate | Verdict |
|-------|-----------|---------|
| user_pseudo_id | 0.00% | ✅ Safe |
| event_name | 0.00% | ✅ Safe |
| event_timestamp | 0.00% | ✅ Safe |
| ga_session_id | 0.00% | ✅ Session reconstruction viable |
| device_category | 0.00% | ✅ Safe |
| geo_country | 0.00% | ✅ Safe |
| traffic_source.source/medium | 0.00% | ⚠️ Populated but quality unknown |
| event_params source/medium | 66.42% | ⚠️ Expected — not every event carries attribution |

**Key insights:**

1. All core fields are bulletproof — zero nulls across the board
   - Session reconstruction is safe to proceed
   - Funnel and segmentation analysis is fully viable

2. traffic_source 0% null doesn't mean it's usable
   - Field has a value on every row, but value could still be garbage
   - 0% null only confirms presence, not quality
   - Needed Query 5 to actually check value quality

3. 66% null on event_params source/medium is expected
   - These params only populate on session_start events
   - Every other event type has null for these — that's normal GA4 behavior
   - Not a data problem

**Verdict:** Core fields are fully trustworthy. Traffic source quality needs
a separate dedicated check.

---

## Query 4 — Ecommerce Field Null Audit

**Question:** Can I trust the revenue and transaction data for financial reporting?

**What it checked:**
- Null rates on all ecommerce fields, filtered to purchase events only
- Whether transaction_id, revenue, shipping, tax are populated

**Important:** Filtered to `event_name = 'purchase'` only.
These fields are only meaningful on purchase events — checking across all
1.47M events would give misleading 99% null rates since non-purchase events
don't have ecommerce data by design.

**What we found:**

| Field | Null Rate | Verdict |
|-------|-----------|---------|
| transaction_id | 1.12% | ⚠️ Mostly good |
| purchase_revenue | 2.73% | ⚠️ Small gap |
| purchase_revenue_in_usd | 0.00% | ✅ Fully populated |
| shipping_value | 100.00% | ❌ Completely empty |
| tax_value | 2.73% | ⚠️ Small gap |
| total_item_quantity | 2.73% | ⚠️ Small gap |

**Key insights:**

1. purchase_revenue_in_usd is the reliable revenue field
   - purchase_revenue has 2.73% null, usd version has 0%
   - Decision: always use purchase_revenue_in_usd for all revenue calculations

2. shipping_value is 100% null — drop it completely
   - Google never instrumented shipping in this GA4 setup
   - All revenue figures represent product revenue only, not total order value
   - Flagged as known limitation, not used anywhere in analysis

3. purchase_revenue, tax_value, and total_item_quantity all share the same
   2.73% null rate — not a coincidence
   - Same subset of events failing to populate the entire ecommerce object
   - Likely a tag firing issue where purchase event fired but ecommerce
     data didn't attach
   - Small enough (< 3%) to not materially affect analysis

**Three decisions coming out of this query:**
- Use purchase_revenue_in_usd for all revenue
- Use COUNT(DISTINCT transaction_id) not COUNT(*) for transaction counts
- Exclude shipping_value from all KPI logic

---

## Query 5 — Traffic Source Quality Audit

**Question:** Of the traffic source data we have — how much is actually real signal?

**What it checked:**
- Classified every user into one of four quality buckets:
  - usable_attributed — real sources (google, facebook, actual campaigns)
  - default_unattributed — direct/(none), real but not actionable
  - placeholder_or_deleted — (not set), (data deleted), <other> — garbage
  - null_or_blank — completely empty
- Measured traffic concentration using HHI and top N source share

**What we found:**

| Metric | Value |
|--------|-------|
| Total users | 79,421 |
| Usable attributed | 35,211 (44.33%) |
| Default unattributed | 18,614 (23.44%) |
| Placeholder/deleted | 25,596 (32.23%) |
| Top 1 source share | 37.49% |
| Top 3 source share | 89.49% |
| Source HHI | 0.2831 |

**Key insights:**

1. More than half the data is not usable for attribution
   - Only 44.33% has clean attributable values
   - 32.23% is obfuscated placeholders specific to this public dataset
   - 23.44% is direct/unknown — real but tells you nothing about channel

2. Traffic is extremely concentrated
   - Top 3 sources = 89.49% of all traffic
   - HHI of 0.2831 confirms moderate-to-high concentration
   - Not enough source diversity to do meaningful channel comparison

3. This query exists to RULE OUT channel analysis, not to do it
   - Without this audit, skipping channel analysis looks like an oversight
   - With it, dropping channel analysis is a documented, evidence-backed decision

**Verdict:** Traffic source is directional context only. Not suitable for
deep attribution analysis. Decision made to exclude it from core story.

---

## Query 6 — Traffic Source Distribution Table

**Question:** What are the actual source/medium values — what does the real
traffic picture look like?

**What it checked:**
- Ranked every source/medium combination by user volume
- Showed what quality bucket each combination falls into
- Calculated cumulative % to show how fast coverage adds up

**What we found (top rows):**

| Rank | Source | Medium | Quality | % Users | Cumulative % |
|------|--------|--------|---------|---------|--------------|
| 1 | google | organic | usable | 32.74% | 32.74% |
| 2 | (direct) | (none) | default | 23.38% | 56.12% |
| 3 | \<other\> | \<other\> | placeholder | ~15% | ~71% |
| 4 | (data deleted) | (data deleted) | placeholder | ~10% | ~81% |
| 5 | google | cpc | usable | ~7% | ~88% |

**Key insights:**

1. google organic is the only clean dominant source at 32.74%
   - Meaningful signal, but alone it can't support a full channel story
   - Everything after it is either direct or obfuscated

2. By row 2 you're already at 56% and half of it is direct/unknown
   - Cumulative % makes this immediately obvious
   - Real signal runs out extremely fast

3. Query 6 completes the picture Query 5 started
   - Q5 = how much of the data is usable (44.33%)
   - Q6 = here's exactly what that data looks like and what the garbage is
   - Together they form the complete justification for dropping
     deep channel analysis

**Verdict:** Confirmed Q5's conclusion. Keep channel analysis coarse,
always include an Unknown/Obfuscated bucket in any source/medium report,
do not make deep attribution claims from this dataset.

---

## EDA Overall Conclusion

| Query | Core Question | Verdict |
|-------|--------------|---------|
| Q1 | Is dataset usable? | ✅ Yes — strong volume, session reconstruction works |
| Q2 | How far do users get? | ✅ All funnel steps present, checkout > cart anomaly flagged |
| Q3 | Are core fields clean? | ✅ Zero nulls across all structural fields |
| Q4 | Is revenue data trustworthy? | ✅ Yes — use usd revenue field and distinct transaction_id |
| Q5 | Is traffic source quality good? | ❌ Only 44% usable — drop deep channel analysis |
| Q6 | What does traffic actually look like? | ❌ Confirms Q5 — too concentrated, too obfuscated |

**What EDA greenlit:**
- Funnel leakage analysis
- Device and geo segmentation
- Session-level behavior and conversion patterns
- Revenue and AOV reporting

**What EDA ruled out:**
- Deep channel attribution analysis
- Source/medium conversion comparison
- Any claim connecting traffic source to conversion outcome