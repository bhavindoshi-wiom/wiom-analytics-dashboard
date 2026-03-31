# Wiom Analytics Dashboard — Reproduction Guide

**Dashboard file:** `wiom_dashboard.html`  
**Live URL:** https://bhavindoshi-wiom.github.io/wiom-analytics-dashboard/wiom_dashboard.html  
**Data source:** Snowflake (DB ID 113) via Metabase  
**Timeframe:** Jan 26, 2026 → present  
**Generated:** Mar 31, 2026

---

## Table of Contents

1. [Dashboard Overview](#1-dashboard-overview)
2. [Infrastructure](#2-infrastructure)
3. [Key Business Definitions](#3-key-business-definitions)
4. [View 1 — Plan Expiry Tracker](#4-view-1--plan-expiry-tracker)
5. [View 2+3 — Pickup Tickets](#5-view-23--pickup-tickets)
6. [Bugs Found & Fixed](#6-bugs-found--fixed)
7. [Full Reproduction SQL](#7-full-reproduction-sql)
8. [Dashboard Defaults & Filters](#8-dashboard-defaults--filters)
9. [Refreshing the Data](#9-refreshing-the-data)

---

## 1. Dashboard Overview

Two-tab static HTML dashboard, fully self-contained (no server needed).

| Tab | Description |
|-----|-------------|
| **View 1** | Plan Expiry Tracker — what happens in R0–15 after a plan expires |
| **View 2+3** | Pickup Ticket Dashboard — self-created & system-created ROUTER_PICKUP tickets |

**Default view on open:** Monthly · PayG · All ticket types

---

## 2. Infrastructure

| Layer | Detail |
|-------|--------|
| Database | Snowflake (`database_id = 113`) |
| Query tool | Metabase |
| Primary table | `T_ROUTER_USER_MAPPING` (plan records) |
| Ticket table | `PROD_DB.DYNAMODB_READ.TASKS` (type = `ROUTER_PICKUP`) |
| Payment events | `CUSTOMER_LOGS` (event = `renewal_fee_captured`) |
| Security deposit | `PROD_DB.CUSTOMER_DB_CUSTOMER_PROFILE_SERVICE_PUBLIC.SECURITY_DEPOSIT_ORDERS` |
| Existing reference cards | `9882` (Renewal Lag), `10374` (ROUTER_PICKUP_TICKETS_PAYG), `10387` (Self PUT Funnel), `10547` (PUT Query R.R/C.R), `10705` (Pickup_Tickets_For_Calling) |

**IST offset:** All `otp_issued_time` and `otp_expiry_time` fields are stored in UTC. Apply `DATEADD(minute, 330, ...)` to convert to IST throughout.

---

## 3. Key Business Definitions

### PayG vs NonPayG
```sql
-- TRUE first install date per device (earliest plan start, ever)
first_install AS (
    SELECT
        router_nas_id AS nas_id,
        CAST(MIN(DATEADD(minute, 330, otp_issued_time)) AS DATE) AS install_date
    FROM T_ROUTER_USER_MAPPING
    WHERE otp='DONE' AND store_group_id=0 AND device_limit>1 AND mobile>'5999999999'
    GROUP BY 1
)

-- Classification
CASE WHEN install_date >= '2026-01-26' THEN 'PayG' ELSE 'NonPayG' END AS user_type
```

> ⚠️ **Critical:** Use the device's first-ever install date — NOT the start date of the current plan being evaluated. Using the per-plan start date causes NonPayG renewal plans (bought after Jan 26) to be misclassified as PayG. This was a bug caught and fixed during this session (see §6).

### Plan Expiry
```sql
DATEADD(minute, 330, otp_expiry_time) AS plan_end_time_ist
```
Filter for valid paid plans:
```sql
WHERE otp = 'DONE'
  AND store_group_id = 0
  AND device_limit > 1       -- paid plan indicator
  AND mobile > '5999999999'  -- exclude test/internal numbers
```

### R15 Matured
Only include plan expiries whose 15-day observation window has fully elapsed:
```sql
DATEADD(day, 15, expiry_date) <= CAST(DATEADD(minute, 330, CURRENT_TIMESTAMP()) AS DATE)
```

### Self-Created vs System-Created Pickup Tickets
Derived from card `10705` (`Pickup_Tickets_For_Calling`):
```sql
-- Last plan expiry before ticket creation (for the device)
last_plan_end_ist = MAX(DATEADD(minute, 330, otp_expiry_time))
                    WHERE otp_expiry_time <= ticket_created_ist

-- Self-created: ticket raised within 14 days of plan expiry
mins_after_expiry = DATEDIFF('minute', last_plan_end_ist, ticket_created_ist)

CASE WHEN mins_after_expiry < 20160 THEN 'Customer'   -- < 14 days
     ELSE                                 'System'     -- >= 14 days (auto-generated at R15)
END AS generated_by
```
> Note: The full definition in card 10705 also flags tickets triggered by a ClevertTap push notification (`router_pickup_ticket_created` workflow) as 'System'. That join was excluded here due to query timeouts on the ClevertTap events table.

---

## 4. View 1 — Plan Expiry Tracker

### Purpose
For each plan expiry (where R15 has matured), classify what action the customer took within the R0–15 window.

### Metrics
| Column | Definition |
|--------|-----------|
| `# Plan expiries` | Distinct plan-expiry events where R15 has matured, from Jan 26, 2026 |
| `(a) Recharge R0–15` | Next plan started within 15 days of expiry (`days_to_recharge <= 15`) |
| `(b) Self PUT R0–15` | A `ROUTER_PICKUP` task was created within 0–15 days of **this specific** expiry |
| `(c) No action R0–15` | Neither (a) nor (b) |

**Guarantee:** `a + b + c = total plan expiries` (mutually exclusive, priority: Recharge > Self PUT > No Action)

### Recharge Definition
```sql
-- LEAD gives next plan's start time for the same device
LEAD(DATEADD(minute, 330, otp_issued_time))
    OVER (PARTITION BY router_nas_id ORDER BY otp_issued_time) AS next_plan_start_time

days_to_recharge = DATEDIFF('day', plan_end_time, next_plan_start_time)

-- Recharged if next plan starts within 15 days (includes advance renewals where days < 0)
-- Consistent with existing card 9882 logic (uses R <= 15, not BETWEEN 0 AND 15)
CASE WHEN next_plan_start_time IS NOT NULL AND days_to_recharge <= 15 THEN 'Recharge'
```

### Self PUT Definition (per-expiry window)
```sql
-- All PUT tickets (not pre-aggregated to MIN)
put_all AS (
    SELECT DISTINCT
        PARSE_JSON(extra_data):nas_id::NUMBER AS nas_id,
        DATEADD(minute, 330, created)         AS put_created_ist
    FROM PROD_DB.DYNAMODB_READ.TASKS
    WHERE type = 'ROUTER_PICKUP'
      AND DATEADD(minute, 330, created) >= '2026-01-26'
),

-- Per expiry: check if ANY PUT falls in the R0-15 window for THIS expiry
MAX(CASE
    WHEN pa.put_created_ist >= e.plan_end_time
     AND pa.put_created_ist <= DATEADD(day, 15, e.plan_end_time)
    THEN 1 ELSE 0
END) AS has_put_in_window
```

### Full View 1 SQL
```sql
WITH meta AS (
    SELECT CAST(DATEADD(minute, 330, CURRENT_TIMESTAMP()) AS DATE) AS ist_today
),
first_install AS (
    SELECT
        router_nas_id AS nas_id,
        CAST(MIN(DATEADD(minute, 330, otp_issued_time)) AS DATE) AS install_date
    FROM T_ROUTER_USER_MAPPING
    WHERE otp='DONE' AND store_group_id=0 AND device_limit>1 AND mobile>'5999999999'
    GROUP BY 1
),
plan_base AS (
    SELECT
        t.router_nas_id                                            AS nas_id,
        DATEADD(minute, 330, t.otp_expiry_time)                   AS plan_end_time,
        CAST(DATEADD(minute, 330, t.otp_expiry_time) AS DATE)     AS expiry_date,
        LEAD(DATEADD(minute, 330, t.otp_issued_time)) OVER (
            PARTITION BY t.router_nas_id ORDER BY t.otp_issued_time
        )                                                          AS next_plan_start_time
    FROM T_ROUTER_USER_MAPPING t
    WHERE t.otp='DONE' AND t.store_group_id=0 AND t.device_limit>1
      AND t.mobile>'5999999999'
      AND DATEADD(minute, 330, t.otp_expiry_time) >= '2026-01-26'
),
expiries AS (
    SELECT
        pb.nas_id, pb.expiry_date, pb.plan_end_time, pb.next_plan_start_time,
        fi.install_date,
        CASE WHEN fi.install_date >= '2026-01-26' THEN 'PayG' ELSE 'NonPayG' END AS user_type,
        DATEDIFF('day', pb.plan_end_time, pb.next_plan_start_time) AS days_to_recharge
    FROM plan_base pb
    JOIN first_install fi ON pb.nas_id = fi.nas_id
    CROSS JOIN meta m
    WHERE pb.expiry_date >= '2026-01-26'
      AND DATEADD(day, 15, pb.expiry_date) <= m.ist_today
),
put_all AS (
    SELECT DISTINCT
        PARSE_JSON(extra_data):nas_id::NUMBER AS nas_id,
        DATEADD(minute, 330, created)         AS put_created_ist
    FROM PROD_DB.DYNAMODB_READ.TASKS
    WHERE type='ROUTER_PICKUP'
      AND DATEADD(minute, 330, created) >= '2026-01-26'
),
expiry_with_put AS (
    SELECT
        e.*,
        MAX(CASE
            WHEN pa.put_created_ist >= e.plan_end_time
             AND pa.put_created_ist <= DATEADD(day, 15, e.plan_end_time)
            THEN 1 ELSE 0
        END) AS has_put_in_window
    FROM expiries e
    LEFT JOIN put_all pa ON e.nas_id = pa.nas_id
    GROUP BY e.nas_id, e.expiry_date, e.plan_end_time, e.next_plan_start_time,
             e.install_date, e.user_type, e.days_to_recharge
),
categorised AS (
    SELECT
        expiry_date, user_type,
        CASE
            WHEN next_plan_start_time IS NOT NULL AND days_to_recharge <= 15 THEN 'Recharge'
            WHEN has_put_in_window = 1                                        THEN 'Self_PUT'
            ELSE                                                               'No_Action'
        END AS bucket
    FROM expiry_with_put
)

SELECT
    'Weekly' AS view_type,
    DATE_TRUNC('week', expiry_date)  AS period_start,
    user_type,
    COUNT(*)                         AS plan_expiries,
    SUM(CASE WHEN bucket='Recharge'  THEN 1 ELSE 0 END) AS recharges_r0_15,
    SUM(CASE WHEN bucket='Self_PUT'  THEN 1 ELSE 0 END) AS self_put_r0_15,
    SUM(CASE WHEN bucket='No_Action' THEN 1 ELSE 0 END) AS no_action_r0_15
FROM categorised GROUP BY 1,2,3

UNION ALL

SELECT
    'Monthly', DATE_TRUNC('month', expiry_date), user_type,
    COUNT(*),
    SUM(CASE WHEN bucket='Recharge'  THEN 1 ELSE 0 END),
    SUM(CASE WHEN bucket='Self_PUT'  THEN 1 ELSE 0 END),
    SUM(CASE WHEN bucket='No_Action' THEN 1 ELSE 0 END)
FROM categorised GROUP BY 1,2,3

ORDER BY view_type, period_start, user_type;
```

---

## 5. View 2+3 — Pickup Tickets

### Purpose
Status breakdown and recovery funnel for `ROUTER_PICKUP` tasks, split by how the ticket was created (self vs system).

### Ticket Status Mapping
| Status code | Label | Meaning |
|-------------|-------|---------|
| 0, 1 | Open | Active, unresolved |
| 3 | Send to Wiom | Escalated |
| 2 | Resolved | Closed — see sub-buckets below |

### Resolved Sub-buckets
```sql
-- Customer recharge
CASE WHEN status=2 AND (
    recovery_type='CUSTOMER_RECOVERED' OR
    (recovery_type IS NULL AND payment_mode IS NOT NULL)
) THEN 'customer_recharged'

-- Router recovered (physical pickup)
CASE WHEN status=2 AND recovery_type='ROUTER_RECOVERED' THEN 'router_recovered'

-- Unidentified (bug / unknown)
CASE WHEN status=2 AND recovery_type IS NULL AND payment_mode IS NULL THEN 'unidentified'
```

`payment_mode` is sourced from `CUSTOMER_LOGS` (event = `renewal_fee_captured`), joined on:
```sql
re.recharged_time BETWEEN t.ticket_created_ist AND t.ticket_due_ist
```

### PayG Recovery Funnel (PayG tickets only)
```sql
-- Stage 1: Router physically recovered
status=2 AND recovery_type='ROUTER_RECOVERED'

-- Stage 2: Customer added UPI for refund
above AND upi_id IS NOT NULL   -- from SECURITY_DEPOSIT_ORDERS

-- Stage 3: Refund successfully processed
above AND refund_status='SUCCESSFUL'
```

### Full View 2+3 SQL
```sql
WITH first_install AS (
    SELECT router_nas_id AS nas_id,
           CAST(MIN(DATEADD(minute, 330, otp_issued_time)) AS DATE) AS install_date
    FROM T_ROUTER_USER_MAPPING
    WHERE otp='DONE' AND store_group_id=0 AND device_limit>1 AND mobile>'5999999999'
    GROUP BY 1
),
raw_tickets AS (
    SELECT
        t.reporter_id                                             AS ticket_id,
        PARSE_JSON(t.extra_data):nas_id::NUMBER                  AS nas_id,
        PARSE_JSON(t.extra_data):customer_account_id::STRING     AS customer_account_id,
        PARSE_JSON(t.extra_data):resolutionType::STRING          AS recovery_type,
        TRY_TO_TIMESTAMP(PARSE_JSON(t.extra_data):last_bill_due::STRING) AS last_bill_due_raw,
        t.id                                                     AS task_id,
        DATEADD(minute, 330, t.created)                          AS ticket_created_ist,
        DATEADD(minute, 330, t.due_date)                         AS ticket_due_ist,
        t.status,
        t.mobile
    FROM PROD_DB.DYNAMODB_READ.TASKS t
    WHERE t.type='ROUTER_PICKUP'
      AND DATEADD(minute, 330, t.created) >= '2026-01-26'
    QUALIFY ROW_NUMBER() OVER (PARTITION BY t.id ORDER BY t.created DESC) = 1
),
last_plan_before_ticket AS (
    SELECT t.ticket_id,
           MAX(DATEADD(minute, 330, p.otp_expiry_time)) AS last_plan_end_ist
    FROM raw_tickets t
    JOIN T_ROUTER_USER_MAPPING p
        ON t.nas_id = p.router_nas_id
       AND DATEADD(minute, 330, p.otp_expiry_time) <= t.ticket_created_ist
       AND p.otp='DONE' AND p.store_group_id=0 AND p.device_limit>1
    GROUP BY 1
),
recharge_events AS (
    SELECT mobile,
           DATEADD(minute, 330, added_time)    AS recharged_time,
           PARSE_JSON(data):paymentMode::STRING AS payment_mode
    FROM CUSTOMER_LOGS
    WHERE event_name='renewal_fee_captured'
      AND DATEADD(minute, 330, added_time) >= '2026-01-26'
),
sdo AS (
    SELECT customer_account_id, upi_id,
           status AS refund_status,
           ROW_NUMBER() OVER (PARTITION BY customer_account_id ORDER BY created_at DESC) AS rn
    FROM PROD_DB.CUSTOMER_DB_CUSTOMER_PROFILE_SERVICE_PUBLIC.SECURITY_DEPOSIT_ORDERS
),
enriched AS (
    SELECT
        t.ticket_id, t.ticket_created_ist, t.ticket_due_ist,
        t.status, t.recovery_type, t.mobile,
        fi.install_date,
        CASE WHEN fi.install_date >= '2026-01-26' THEN 'PayG' ELSE 'NonPayG' END AS user_type,
        COALESCE(t.last_bill_due_raw, lp.last_plan_end_ist) AS expiry_ref,
        DATEDIFF('minute', expiry_ref, t.ticket_created_ist) AS mins_after_expiry,
        CASE WHEN mins_after_expiry < 20160 THEN 'Customer' ELSE 'System' END AS generated_by,
        re.payment_mode,
        s.upi_id,
        s.refund_status
    FROM raw_tickets t
    LEFT JOIN last_plan_before_ticket lp ON t.ticket_id = lp.ticket_id
    LEFT JOIN first_install fi            ON t.nas_id = fi.nas_id
    LEFT JOIN recharge_events re
        ON t.mobile = re.mobile
       AND re.recharged_time BETWEEN t.ticket_created_ist AND t.ticket_due_ist
    LEFT JOIN sdo s ON t.customer_account_id = s.customer_account_id AND s.rn = 1
)

-- Replace <TICKET_TYPE_FILTER> with:
--   generated_by = 'Customer'  → self-created tickets (View 2)
--   generated_by = 'System'    → system-created tickets (View 3)
--   1=1                        → all tickets (combined)

SELECT
    'Weekly' AS view_type,
    DATE_TRUNC('week', CAST(ticket_created_ist AS DATE)) AS period_start,
    user_type,
    COUNT(DISTINCT ticket_id) AS total_tickets,
    COUNT(DISTINCT CASE WHEN status IN (0,1) THEN ticket_id END) AS status_open,
    COUNT(DISTINCT CASE WHEN status=3       THEN ticket_id END) AS status_send_to_wiom,
    COUNT(DISTINCT CASE WHEN status=2       THEN ticket_id END) AS status_resolved,
    COUNT(DISTINCT CASE WHEN status=2 AND (
        recovery_type='CUSTOMER_RECOVERED' OR
        (recovery_type IS NULL AND payment_mode IS NOT NULL)
    ) THEN ticket_id END)                                       AS resolved_cust_recharge,
    COUNT(DISTINCT CASE WHEN status=2 AND recovery_type='ROUTER_RECOVERED'
                        THEN ticket_id END)                     AS resolved_rtr_recovered,
    COUNT(DISTINCT CASE WHEN status=2
        AND recovery_type IS NULL AND payment_mode IS NULL      THEN ticket_id END) AS resolved_unidentified,
    -- PayG recovery funnel (only meaningful for PayG rows)
    COUNT(DISTINCT CASE WHEN user_type='PayG' AND status=2
        AND recovery_type='ROUTER_RECOVERED'                    THEN ticket_id END) AS payg_rr,
    COUNT(DISTINCT CASE WHEN user_type='PayG' AND status=2
        AND recovery_type='ROUTER_RECOVERED' AND upi_id IS NOT NULL
                                                                THEN ticket_id END) AS payg_rr_upi,
    COUNT(DISTINCT CASE WHEN user_type='PayG' AND status=2
        AND recovery_type='ROUTER_RECOVERED' AND upi_id IS NOT NULL
        AND refund_status='SUCCESSFUL'                          THEN ticket_id END) AS payg_rr_upi_refund
FROM enriched
WHERE <TICKET_TYPE_FILTER>
GROUP BY 1,2,3

UNION ALL

SELECT 'Monthly', DATE_TRUNC('month', CAST(ticket_created_ist AS DATE)), user_type,
    COUNT(DISTINCT ticket_id),
    COUNT(DISTINCT CASE WHEN status IN (0,1) THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN status=3       THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN status=2       THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN status=2 AND (recovery_type='CUSTOMER_RECOVERED' OR (recovery_type IS NULL AND payment_mode IS NOT NULL)) THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN status=2 AND recovery_type='ROUTER_RECOVERED' THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN status=2 AND recovery_type IS NULL AND payment_mode IS NULL THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN user_type='PayG' AND status=2 AND recovery_type='ROUTER_RECOVERED' THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN user_type='PayG' AND status=2 AND recovery_type='ROUTER_RECOVERED' AND upi_id IS NOT NULL THEN ticket_id END),
    COUNT(DISTINCT CASE WHEN user_type='PayG' AND status=2 AND recovery_type='ROUTER_RECOVERED' AND upi_id IS NOT NULL AND refund_status='SUCCESSFUL' THEN ticket_id END)
FROM enriched
WHERE <TICKET_TYPE_FILTER>
GROUP BY 1,2,3

ORDER BY view_type, period_start, user_type;
```

---

## 6. Bugs Found & Fixed

### Bug 1 — PayG classification using per-plan install date (View 1)
**Symptom:** PayG expiry volumes spiked from week of Feb 23 (~173 → 33,927) while NonPayG dropped to near zero in the same week. Numbers looked inverted.

**Root cause:** The `install_date` was computed from each plan row's own `otp_issued_time`, not the device's first-ever plan start. A NonPayG device that installed in December 2025 and renewed in February 2026 had its renewal plan classified as PayG (because that plan's start date was ≥ Jan 26).

**Fix:** Pre-aggregate to get the true first install date per device using `MIN(otp_issued_time)` across all plans.

```sql
-- WRONG (per-plan start date)
CAST(DATEADD(minute, 330, otp_issued_time) AS DATE) AS install_date

-- CORRECT (first-ever install date per device)
SELECT router_nas_id AS nas_id,
       CAST(MIN(DATEADD(minute, 330, otp_issued_time)) AS DATE) AS install_date
FROM T_ROUTER_USER_MAPPING
WHERE otp='DONE' AND store_group_id=0 AND device_limit>1 AND mobile>'5999999999'
GROUP BY 1
```

---

### Bug 2 — PUT window scoped globally (not per expiry) (View 1)
**Symptom:** Self PUT count was ~4,800/week for NonPayG but dropped to ~1,300/week after fix. No Action was correspondingly understated.

**Root cause:** The query used `MIN(created)` globally per NAS device — whichever ROUTER_PICKUP task was created first for that device since Jan 26, regardless of which plan expiry it corresponded to. A device with a PUT from an earlier cycle would have that old ticket masking whether a newer expiry had a qualifying PUT.

**Example of incorrect behavior:**
- Device expires Jan 30 (expiry A), expires Mar 5 (expiry B)
- PUT created Feb 3 (valid for expiry A)
- PUT created Mar 7 (valid for expiry B)
- With `MIN()`: `first_put = Feb 3`
- For expiry B: `DATEDIFF(Mar 5, Feb 3) = -30` → correctly no PUT (happens to work)
- BUT: if MIN was Mar 12 (only PUT, for a late expiry), an earlier expiry would miss it

**Fix:** Instead of pre-aggregating to `MIN`, collect all PUT dates per device and do a window-scoped join per expiry:

```sql
-- Check if ANY PUT falls within R0-15 of THIS specific expiry
MAX(CASE
    WHEN pa.put_created_ist >= e.plan_end_time
     AND pa.put_created_ist <= DATEADD(day, 15, e.plan_end_time)
    THEN 1 ELSE 0
END) AS has_put_in_window
```

---

### Bug 3 — Recharge boundary excluded advance renewals (View 1)
**Symptom:** Minor undercount of recharges.

**Root cause:** Condition `days_to_recharge BETWEEN 0 AND 15` excluded negative values (i.e., customers who renewed before their current plan expired). Existing card 9882 uses `R <= 15` which includes advance renewals.

**Fix:** Changed to `days_to_recharge <= 15` (while still requiring `next_plan_start_time IS NOT NULL`).

---

### Bug 4 — `SHARD` column not found (View 1, initial attempt)
**Symptom:** SQL compilation error: `invalid identifier 'SHARD'`

**Root cause:** Initial query used `idmaker(shard, 0, router_nas_id)` copied from existing cards (e.g., 9882). The `shard` column exists in some contexts but was not accessible in a plain `SELECT` from `T_ROUTER_USER_MAPPING`.

**Fix:** Use `router_nas_id` directly as the NAS identifier throughout. For joining with the TASKS table (which stores NAS ID as `PARSE_JSON(extra_data):nas_id::NUMBER`), join on `e.nas_id = pt.nas_id` directly.

---

## 7. Full Reproduction SQL

Both queries are provided in full in §4 and §5. To reproduce in Metabase:

1. Open Metabase → New Question → Native Query
2. Select database: **Snowflake** (ID 113)
3. Paste the relevant SQL
4. For View 2+3, replace `<TICKET_TYPE_FILTER>` with:
   - `generated_by = 'Customer'` for self-created
   - `generated_by = 'System'` for system-created
   - `1=1` for all tickets

---

## 8. Dashboard Defaults & Filters

| Filter | Default | Options |
|--------|---------|---------|
| User type | PayG | All users · PayG · NonPayG |
| Timeline | Monthly | Weekly · Monthly |
| Ticket type (View 2+3 only) | All tickets | All tickets · Self-created · System-created |

**PayG Recovery Funnel** is only shown when user type = PayG or All users.

**Total row** is appended at the bottom of every table, summarising all visible periods.

---

## 9. Refreshing the Data

The HTML file has all data hardcoded as of Mar 31, 2026. To refresh:

1. Run the View 1 SQL (§4) and View 2+3 SQL (§5) in Metabase
2. Copy the result rows into the `V1` and `V23` arrays in the `<script>` block of `wiom_dashboard.html`
3. Each row follows the format:

**View 1:**
```js
{v:"Weekly|Monthly", p:"YYYY-MM-DD|YYYY-MM", u:"PayG|NonPayG",
 e:<expiries>, r:<recharges>, s:<self_put>, n:<no_action>}
```

**View 2+3:**
```js
{k:"Self|System", v:"Weekly|Monthly", p:"YYYY-MM-DD|YYYY-MM", u:"PayG|NonPayG",
 t:<total>, op:<open>, sw:<send_to_wiom>, rs:<resolved>,
 cr:<cust_recharge>, rr:<rtr_recovered>, ui:<unidentified>,
 prr:<payg_rr>, pu:<payg_rr_upi>, prf:<payg_rr_upi_refund>}
```

4. Commit and push:
```bash
cd /path/to/wiom-analytics-dashboard
git add wiom_dashboard.html
git commit -m "Refresh data — <date>"
git push origin main
```

---

## Reference: Existing Metabase Cards Used

| Card ID | Name | Used for |
|---------|------|----------|
| 9882 | Renewal Lag After First Paid Plan Expiry | Recharge logic pattern (`R <= N`), `T_ROUTER_USER_MAPPING` filter pattern |
| 9444 | PayG Users Recharge Behaviour | PayG plan ID set, STP plan definitions |
| 10374 | ROUTER_PICKUP_TICKETS_PAYG | Self-created threshold (`< 20540 min`), resolution logic |
| 10387 | Funnel - churn - put - Self put created | `generated_by` logic, ticket window joins |
| 10547 | PUT_Query(R.R/C.R) | `cx_type` PayG/nPayG classification, security deposit join pattern |
| 10705 | Pickup_Tickets_For_Calling | `generated_by` definition, `last_bill_due` from `extra_data`, NAS ID join pattern |
| 10368 | PUT - Security Deposit Refund Status | Resolved status sub-buckets, `recovery_type` values |
| 10706 | pickup_tickets_live_status | TICKETS table schema reference |
