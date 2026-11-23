# Reports — UI Integration Guide

**Audience:** UI / integration team
**Screen:** Reports (catalog → report view with Table / Chart tabs, Group By, pivots, filter chips, Export)
**Status:** code complete on `Deployment/QA_V1`; live after the next DLP API deploy + APIM registration

---

## 1. The one rule

> **One body drives everything.** Build one request object from the screen's controls
> (dates, mode/pivot, filter chips, search, sort, page) and send it — unchanged — to
> `summary`, `chart`, `grid` and the export. The server guarantees the four describe the same rows.

```
Headers: Authorization: Bearer <JWT> · ClientGUID: <tenant> · Content-Type: application/json
```

## 2. Endpoints

| Method | Route | Purpose |
|---|---|---|
| GET | `reports/catalog` | Every report: id, name, category, views, columns, sort keys, filter chips, KPI tiles. Cache it. |
| GET | `reports/{reportId}/definition` | One report's catalog entry |
| POST | `reports/{reportId}/filters` | Filter chips **with option values** for the window |
| POST | `reports/{reportId}/summary` | The 4 KPI tiles |
| POST | `reports/{reportId}/chart` | The chart (400 for File Tracing, which has none) |
| POST | `reports/{reportId}/grid` | One page of the table |
| POST | `reports/{reportId}/status` | **Status reports only** — the composite payload both tabs render (see below) |
| POST | `reports/export/{reportId}/grid/xlsx` | Excel: the filtered table (all pages) |
| POST | `reports/export/{reportId}/chart/xlsx` | Excel: KPIs + chart table + chart data |
| POST | `reports/export/{reportId}/full/xlsx` | Excel: all three sheets |

`reportId` values: `daily`, `weekly`, `monthly` (Summary — status reports), `upload`,
`blocked`, `tracing`, `risk`, `shadow`, `investigation`, `override`, `policy`,
`inventory`, `encryption`.

### Status reports (`daily` / `weekly` / `monthly`)

The three Summary reports compare one whole period — a UTC day, a Monday-to-Sunday
week, a calendar month — with the period immediately before it. Dates absent = the
last **complete** period; a request with dates gets the period containing `toDate`.

The catalog marks them with **`layout: "status"`** (every other report:
`layout: "grid"`). Switch screens on that flag, never on the category name or the
ids.

* **`POST reports/{id}/status`** returns everything in one read:
  * `sections[]` — the Table View: every metric with `display`, `previousDisplay`,
    `changeDisplay` ("+7.0%", "+4", "-1.2 pts"), raw `current`/`previous`/`delta`/
    `deltaPercent`, `direction` (up/down/flat) and `tone` (good/bad/neutral).
    Snapshot sections (file inventory, disk encryption) carry `snapshot: true` and a
    `note` — discovery data has no history, so they have no previous column.
  * typed blocks for the Chart View: `days` (zero-filled bars), `channels`,
    `topPolicies`, `riskIncreases`, `shadowAiTools`, `incidentSeverities`,
    `unencryptedDevices`, plus the window pair and the column labels
    (`currentLabel` / `previousLabel`).
  * **monthly only**: `trends` (six series × six months — dataEvents, blockedEvents,
    blockRate, policyHits, incidentsOpened, highRiskUsers) and `departments`
    (events / blocked / block rate / incidents / avgRisk / riskChange per
    department, "Unassigned" for users without one). Empty arrays on daily/weekly;
    the monthly Table View adds a *By department · events* section and its `section`
    filter accepts `departments`.
* **Zero vs "—"**: a channel, policy, department or shadow-AI tool absent from one
  period shows **0** for it (absence from a period-scoped dataset means zero events
  then), and a row that went quiet keeps its place. "—" appears only on snapshot
  sections, where no previous figure exists. `shadowAiTools` accordingly lists every
  tool seen in either period (`events: 0` + `previousEvents` for one that went
  quiet); the Overview tile counts this period's detections only.
* **Policy hits** are rows the policy engine governed: `PolicyAction` is a real
  verdict (Allow/Warn/Block), not `UnCovered`, the crunch's no-policy marker. The
  tile equals the sum of the per-policy hits. On a tenant with a catch-all policy
  every event is covered, so hits there legitimately equal Data events.
* **Period labels**: `currentLabel`/`previousLabel` are relative for the latest
  complete period ("This week" / "Last week") and dated for any past one
  ("Sep 14 – 20" / "Sep 7 – 13"; daily "Sep 16"; monthly "September 2026"; year
  appended when not the current one). Print them as-is.
* **Shadow AI name hygiene**: blank or purely-numeric tool names are dropped as
  agent noise, and variants like "DeepSeek" / "DeepSeek AI" merge into one tool
  (events sum; users take the largest variant; a decision on any variant decides
  the tool).
* `grid` / `summary` / the Excel export work as on every other report; the grid is
  the flattened Table View (filter it with the `section` chip). `chart` serves the
  one block the generic shapes carry — the daily events/blocked **trend** — which is
  also the Excel `full` export's chart sheet; the screen's Chart View renders the
  `status` payload.
* Web filtering, tenant restrictions and mean-time-to-close from the design are not
  served: the platform stores no data for them (the omit-rather-than-invent rule).

## 3. The request body

```json
{
  "fromDate": "2026-08-12",            // optional; both absent = last 30 days
  "toDate": "2026-09-10",              // date-only covers the whole day
  "mode": "incident",                  // Group By menu: incident|user · volume|endpoint · event|file
  "pivot": "user",                     // Upload Activity only: event|user|endpoint|url|app|device
  "filters": {                         // chip key → value or [values]; OR within a key, AND across keys
    "action": "Blocked",
    "content": ["PHI", "PII"]
  },
  "search": "ferrari",                 // contains-match across the report's text columns
  "sortBy": "attempts",                // one of the view's sortKeys (catalog); default per view
  "sortDescending": true,
  "page": 1,
  "pageSize": 25                       // 1–200
}
```

Everything is optional. Unknown `filters` keys, non-whitelisted `sortBy`, unknown `mode`/`pivot`,
inverted dates, ranges over 366 days and out-of-range paging come back as **400** with every
problem listed in `errors` — fix the request, do not retry blindly. Dates are UTC; `fromDate`
inclusive, the echoed `toDate` is the **exclusive** end.

## 4. Responses (all in the usual envelope `{ success, message, data, errors }`)

### Catalog item
```json
{
  "id": "upload", "name": "Upload Activity Log", "category": "Activity",
  "dateField": "Upload date", "groupByLabel": "None", "supportsDateRange": true,
  "hasChart": true, "chartKind": "trend", "defaultView": "event",
  "views": [{ "key": "event", "label": "Event", "kind": "pivot",
              "columns": [{ "key": "receivedTime", "label": "Date & time", "kind": "date", "align": "left", "flex": 1.1, "subKey": null, "sortKey": "receivedTime" }, ...],
              "sortKeys": ["receivedTime","user","endpoint","application","content","size","action"],
              "defaultSort": "receivedTime", "defaultSortDescending": true }, ...],
  "filters": [{ "key": "fileSize", "label": "File size", "source": "static",
                "options": [{ "value": "lt1mb", "label": "Under 1 MB" }, ...] }, ...],
  "kpis": [{ "key": "attempts", "label": "Upload attempts" }, ...]
}
```
* `views.kind`: `pivot` → send as `pivot`; `mode` → send as `mode`; `default` → send nothing.
* `columns.kind`: `text` · `number` · `date` (UTC ISO) · `size` (bytes, format client-side) ·
  `tag` (chip) · `percent` (0–100) · `list` (string array). `subKey` names the row key for the
  second line under the value. `sortKey` null = not sortable server-side.
* `supportsDateRange: false` (inventory, encryption) → hide the date control; the window is ignored.

### Grid
```json
{
  "reportId": "upload", "view": "event", "fromDate": "...", "toDate": "...",
  "columns": [...same as the view...],
  "rows": [ { "id": 88214, "receivedTime": "2026-09-10T14:22:00Z", "user": "Mac Ferrari",
              "designation": "Finance Controller", "endpoint": "IN-LT-4482", "application": "Chrome",
              "url": "https://drive.google.com/...", "content": "PFI", "size": 50530304, "action": "Blocked", "policy": "Finance — payroll egress" } ],
  "totalCount": 18420, "page": 1, "pageSize": 25, "totalPages": 737,
  "appliedSort": { "sortBy": "receivedTime", "sortDescending": true },
  "appliedFilters": [{ "key": "action", "label": "Action", "values": ["Blocked"] }]
}
```
Rows are dictionaries keyed by column key and sub-key; extra keys (`id`, `userId`,
`endpointUserId`, `runId`, `traceKey`, `policyId`, `url`, `fileName` …) are there for
drill-downs and may be ignored. Empty state: `totalCount = 0`, `rows = []`.

### Summary
```json
{ "kpis": [ { "key": "blocked", "label": "Blocked", "value": 1284, "display": "1,284", "tone": "danger" }, ... ] }
```
`tone`: `default` · `danger` · `warning` · `success` — the colour the design gives the tile.
`display` is pre-formatted ("41 TB", "38 / 100"); `value` is the raw number (null when not numeric).

### Chart
```json
{ "kind": "trend", "title": "Upload attempts over the period", "note": "Daily volume, allowed against blocked",
  "labels": ["12 Aug", "13 Aug", ...],
  "datasets": [ { "key": "allowed", "label": "Allowed uploads", "values": [312, 298, ...], "percentages": null },
                { "key": "blocked", "label": "Blocked uploads", "values": [41, 37, ...] } ],
  "points": [], "total": 18420 }
```
| `kind` | Use |
|---|---|
| `trend` | `labels` = days; datasets `allowed`, `blocked`, `total` |
| `donut`, `stack`, `funnel`, `treemap`, `columns` | `labels` = slices/stages/tiles/buckets; one dataset with `values` and `percentages` (of `total`) |
| `grouped` | `labels` = top-5 policies; datasets `blocks`, `overrides`, `approvals` |
| `scatter` | `points[]` with `label`, `x` (users), `y` (uploads), `size` (sensitive uploads), `series` (status) |

### Filters
Same shape as the catalog's `filters`, but `source: "data"` chips carry the values that exist in
the tenant's data for the window (applications, content types, policies, classifications, disk
states …). `source: "text"` accepts any typed value.

**People, machines and policies send an id.** Wherever a report publishes `user`, `endpoint`
or `policy`, its options are `{ "value": "<id>", "label": "<name>" }` — show the label, send
the value. All three also accept a **name**, so these work and mean the same thing:

```json
"filters": { "user": ["14"] }                  // the row's userId — exact, survives renames
"filters": { "user": ["Mac Ferrari"] }         // the row's user (display name)
"filters": { "user": ["ITH8"] }                // employee code
"filters": { "endpoint": ["4482"] }            // the row's endpointUserId
"filters": { "endpoint": ["IN-LT-4482"] }      // machine name
```

Use them for the drill-down funnels: take `userId` / `endpointUserId` off a row in the **User**
or **Endpoint** pivot and post it back with `pivot: "event"` to land on that person's or that
machine's events. Prefer the id — two people can share a display name, and a name can change
mid-window while the id does not.

Which reports publish which:

| Filter | Reports |
|---|---|
| `user` | upload, blocked, tracing, risk, investigation, override, encryption |
| `endpoint` | upload, inventory, encryption |
| `policy` | blocked, override, policy |

**A numeric value is tried as the id *and* as a name**, never as the id alone — a person whose
display name or employee code happens to be digits is still found.

**`endpoint` means the id that report's own rows carry.** On Upload Activity that is
`endpointUserId`; on Classified File Inventory and Disk Encryption it is `endpointId`. Take the
id off the row you are drilling from and it is always the right one.

Two behaviours worth knowing:

- **Option lists do not shrink as you filter.** They are built from the report's own population
  for the window — everyone who appears in *that* grid — so Blocked Uploads lists only people
  who were blocked, and User Risk Scores lists everyone the snapshots scored even if they moved
  no files. Picking an option always returns at least one row.
- **File Tracing filters the events, not the chain.** A file is multi-path because of everything
  that happened to it, so filtering by user shows that person's hops within files that still
  qualify. Scoping the scan instead would hide a file they uploaded and a colleague copied,
  which is the case tracing exists to surface.

## 5. Export

```
POST reports/export/upload/grid/xlsx      body = the grid body
→ 200  the .xlsx (Content-Disposition: attachment; filename=Upload_Activity_Log_event_20260914_103000.xlsx)
→ 400  invalid body / format (JSON envelope)
→ 404  unknown report
```
* `page`/`pageSize` are ignored — the file holds the complete filtered set, all pages.
* The data sheet is capped (server default 50,000 rows; send `"maxRows": n` to lower it).
  Response headers: `X-Report-Row-Count`, `X-Report-Total-Rows`, `X-Report-Truncated`
  (`true` when the cap applied; the file's first line then reads "first N of M rows").
* Sheets: **Data** (grid export), **Summary + Chart Data** (chart export), all three (full export).
  The Data sheet is an Excel table with AutoFilter and a frozen header; dates are UTC.
* Only `xlsx` exists. Same `saveBlob` code as the Log Summary export (read the filename from
  `Content-Disposition`; `Access-Control-Expose-Headers` is set).

## 6. Suggested flow

1. On first load `GET reports/catalog` → build the catalog table and the Analytics drawer
   (group by `category`).
2. Opening a report: read its `views` → render the pivot tabs (`kind: pivot`) or the Group By
   menu (`kind: mode`); read `filters` → render chips; `POST filters` for option values.
3. **Run Report** → send the body to `summary`, `chart` (if `hasChart`) and `grid` in parallel.
4. Pager / rows-per-page / column sort → change `page`, `pageSize`, `sortBy`, `sortDescending`
   and re-post `grid` only.
5. Export ▾ → post the same body to the export route; branch on status code; save the blob.

## 7. Facts to rely on

* Filtered `totalCount` for `filters: { "action": "Blocked" }` equals the **Blocked** tile
  (blocks that were later overridden count in both; the row's `action` shows the final state).
* Snapshot reports (`inventory`, `encryption`) ignore dates — `supportsDateRange` says so.
* `tracing` has no chart; `chart` answers 400 for it. Its summary works.
* Sort direction defaults to the view's `defaultSortDescending`; send `sortDescending`
  explicitly when the user clicks a header.
* **`fileSize` on Upload Activity is now `size`**, matching the column it filters. The old key
  still works — the server canonicalises it — but `appliedFilters` echoes `size`, and the
  catalog carries `"aliases": ["fileSize"]` on that filter. Switch when convenient.
* Override & Approval Requests now has **Type** and **Policy** columns, so its `type` and
  `policy` filters point at something visible. Both were already in every row.
* A filter key always names a column of at least one of the report's views, except for
  dimensions the design deliberately does not draw: `engine`, `detected`, `classification`,
  `diskType`, `method`, `department`, `movement`, `destination`, `band`, `state`, `outcome`.
  A test enforces that list, so it cannot drift silently.
* Reports answer in the tenant of the `ClientGUID` header, nothing in the body can change that.

## 8. Availability checklist (backend/ops — for transparency)

1. DLP API deployed from `Deployment/QA_V1`.
2. No database script: the reports read existing tables and indexes.
3. APIM operations on `qadlp`: `GET /reports/catalog`, `GET /reports/{reportId}/definition`,
   `POST /reports/{reportId}/{filters|summary|chart|grid}`,
   `POST /reports/export/{reportId}/{grid|chart|full}/{format}`.
4. Optional `appsettings.json`: `Reports:MaxExportRows`, `Reports:ExportBatchSize`.
