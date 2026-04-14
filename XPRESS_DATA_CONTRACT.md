# Xpress DXP Platform — Data Contract
**Version:** 1.2 · **Last updated:** 2026-04-14
**Owner:** Nathan (Ops/Strategy)  
**Repo:** `nathant30/dxp-ops` · `XPRESS_DATA_CONTRACT.md`

> **For Claude Code:** Read this file before touching `supabase/functions/databricks-proxy/index.ts` or any `DB_*` variable in `index.html`. Update the relevant section in the same commit when you add or change anything. Never remove a field marked Required.

---

## How This Contract Works

```
Databricks (xpressdbprod_catalog.xpress)  <- source of truth, read-only
    | SQL queries
Supabase Edge Function: databricks-proxy  <- the only translation layer
    | JSON via HTTPS POST { type, ... }
index.html global JS variables (DB_*)     <- frontend cache
    |
UI components                             <- what Nathan sees
```

**Edge function URL:** `https://nycqnxaxcpybjpvslwoc.supabase.co/functions/v1/databricks-proxy`
**Auth:** `apikey` header with anon key — `verify_jwt` MUST be `false`
**Request:** `POST` · `Content-Type: application/json` · body `{ "type": "<query_type>" }`
**Response:** `{ "data": [...], "today": "YYYY-MM-DD" }`

---

## Maintenance Rules

### When to update this file
Update this contract **in the same commit** whenever you:
- Add a new field to any edge function query
- Add a new query type (`type` value)
- Change a field name in the edge function
- Change a frontend `DB_*` mapping in `index.html`
- Change any StatusId grouping or VehicleTypeId scope

### What you must NEVER do without explicit instruction from Nathan
1. **Remove a Required field** — frontend silently shows P0 or "No data"
2. **Rename a field** — use exact snake_case names in this document
3. **Set `verify_jwt: true`** — frontend sends only `apikey`, not JWT; causes 401 on everything and kills all data
4. **Remove a VehicleTypeId** from `services_overview` — all 5 must always be present
5. **Rewrite the edge function from scratch** — extend, don't replace; rewriting drops required fields
6. **Change StatusId groupings** without updating this document

### Safe to do without updating this contract
- Adding a new query type (frontend ignores unknown types)
- Adding a new optional field to an existing query (frontend uses `|| 0` fallbacks)
- Improving SQL performance without changing output shape
- Adding SQL comments

---

## Reference Tables

### VehicleTypeId -> Service
| vtid | Service | Driver type | Xpress revenue model |
|---|---|---|---|
| `16` | Taxi (EV) | Salaried | 100% of Amount + BookingFeeAmount |
| `1` | Moto Regular | Marketplace | CommissionAmount (or Amount x 20%) |
| `2` | Go Car (Sedan) | Marketplace | CommissionAmount (or Amount x 20%) |
| `3` | Go SUV | Marketplace | CommissionAmount (or Amount x 20%) |
| `10` | e-Trike | Marketplace | **Flat P25 per completed booking** — NOT fare amount |

### StatusId Definitions
| StatusId | Meaning | Grouping |
|---|---|---|
| `1` | Queued — no driver yet | queued_now |
| `2` | Driver searching | queued_now |
| `3` | Driver assigned / en route to pickup | in_trip_now |
| `5` | Passenger in vehicle | in_trip_now |
| `7` | Completed | revenue/trip counts |
| `8` | Cancelled by user | cancelled |
| `10` | Expired — no driver accepted | expired |
| `11` | Active / ongoing | in_trip_now |
| `12` | Other active state | monitor only |

**Critical groupings — never change without updating here:**
- `queued_now` = StatusId IN (1, 2) only
- `in_trip_now` = StatusId IN (3, 5, 11) — StatusId 4 does NOT exist in this system
- `completed` = StatusId = 7
- `cancelled` = StatusId = 8
- `expired` = StatusId = 10

### Timezone Rule
All Databricks timestamps are UTC. Always use:
```sql
CONVERT_TIMEZONE('UTC', 'Asia/Manila', timestamp_column)
```
**Never use `+ INTERVAL 8 HOURS`** — use CONVERT_TIMEZONE consistently on both filter conditions and extracted values to avoid misaligned hourly buckets.

---

## Query Type Contracts

### `today_overview`
**Frontend variable:** `DB_TODAY`
**Triggered:** every dbRefreshAll()
**Used by:** Header pills (Revenue, Trips), buildCmd() hero card

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| total_revenue | number | YES | Header pill "PX", buildCmd hero |
| total_trips | number | YES | Header pill "Trips X" |
| active_drivers | number | YES | Fallback for "In Trip" pill |
| avg_fare | number | YES | BEP auto-calculation |
| total_cancelled | number | YES | buildCmd completion rate |
| total_expired | number | YES | buildCmd completion rate |
| snapshot_time | string | YES | Displayed as "as of HH:mm PHT" |

Scope: VehicleTypeId = 16 · today PHT

---

### `today_shifts`
**Frontend variable:** `ASHIFTS` array
**Triggered:** every dbRefreshAll()
**Used by:** buildCmd active shifts, buildAssets, Fleet shift board

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| vehicle_id | number | YES | Matched to ASSETS[].vehicle_id |
| rider_id | number | YES | ASHIFTS[].rider_id |
| driver_name | string | YES | Displayed in shift cards |
| shift_code | string | YES | AM1/AM2/PM1/PM2/PM3 |
| trips | number | YES | Trip count display |
| revenue | number | YES | Stored as gross_revenue |
| status | string | YES | 'active' or 'completed' |
| started_at_pht | string | YES | Stored as started_at |
| ended_at_pht | string | optional | Stored as ended_at |
| shift_hours | number | optional | Elapsed shift time |

Scope: VehicleTypeId = 16 · today PHT

---

### `revenue_by_asset`
**Frontend variable:** `DB_ASSET_REV` object keyed by asset_id
**Triggered:** every dbRefreshAll()
**Used by:** Asset cards, buildCmd fleet revenue

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| asset_id | number | YES | Matched to ASSETS[].vehicle_id -> asset_id key |
| revenue | number | YES | DB_ASSET_REV[asset_id] |

Scope: VehicleTypeId = 16 · StatusId = 7 · today PHT

---

### `driver_perf`
**Frontend variable:** updates D4W array in-place
**Triggered:** every dbRefreshAll()
**Used by:** Fleet driver cards, tier calculations

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| driver_id | number | YES | Matched to D4W[].id |
| trips | number | YES | d.trips |
| revenue | number | YES | d.rev -> computes rpd, rph |
| days | number | YES | d.days active |

Scope: VehicleTypeId = 16 · StatusId = 7 · rolling 7 days

---

### `services_overview`
**Frontend variables:** DB_SERVICES keyed by vtid string · DB_QUEUED_NOW · DB_IN_TRIP_NOW
**Triggered:** every dbRefreshAll(), also by Yesterday/MTD period buttons
**Used by:** Service cards, buildCmd hero, Performance KPIs, "In Trip" header pill

WARNING: Must include all 5 vtids — IN (1, 2, 3, 10, 16) — removing any vtid breaks its service card

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| vehicle_type_id | number | YES | Dict key — must be 1, 2, 3, 10, or 16 |
| trips | number | YES | Trip count on service cards |
| cancelled | number | YES | Completion rate |
| expired | number | YES | Completion rate |
| active_drivers | number | YES | "X active" on cards and Company KPIs |
| gtv | number | YES | Platform GTV on service cards |
| xpress_revenue | number | YES | Revenue on Taxi card and Performance section |
| commission | number | YES | Revenue on Moto/GoCar/GoSUV cards |
| avg_fare | number | YES | Displayed on cards and BEP calculation |
| queued_now | number | YES | "Waiting" count = StatusId IN (1,2) only |
| in_trip_now | number | YES | "In Trip" header pill = StatusId IN (3,5,11) |

e-Trike special rule: xpress_revenue for vtid 10 = COUNT(completed) x 25
Live counts (queued_now, in_trip_now) have NO date filter.
Historical counts filtered to today PHT.

---

### `rideshare_driver_perf`
**Frontend variable:** DB_RIDESHARE_PERF array
**Triggered:** every dbRefreshAll()
**Used by:** Rideshare driver tables, Performance Driver KPIs

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| driver_id | number | YES | d.id |
| driver_name | string | YES | d.name |
| vehicle_type_id | number | YES | d.vtid — service filter |
| trips | number | YES | d.trips |
| gtv | number | YES | d.gtv |
| commission | number | YES | d.commission |
| days | number | YES | d.days |
| avg_fare | number | YES | d.avg_fare |
| incentives | number | optional | d.incentives |
| avg_rating | number | optional | d.avg_rating |

Scope: VehicleTypeId IN (1,2,3,10) · StatusId = 7 · current month to date

---

### `demand_by_hour`
**Frontend variable:** DB_DEMAND_HOURS array
**Triggered:** every dbRefreshAll()
**Used by:** Hourly demand bar chart

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| pht_hour | number | YES | Hour 0-23 PHT, x-axis |
| vehicle_type_id | number | YES | d.vtid — filtered by service selector |
| completed | number | YES | Green bar height |
| cancelled | number | YES | Red bar overlay |
| total_requested | number | YES | Total demand |

Scope: VehicleTypeId IN (1,2,3,10,16) · StatusId IN (7,8,10) · today PHT

---

### `fleet_supply_health`
**Frontend variable:** DB_FLEET_HEALTH array
**Triggered:** dbRefreshAll() when on Fleet or Performance tab
**Used by:** Fleet driver list, ALL Performance tabs (Driver KPIs, Pod KPIs, Company KPIs, Unit Economics)

WARNING: This query powers the entire Performance section. Missing fields here cause P0 across all Performance tabs.

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| vehicle_type_id | number | YES | d.vtid |
| driver_id | number | YES | d.id |
| driver_name | string | YES | d.name |
| lifecycle | string | YES | d.lifecycle — New/Ramping/Established/Tenured |
| days_since_activation | number | YES | d.days_active |
| trips_today | number | YES | d.trips_today |
| rev_today | number | YES | d.rev_today — core Performance metric |
| commission_today | number | YES | d.commission_today |
| trips_mtd | number | YES | d.trips_mtd |
| rev_mtd | number | YES | d.rev_mtd |
| shift_hrs_today | number | YES | d.shift_hrs_today — TAXI ONLY, NULL for rideshare. Source: xpressridershiftlog capped 1-14h |
| in_trip_hrs_today | number | YES | d.in_trip_hrs_today — PickedUpAt to CompletedAt sum |
| avg_fare_today | number | YES | d.avg_fare — used for BEP auto-calc |
| avg_rating | number | YES | d.avg_rating |
| incentives_mtd | number | optional | d.incentives_mtd |

Frontend-derived (not from SQL):
  d.rev_per_shift_hr = d.shift_hrs_today > 0 ? Math.round(d.rev_today / d.shift_hrs_today) : 0

Scope: VehicleTypeId IN (1,2,3,10,16) · current month to date for MTD · today PHT for _today fields · LIMIT 1000

---

### `asset_mtd`
**Frontend variable:** DB_ASSET_MTD keyed by vehicle_id string
**Triggered:** dbRefreshAll() when on Assets tab
**Used by:** Asset drawer MTD panel

| Field | Type | Required | Frontend reads as |
|---|---|---|---|
| vehicle_id | number | YES | Dict key |
| trips_mtd | number | YES | d.trips_mtd |
| rev_mtd | number | YES | d.rev_mtd |
| days_active_mtd | number | YES | d.days_active_mtd |
| avg_fare_mtd | number | YES | d.avg_fare_mtd |
| am_trips_today | number | YES | d.am_trips_today — hour 4-13 PHT |
| am_rev_today | number | YES | d.am_rev_today |
| pm_trips_today | number | YES | d.pm_trips_today — hour >=14 PHT |
| pm_rev_today | number | YES | d.pm_rev_today |
| rev_today | number | YES | d.rev_today |
| utilized_hrs_today | number | YES | d.utilized_hrs |
| utilized_hrs_mtd | number | YES | d.utilized_hrs_mtd |

Scope: VehicleTypeId = 16 · current month to date

---

### `app_settings`
**Frontend variable:** OPS_DAY_CONFIG
**Triggered:** loadOpsConfig() at startup — routed through edge function for CORS
**Source:** Supabase app_settings table (NOT Databricks)

| Key | Type | Required | Frontend reads as |
|---|---|---|---|
| ops_day_config (key) | object | YES | Object.assign(OPS_DAY_CONFIG, value) |
| ops_day_config.day_start_hour | number | YES | When ops day resets (default: 4) |
| ops_day_config.am_shift | object | optional | { start_hour, end_hour, label } |
| ops_day_config.pm_shift | object | optional | { start_hour, end_hour, label } |

---

## Frontend Global Variables

| Variable | Populated by | Notes |
|---|---|---|
| DB_TODAY | today_overview | Taxi-only overview object |
| DB_SERVICES | services_overview | Keyed by vtid string |
| DB_QUEUED_NOW | services_overview | Keyed by vtid string |
| DB_IN_TRIP_NOW | services_overview | Keyed by vtid string |
| DB_RIDESHARE_PERF | rideshare_driver_perf | Array |
| DB_DEMAND_HOURS | demand_by_hour | Array |
| DB_FLEET_HEALTH | fleet_supply_health | Array — powers Performance section |
| DB_ASSET_REV | revenue_by_asset | Keyed by asset_id |
| DB_ASSET_MTD | asset_mtd | Keyed by vehicle_id string |
| ASHIFTS | today_shifts | Array of shift records |
| OPS_DAY_CONFIG | app_settings (Supabase) | Shift window config |
| PERF_CONFIG | localStorage['PERF_CONFIG'] | User-set targets |

---

## PERF_CONFIG Keys (localStorage, editable via Targets modal)

| Key | Default | Purpose |
|---|---|---|
| taxi_am_qty | 0 | Taxi AM shift driver count |
| taxi_pm_qty | 0 | Taxi PM shift driver count |
| taxi_rev_per_shift | 2600 | Revenue target per taxi shift (₱) |
| taxi_shift_hrs | 9 | Standard taxi shift length (hrs) |
| moto_am_qty | 0 | Moto Taxi AM shift driver count |
| moto_pm_qty | 0 | Moto Taxi PM shift driver count |
| moto_rev_per_shift | 1200 | Revenue target per moto shift (₱) |
| moto_shift_hrs | 9 | Standard moto shift length (hrs) |
| gocar_trips_target | 20 | Go Car daily trip target |
| gosuv_trips_target | 10 | Go SUV daily trip target |
| moto_trips_target | 50 | Moto Regular daily trip target |
| etrike_trips_target | 30 | e-Trike daily trip target |
| commission_pct_target | 20 | Rideshare commission target (%) |
| pod_rev_target | 2600 | Per-driver revenue target used for pod on-track calculation |
| asset_daily_target | 3000 | Daily revenue target per taxi asset — synced to ASET.defaultTarget on save |
| util_target | 70 | Taxi utilization target (%) |
| driver_shift_cost | 1432 | ₱/shift cost used in BEP formula |

BEP formula:
  bep_trips = Math.ceil(driver_shift_cost / avg_fare)
  avg_fare from DB_SERVICES['16'].avg_fare (live) or DB_TODAY.avg_fare (fallback)

Fleet daily target formula:
  fleetTarget = (taxi_am_qty + taxi_pm_qty) * taxi_rev_per_shift
              + (moto_am_qty + moto_pm_qty) * moto_rev_per_shift

---

## Shift Windows

| Code | Label | Start PHT | Driver target |
|---|---|---|---|
| AM1 | Morning 1 | 04:00 | 17 |
| AM2 | Morning 2 | 05:00 | 7 |
| PM1 | Afternoon 1 | 14:00 | 16 |
| PM2 | Afternoon 2 | 15:00 | 28 |
| PM3 | Afternoon 3 | 16:00 | 8 |

AM total: 24 drivers · PM total: 52 drivers · Fleet: 56 vehicles x 2 shifts = 112 shift-slots/day

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-14 | Initial contract from live codebase audit |
| 1.1 | 2026-04-14 | Added maintenance rules, CC instructions, changelog, update workflow |
| 1.2 | 2026-04-14 | PERF_CONFIG: replaced moto_qty with moto_am_qty+moto_pm_qty; added moto_trips_target, etrike_trips_target, pod_rev_target, asset_daily_target |
