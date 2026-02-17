# Funnel Analysis Methods — SQL Approaches

## Background

When analyzing user journeys from event-level data, there are 4 main ways to build a funnel in SQL. Each has different accuracy, performance, and complexity tradeoffs. This doc walks through all 4 with examples and when to use each.

**The core problem:** event data is messy. Users don't follow clean linear paths — they browse, backtrack, remove items, come back later. So how you define "did this user progress through the funnel" actually matters a lot.

**Reference Source:** https://www.quadratichq.com/blog/funnel-analysis-from-signup-to-activation-and-beyond

---

## Setup — Sample Data
```
| user  | event        | timestamp |
|-------|-------------|-----------|
| UserA | view_item   | 10:01     |
| UserA | add_to_cart | 10:05     |
| UserA | purchase    | 10:09     |
| UserB | view_item   | 10:02     |
| UserB | purchase    | 10:06     |
| UserC | view_item   | 10:03     |
| UserC | add_to_cart | 10:04     |
```

Expected result: 3 viewed, 2 carted, 1 purchased (only UserA completed all 3 steps in order)

---

## Method 1 — The Bad (Naive Count)

### What it does
Just counts how many times each event fired. No connection between users across steps.
```sql
SELECT
  COUNTIF(event = 'view_item')   AS viewers,
  COUNTIF(event = 'add_to_cart') AS carters,
  COUNTIF(event = 'purchase')    AS buyers
FROM events
```

### Result
```
| viewers | carters | buyers |
|---------|---------|--------|
| 3       | 2       | 2      |
```

### Why this is wrong
UserB never added to cart but still purchased. UserC added to cart but never purchased.
This method doesn't care — it just counts raw events independently.

Conversion = buyers/viewers = 2/3 = 67% — but this number is meaningless because
you're not tracking whether the SAME user went through all steps in order.

Can also give you conversion rates over 100% if more people purchased than viewed
(e.g. users who came in through a direct purchase link).

### Pros / Cons
```
✅ Simple to write
❌ Wrong answers — counts are independent, not connected
❌ Can give conversion rates over 100%
❌ Never use this for actual funnel analysis
```

---

## Method 2 — MIN Timestamp (First-Touch Attribution)

### What it does
For each user, grab the FIRST time each event happened.
Then check if those first occurrences happened in the right order.
```sql
WITH funnel AS (
  SELECT
    user,
    MIN(IF(event = 'view_item',   timestamp, NULL)) AS ts_view,
    MIN(IF(event = 'add_to_cart', timestamp, NULL)) AS ts_cart,
    MIN(IF(event = 'purchase',    timestamp, NULL)) AS ts_purchase
  FROM events
  GROUP BY user
)
SELECT
  COUNT(*)                            AS reached_view,
  COUNTIF(ts_cart >= ts_view)         AS reached_cart,
  COUNTIF(ts_purchase >= ts_cart)     AS reached_purchase
FROM funnel
```

### Intermediate result (one row per user)
```
| user  | ts_view | ts_cart | ts_purchase |
|-------|---------|---------|-------------|
| UserA | 10:01   | 10:05   | 10:09       |
| UserB | 10:02   | null    | 10:06       |
| UserC | 10:03   | 10:04   | null        |
```

### Final result
```
| reached_view | reached_cart | reached_purchase |
|-------------|-------------|-----------------|
| 3           | 2           | 1               |
```

Order check logic:
- UserA: cart(10:05) >= view(10:01) ✅ → purchase(10:09) >= cart(10:05) ✅ → counts
- UserB: ts_cart is null → fails at cart step ❌
- UserC: ts_purchase is null → fails at purchase step ❌

### The attribution question — first-touch vs last-touch

**First-touch (MIN)** = most conservative view
- Takes the earliest occurrence of each event
- If the first cart happened before the first view → doesn't count the user
- Undercounts rather than overcounts
- Good when you want a conservative baseline

**Last-touch (MAX)** = more forgiving
- Takes the latest occurrence of each event
- More likely to find a valid ordering even in messy journeys
- Better when users are expected to repeat steps (e.g. returning to cart multiple times)

Example where they differ:
```
| event        | timestamp |
|-------------|-----------|
| add_to_cart | 10:01     |  ← cart before view
| view_item   | 10:05     |
| add_to_cart | 10:08     |  ← cart after view
| purchase    | 10:12     |
```

MIN timestamp: ts_cart = 10:01, ts_view = 10:05 → cart < view → ❌ doesn't count
MAX timestamp: ts_cart = 10:08, ts_view = 10:05 → cart > view → ✅ counts

### Known limitations

1. **Doesn't track item_id** — user could view item A, cart item B, purchase item C.
   All three are different products but the session still looks like a valid conversion.
   Analysis is session-level behavior, not product-level conversion.

2. **Misses path complexity** — if user abandoned a cart and came back through a
   completely different path, MIN timestamp can't see that. It just grabbed the
   earliest timestamps.

3. **remove_from_cart not handled** — if user adds item, removes it, then purchases
   something else, it still looks cart-assisted.

### Why it still works despite limitations

- **Conservative direction** — undercounts rather than overcounts, so findings
  are if anything stronger than what the numbers show
- **Effect sizes are large** — non-cart at 73% revenue share is robust enough
  that individual misclassifications don't flip the conclusion
- **Session-level question** — the question being asked is "what paths do users
  take to purchase" not "did this user convert on this specific product"
- **Standard starting point** — right tool for exploratory analysis before
  building production-grade systems

### Pros / Cons
```
✅ Simple to write
✅ Correct — tracks same user through steps in order
✅ Fast — one GROUP BY, no joins
✅ Conservative — undercounts not overcounts
✅ Good enough for exploratory and directional analysis
⚠️  Misses path complexity (cart abandonment, item switching)
⚠️  Individual session classification can be wrong
❌ Not precise enough for production KPI dashboards
```

---

## Method 3 — JOIN-Based Sequential Funnel

### What it does
Instead of collapsing everything into one row per user first, keeps events as
separate rows and JOINs them step by step with a timestamp condition.

Core condition on every JOIN:
**next event timestamp must be AFTER previous event timestamp**
```sql
WITH view_events AS (
  SELECT user, timestamp AS view_time
  FROM events
  WHERE event = 'view_item'
),
cart_events AS (
  SELECT user, timestamp AS cart_time
  FROM events
  WHERE event = 'add_to_cart'
),
purchase_events AS (
  SELECT user, timestamp AS purchase_time
  FROM events
  WHERE event = 'purchase'
)
SELECT
  COUNT(DISTINCT v.user) AS reached_view,
  COUNT(DISTINCT c.user) AS reached_cart,
  COUNT(DISTINCT p.user) AS reached_purchase
FROM view_events v
LEFT JOIN cart_events c
  ON v.user = c.user
  AND c.cart_time >= v.view_time
LEFT JOIN purchase_events p
  ON c.user = p.user
  AND p.purchase_time >= c.cart_time
```

### Why COUNT DISTINCT is critical here

With multiple events per user, JOIN generates ALL possible valid combinations:
```
UserA has: view(10:01), view(10:05), cart(10:03), cart(10:07), purchase(10:12)

JOIN produces:
| view  | cart  | purchase |
|-------|-------|----------|
| 10:01 | 10:03 | 10:12    |
| 10:01 | 10:07 | 10:12    |
| 10:05 | 10:07 | 10:12    |
```

One real user, one real purchase — but 3 rows.
COUNT DISTINCT collapses this back to 1. Without it, numbers are completely wrong.

### Why JOIN is more accurate than MIN timestamp

MIN timestamp asks:
"Was the FIRST time you did A before the FIRST time you did B?"

JOIN asks:
"Did you EVER do A and then B after it, at any point in the session?"

Example where JOIN catches what MIN misses:
```
| event        | timestamp |
|-------------|-----------|
| add_to_cart | 10:01     |  ← first cart is before view
| view_item   | 10:05     |
| add_to_cart | 10:08     |  ← second cart is after view
| purchase    | 10:12     |
```

MIN: ts_cart = 10:01, ts_view = 10:05 → ❌ misses this user
JOIN: finds view(10:05) → cart(10:08) → purchase(10:12) → ✅ counts this user

### Performance problem

JOIN compares every view event against every cart event for the same user.
If a user has 10 views and 5 carts → up to 50 comparisons just for one user.
This is O(n²) complexity — as dataset grows, query cost grows exponentially.

Fine at 108K sessions. Painful at 10M+ sessions.

### Pros / Cons
```
✅ More accurate — tracks actual event pairs not just earliest timestamps
✅ Catches real conversions that MIN timestamp misses
✅ More forgiving of messy user journeys
⚠️  Must use COUNT DISTINCT or numbers are wrong
❌ More complex to write
❌ Slow on large datasets — O(n²) complexity
❌ Overkill for exploratory analysis
```

---

## Method 4 — Window Functions (LEAD)

### What it does
Instead of matching events against each other, sorts everything by timestamp
and peeks at what comes NEXT using the LEAD() window function.

Core insight: if events are already sorted by time, you don't need to compare
every pair — just look at the next row.
```sql
SELECT
  user,
  event,
  timestamp,
  LEAD(event) OVER (
    PARTITION BY user
    ORDER BY timestamp
  ) AS next_event,
  LEAD(timestamp) OVER (
    PARTITION BY user
    ORDER BY timestamp
  ) AS next_timestamp
FROM events
```

### Result
```
| user  | event        | timestamp | next_event   | next_timestamp |
|-------|-------------|-----------|-------------|----------------|
| UserA | view_item   | 10:01     | add_to_cart | 10:05          |
| UserA | add_to_cart | 10:05     | purchase    | 10:09          |
| UserA | purchase    | 10:09     | null        | null           |
| UserB | view_item   | 10:02     | purchase    | 10:06          |
| UserB | purchase    | 10:06     | null        | null           |
| UserC | view_item   | 10:03     | add_to_cart | 10:04          |
| UserC | add_to_cart | 10:04     | null        | null           |
```

Each row now knows what happened NEXT without any joining.

### Finding transitions

To find view → cart transitions:
```sql
WHERE event = 'view_item'
AND next_event = 'add_to_cart'
```

No massive join needed. Just filter rows where the next event is what you want.

### Why this is faster than JOIN

JOIN: compares every event against every other event = O(n²)
LEAD: looks at next row in already-sorted list = O(n)

At 10M sessions that difference is enormous.

### The key limitation

LEAD only looks ONE row ahead — so it only catches IMMEDIATE transitions.
```
| event        |
|-------------|
| view_item   |
| page_view   |  ← interrupting event
| add_to_cart |
```

LEAD sees view_item → next is page_view. Not add_to_cart. Misses the transition.

To handle non-consecutive events you need tricks like LEAD with IGNORE NULLS
or stacking multiple window functions. That's why Mitzu calls it "the ugly" —
it's the fastest but requires the most clever implementation.

### Pros / Cons
```
✅ Fastest approach — O(n) complexity
✅ No combinatorial explosion like JOIN
✅ Best for production systems at scale (10M+ sessions)
⚠️  Only catches immediate consecutive transitions by default
⚠️  More complex to implement correctly for non-consecutive events
❌ Requires extra tricks (IGNORE NULLS, stacked windows) for messy journeys
```

---

## Summary — All 4 Methods
```
| Method           | Accuracy    | Speed    | Complexity | Best for              |
|-----------------|-------------|----------|------------|-----------------------|
| Naive count     | Wrong       | Fast     | Simple     | Never                 |
| MIN timestamp   | Conservative| Fast     | Simple     | Exploration           |
| JOIN            | More precise| Slow     | Medium     | Production (small-mid)|
| Window functions| Depends     | Fastest  | Complex    | Production at scale   |
```

## Decision framework

- Exploratory analysis, finding the big story → **MIN timestamp**
- Production dashboard, finance/product teams → **JOIN with COUNT DISTINCT**
- Production at scale (10M+ sessions) → **Window functions**
- Never → **Naive count**

## Key methodological note

MIN timestamp was used in this project because:
1. Goal was exploratory — find the dominant pattern, not precise individual classification
2. Conservative baseline — undercounts rather than overcounts, so findings are
   if anything stronger than what numbers show
3. Effect sizes are large enough (73% non-cart revenue share) that individual
   misclassifications don't change the conclusion
4. Session-level question — asking "what paths do users take" not
   "did this user convert on this specific product"

Next iteration would use JOIN-based sequential matching for more precise
path classification before putting metrics into a production dashboard.