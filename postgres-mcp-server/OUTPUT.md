# FutureProofDS

This workspace uses a **PostgreSQL MCP server** so assistants can call MCP tools instead of only suggesting SQL. Below is a concrete log of tool **inputs** and **results** from example sessions.

Server name in Cursor appear as `postgres`. The tools documented here match the server in [`postgres-mcp-server/`](postgres-mcp-server/).

---

## 1. `list_tables`

**MCP function:** `list_tables`  
**Input (arguments):**

```json
{}
```

**Result (human-readable):**

| table          |
|----------------|
| payments       |
| sessions       |
| subscriptions  |
| users          |

**Tool output (JSON array):**

```json
[
  {
    "table": "payments"
  },
  {
    "table": "sessions"
  },
  {
    "table": "subscriptions"
  },
  {
    "table": "users"
  }
]
```

---

## 2. `execute_sql` — tables in `public`

**MCP function:** `execute_sql`  
**Input:**

```json
{
  "sql": "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' AND table_type = 'BASE TABLE' ORDER BY table_name;"
}
```

**Result (human-readable):**

| table_name     |
|----------------|
| payments       |
| sessions       |
| subscriptions  |
| users          |

**Tool output (JSON array):**

```json
[
  {
    "table_name": "payments"
  },
  {
    "table_name": "sessions"
  },
  {
    "table_name": "subscriptions"
  },
  {
    "table_name": "users"
  }
]
```

---

## 3. `execute_sql` — revenue last 30 days (analytics)

**MCP function:** `execute_sql`  
**Input:**

```json
{
  "sql": "SELECT\n    s.plan,\n    u.country,\n    u.device_type,\n    SUM(p.amount_usd) AS total_revenue,\n    COUNT(DISTINCT p.subscription_id) AS paying_subscriptions,\n    AVG(p.amount_usd) AS average_payment_amount\nFROM payments p\nLEFT JOIN subscriptions s\n    ON p.subscription_id = s.subscription_id\nLEFT JOIN users u\n    ON s.user_id = u.user_id\nWHERE p.payment_date >= CURRENT_DATE - INTERVAL '30 days'\nGROUP BY\n    s.plan,\n    u.country,\n    u.device_type\nORDER BY\n    s.plan,\n    u.country,\n    u.device_type;"
}
```

**Result:** **No rows** — with `CURRENT_DATE` after the last payment in this database (latest payments around **2025-09-17**), the trailing 30-day window can be empty.

**Tool output (JSON array):**

```json
[]
```

---

## 4. `execute_sql` — monthly revenue by plan and country

**MCP function:** `execute_sql`  
**Input:**

```json
{
  "sql": "SELECT\n    date_trunc('month', p.payment_date)::date AS revenue_month,\n    s.plan,\n    u.country,\n    SUM(p.amount_usd) AS total_revenue\nFROM payments p\nLEFT JOIN subscriptions s\n    ON p.subscription_id = s.subscription_id\nLEFT JOIN users u\n    ON s.user_id = u.user_id\nGROUP BY\n    date_trunc('month', p.payment_date)::date,\n    s.plan,\n    u.country\nORDER BY\n    revenue_month,\n    s.plan,\n    u.country;"
}
```

**Result summary:**

| Metric | Value |
|--------|--------|
| Row count | **354** |
| `revenue_month` range | **2022-01-01** through **2025-09-01** (month buckets) |
| `plan` values | `annual`, `monthly` |
| `country` values | `EU`, `India`, `Rest`, `US` |
| Sum of `total_revenue` over all rows | **1,030,560.00** USD |

The MCP tool returns **one JSON object per row**. Sample objects:

```json
{"revenue_month": "2022-01-01", "plan": "annual", "country": "Rest", "total_revenue": "200.00"}
{"revenue_month": "2022-01-01", "plan": "monthly", "country": "US", "total_revenue": "20.00"}
{"revenue_month": "2025-09-01", "plan": "monthly", "country": "US", "total_revenue": "9640.00"}
```

**Sample rows** (same query, tabular — first and last month only; full result is 354 rows):

| revenue_month | plan    | country | total_revenue |
|---------------|---------|---------|---------------|
| 2022-01-01    | annual  | Rest    | 200.00        |
| 2022-01-01    | annual  | US      | 100.00        |
| 2022-01-01    | monthly | EU      | 30.00         |
| 2022-01-01    | monthly | Rest    | 40.00         |
| 2022-01-01    | monthly | US      | 20.00         |
| 2025-09-01    | annual  | EU      | 1800.00       |
| 2025-09-01    | annual  | India   | 2100.00       |
| 2025-09-01    | annual  | Rest    | 1100.00       |
| 2025-09-01    | annual  | US      | 3500.00       |
| 2025-09-01    | monthly | EU      | 4870.00       |
| 2025-09-01    | monthly | India   | 4930.00       |
| 2025-09-01    | monthly | Rest    | 4930.00       |
| 2025-09-01    | monthly | US      | 9640.00       |

---

## MCP server setup

See [`postgres-mcp-server/README.md`](README.md) for installation, `.env`, and Cursor configuration.

---

*Future Proof Data Science — Teaching data scientists to optimize workflows with AI*
