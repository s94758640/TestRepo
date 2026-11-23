# Reports Module — Implementation Plan and Design Record

**Status:** implemented on `Deployment/QA_V1` · **Owner:** DLP API team · **Date:** 2026-09-14
**UX reference:** "DLP Reports" design (catalog + report view with Table / Chart tabs, Group By, pivots, filter chips, Export)

This document records what was found in the existing solution, how each element of the
Reports design maps onto data the platform already stores, where the design asked for
something the data cannot supply, and how the module is built. The UI contract lives in
`Reports-UI-Integration.md`.

---

## 1. Discovery — what already existed

| Area | Finding | Consequence for Reports |
|---|---|---|
| Runtime | .NET 8, EF Core 9 + Npgsql, PostgreSQL | EF LINQ with `EF.Functions.ILike` / `JsonExistAny`; no raw SQL |
| Architecture | `SFM.DLP.API` (controllers) → `SFM.DLP.Application` (services, DTOs) → `SFM.DLP.Infrastructure` (repositories, `QaSfmDlpContext`) → `SFM.DLP.Domain` (entities) | Reports follows the File Trace module: repository in Infrastructure, service + handlers + DTOs in Application, thin controllers |
| Tenancy | One database per client. `Program.cs` resolves the connection from the `ClientGUID` header (or claim) when `QaSfmDlpContext` is created | Every report query is tenant-scoped by construction; no request carries a tenant id |
| Auth | JWT bearer; `BaseController` carries `[Authorize]`; claims `UserId`, `ClientId` | Both report controllers derive from `BaseController` |
| Envelope | `ApiResponse<T>` (`success`, `message`, `data`, `errors`), `ApiExceptionMiddleware` first in the pipeline, model-binding failures also answer in the envelope | Reused as-is |
| Requests | `QueryParameters` with multi-value `LikeFilters`/`ExactFilters` and the `MultiValueFilter*Converter` (single value or array) | Reports reuse the converter for its `filters` map |
| Paging | `PagedResultDto<T>`, `PaginatedResponse<T>`, `TransferLogsPageDto` — three shapes in use | Reports return one explicit `ReportGridResponse` (rows + columns + paging) |
| Logging | `ILogRepository.LogAsync` per action with a stopwatch; `ILogger` for background work | Reused |
| Excel | **ClosedXML 0.104.2 (MIT)** already referenced by the API for `TransferLogExportBuilder`; itext7 for PDF | Reused; the same package is now referenced by `SFM.DLP.Application` so the builder sits beside the handlers and is unit-tested |
| Export precedent | `ExportTransferLogs/{format}`: same body as the grid, page ignored, 5,000-row sync/async switch with `dlp."ExportJob"` | Reports keep "body = grid body, page ignored"; large files are capped rather than queued (see §7) |
| Tests | xUnit in `SFM.Platform.Storage.Tests`, no database provider | Reports tests run against an in-memory repository fake and read produced workbooks back with ClosedXML |
| Indexes | `crunch."LandingLogsCrunched"`: `(ReceivedTime)`, `(UserID, ReceivedTime)`, `(EndpointUserId, ReceivedTime)`, `(AppDeviceName, ReceivedTime, IsDevice)`; `discovery."File"`: `(EndpointId, IsSensitive)`, GIN on `Classifications` | Every crunch query starts from the `ReceivedTime` window; pivots group on the indexed keys; classification filters use `?\|` on the GIN index |

### Source tables per report

| Report (design id) | Tables |
|---|---|
| Upload Activity Log (`upload`) | `crunch."LandingLogsCrunched"`, `sfm."User"`, `sfm."Endpoint"`, `dlp."EndpointDailyStatus"` |
| Blocked Uploads by Policy (`blocked`) | `crunch."LandingLogsCrunched"` |
| File Tracing (`tracing`) | `crunch."LandingLogsCrunched"`, `dlp."ExtensionChangeLog"` |
| User Risk Scores (`risk`) | `public.dlp_core_dailyrisksnapshot`, `sfm."User"`, `crunch."LandingLogsCrunched"`, `dlp."OverrideRequest"` |
| Shadow AI Detections (`shadow`) | `public.defensly_shadow_ai_runs`, `…_tool_findings`, `…_tool_approvals` (via existing `IShadowAiGovernanceRepository`) |
| Incident Investigations (`investigation`) | `crunch."LandingLogsCrunched"` (IsInvestigation), `public.defensly_user_incident_investigator`, `sfm."User"` |
| Override & Approval Requests (`override`) | `dlp."OverrideRequest"`, `sfm."User"` |
| Policy Enforcement Summary (`policy`) | `dlp."Policy"`, `sfm."UserPolicyMap"`, `dlp."PolicyRuleAppMap"`, `dlp."PolicyRuleDeviceMap"`, `dlp."PolicyRuleContentType"`, `crunch."LandingLogsCrunched"`, `dlp."OverrideRequest"`, `sfm."User"` |
| Classified File Inventory (`inventory`) | `discovery."File"`, `discovery."Volume"`, `discovery."ScanSummary"`, `sfm."Endpoint"` |
| Disk Encryption Status (`encryption`) | `discovery."Volume"`, `sfm."Endpoint"`, `sfm."User"` |

---

## 2. Design → API mapping

### 2.1 Screen structure

| UI element | API | Notes |
|---|---|---|
| Catalog table (name, category, "System Generated") | `GET reports/catalog` | Static per deployment |
| Analytics drawer (folders by category) | same catalog, grouped client-side by `category` | |
| Report header (category, name, date range) | `ReportSummaryResponse.FromDate/ToDate` echo the applied window | |
| "Last 30 days" date preset | body `fromDate` / `toDate`; absent = last 30 days | |
| Filter chips (+ options) | `POST reports/{id}/filters` | static options from the catalog, data-driven options for the window |
| "+ Filter" / "Clear filters" | body `filters` map | unknown keys → 400 |
| Run Report | `summary` + `chart` + `grid` with the same body | |
| Table View grid | `POST reports/{id}/grid` | server-side filter/sort/page, `totalCount` |
| Rows per page 10/25/50/100 | `pageSize` (1–200) | |
| Pager | `page`, `totalPages` | |
| Column sort | `sortBy` (whitelisted per view), `sortDescending` | |
| Group By menu (Incident/User, Volume/Endpoint, Event/File) | body `mode` | |
| Upload Activity pivot tabs | body `pivot` (`event`/`user`/`endpoint`/`url`/`app`/`device`) | |
| Chart View KPI strip | `POST reports/{id}/summary` | 4 tiles per report, keys in the catalog |
| Chart View chart | `POST reports/{id}/chart` | `kind` + labels/datasets or points |
| Export ▾ | `POST reports/export/{id}/grid\|chart\|full/xlsx` | same body as the grid |
| Empty state ("No rows match these filters") | `totalCount = 0`, `rows = []` | |
| Drill-down (view trace, user detail) | rows carry ids (`id`/`enrichedId`, `userId`, `endpointUserId`, `runId`, `traceKey`) | existing detail endpoints take them |

### 2.2 Per-report component mapping

Each row: **design element → endpoint → handler → query → response field**. Handlers live in
`SFM.DLP.Application/Services/Reports/Handlers`; queries in
`SFM.DLP.Infrastructure/Repositories/ReportsRepository.*.cs`.

**Upload Activity Log** (`UploadActivityReportHandler`)

| Element | Query (`ReportsRepository.Uploads.cs`) | Response |
|---|---|---|
| KPIs: attempts / blocked / overridden / endpoints | `GetUploadSummaryAsync` — one `GROUP BY 1` row + 3 distinct counts | `summary.kpis[attempts|blocked|overridden|endpoints]` |
| Trend chart (allowed vs blocked per day) | `GetUploadDailyTrendAsync` — group by year/month/day | `chart.kind=trend`, datasets `allowed`, `blocked`, `total`, zero-filled labels |
| Event grid | `GetUploadEventsAsync` — window + filters, sort whitelist, `OFFSET/LIMIT` | columns `receivedTime, user(+designation), endpoint, application(+url), content, size, action` |
| User pivot | `GetUploadsByUserAsync` (GROUP BY UserID) + `GetUserDirectoryAsync` (department, email) + `GetTopContentTypeByUserAsync` | `user(+email), department, attempts, blocked, overridden, topContent, lastAttempt` |
| Endpoint pivot | `GetUploadsByEndpointAsync` (GROUP BY EndpointUserId) + `GetLatestByEndpointUserAsync` + `GetEndpointDirectoryAsync` (latest daily status) | `endpoint, primaryUser(+designation), os, attempts, blocked, agent, lastAttempt` |
| URL pivot | `GetUploadsByUrlAsync` (GROUP BY URL) + `GetLatestByUrlAsync` | `url(+application), users, attempts, blocked, volume, state` |
| App pivot | `GetUploadsByApplicationAsync` (IsDevice = false) | `application, users, attempts, blocked, blockRate` |
| Device pivot | `GetUploadsByDeviceAsync` (IsDevice = true) + `GetLatestByAppDeviceAsync` | `device, type, endpoint, user, transfers, volume, action` |
| Filters: Application, Content, File size, Action | `UploadBase()` predicate | data / data / static / static |
| Filters: User, Endpoint (drill-down, not chips in the design) | `UserPredicate` / `EndpointPredicate` in `UploadBase()`; options from `GetDistinctUsersAsync` / `GetDistinctEndpointsAsync` | data; each value matches an id **or** a name (see §3) |

**Blocked Uploads by Policy** (`BlockedUploadsReportHandler`) — same base query with `BlockedOnly = true`.

| Element | Query | Response |
|---|---|---|
| KPIs: blocks / AI / rule / policies triggered | `GetUploadSummaryAsync` (Total, AiScanned, Total−AiScanned, DistinctPolicies) | |
| Donut by detected classification | `GetUploadCountsByContentTypeAsync` | `chart.kind=donut`, percentages of total |
| Grid | `GetUploadEventsAsync` | `receivedTime, user, policy, application, detected, engine (AI/Rule), outcome (Blocked/Overridden/Pending)` |
| Filters | policy, detected (content), engine (IsAIScan), outcome (override flags), application | |

**User Risk Scores** (`UserRiskScoresReportHandler`) — derived, in memory.

| Element | Query (`ReportsRepository.Risk.cs`, `.Uploads.cs`, `.Overrides.cs`) | Response |
|---|---|---|
| Score / band / status | `GetRiskWindowsAsync` — peak `weighted_risk_score_v2_d` per user over the window, band from the peak day (`RiskBands.FromBaseline`) | `score`, `band`, `status`, `average` |
| Trend vs previous window | `GetPeakScoresAsync` over the preceding window of equal length | `trend` ("+14", "New"), `trendDelta` |
| Incidents | `CountBlockOrWarnByUserAsync` (Block + Warn crunch rows) | `incidents` |
| Overrides "n of m" | `CountSelfOverridesByUserAsync` ÷ `TrayAppSettings.MonthlySelfOverrideLimit` | `overrides`, `overridesUsed`, `overridesLimit` |
| Department / email | `GetUserDirectoryAsync` | |
| KPIs | users scored, High band (High + Critical), Medium band, average peak | |
| Columns chart | users per score bucket 0–20 … 81–100 | |

**Shadow AI Detections** (`ShadowAiReportHandler`) — latest run in the window via the existing `IShadowAiGovernanceRepository`; findings' `evidence` JSON gives users / uploads / sensitive uploads; the tenant's tool approvals overlay the agent's verdict → `Whitelisted` / `Pending` / `Blocked`. Scatter chart: X = users, Y = uploads, size = sensitive uploads.

**Incident Investigations** (`InvestigationsReportHandler`)

| View | Query | Response |
|---|---|---|
| Incident (default) | `GetUploadEventsAsync` with `InvestigationOnly = true` + `GetLatestInvestigatorRunByUserAsync` + user directory | `incidentId (INC-{EnrichedID}) (+reportId INV-{run})`, `detected`, `user(+department)`, `finding`, `classification`, `severity`, `state (Open/Closed from ProcessStatus)` |
| User | `GetInvestigatorRunsAsync` (SQL join to users, ILIKE search, severity CASE sort) + `CountInvestigationsByUserAsync` | `reportId, user(+department), opened, incidents, primaryFinding, confidence, severity` |
| KPIs | runs total (`GetInvestigatorRunCountsAsync`), open / closed / critical incidents (`CountInvestigationsAsync`) | |
| Funnel | reports generated → key finding identified → routed to human (confidence < 70) → escalated (High/Critical) → incidents still open | |

**Override & Approval Requests** (`OverridesReportHandler`) — `GetOverrideRequestsAsync` (SQL, left joins to requester/reviewer), `GetOverrideSummaryAsync`, `CountSelfOverridesByUserAsync` for the quota column; stack chart of self / admin approved / pending / denied / cancelled.

**Policy Enforcement Summary** (`PolicyEnforcementReportHandler`) — `GetPoliciesAsync` + `GetPolicyActivityAsync` (crunch grouped by PolicyName) + `GetBlockedContentByPolicyAsync` (top trigger) + `CountAdminApprovedByPolicyAsync` + `GetPolicyRuleCountsAsync` (the "18 apps, 6 content types" sub-line) + `GetActiveUserPolicyMapsAsync`/`GetActiveUserScopesAsync` (users covered through the same hierarchy `UserDetailsService.GetPolicyMapForUser` uses). Grouped chart: top five policies by blocks / overrides / approvals.

**Classified File Inventory** (`FileInventoryReportHandler`) — `GetDiscoveryFilesAsync` (SQL join File↔Volume, `jsonb ?| text[]` classification filter, paged), `GetDiscoveryInventoryCountsAsync`, `CountDiscoveryFilesByClassificationAsync` (one indexed count per class) for the treemap.

**Disk Encryption Status** (`DiskEncryptionReportHandler`) — `GetDiscoveryVolumesAsync` (SQL filters) shaped per volume or rolled up per endpoint (posture: Encrypted / Partial / Suspended / Unencrypted); donut of volume states.

**File Tracing** (`FileTracingReportHandler`) — `GetTraceEventsAsync` (thin crunch rows for the window, capped at 100,000) + `GetRenameEventsAsync`; chain = lower-cased file name; movement classified from transfer type / app / device flag (mirrors `FileTraceService.ClassifyTransfer`); only chains with ≥ 2 movement types are traced. Event view and File view; no chart (as designed).

---

## 3. Deviations from the design and assumptions

Where the design showed something the database does not hold, the column was **omitted**
rather than invented. Each case:

| Report | Design element | Decision |
|---|---|---|
| Upload / App pivot | VENDOR, POSTURE | No vendor or posture data exists for applications — omitted. |
| Upload / URL pivot | CATEGORY (Cloud storage, AI assistant …) | Crunch rows carry the application *name*, not a category taxonomy — omitted; the latest application name is the sub-line instead. |
| Upload / URL, Device pivots | STATE / ACTION tag per group | A group has many verdicts; the tag shows the **most recent row's** verdict (same rule `GetTransferLogs` applies to `AppliedPolicy`). |
| Upload / Device pivot | VID/PID sub-line | Not stored — omitted. TYPE = the device master name the crunch stamps into `ApplicationCategoryDescription`. |
| Upload / Endpoint pivot | AGENT Active/Offline | From the latest `dlp."EndpointDailyStatus"` row: `Offline` → Offline, anything else → Active, no row → Unknown. |
| Upload | Action filter semantics | "Blocked" = policy verdict Block **including** blocks later overridden (so the filtered total equals the Blocked tile); "Overridden"/"Pending" = override state. The ACTION column shows the final state. |
| Blocked | OUTCOME "Approved" vs "Overridden" | Both are `IsOverrideApplied`; distinguishing self from admin approval needs a per-row join to `OverrideRequest` — shown as "Overridden". |
| Risk | INCIDENTS | Block + Warn crunch rows for the user in the window. |
| Risk | OVERRIDES "n of 6" | Self-overrides in the window over `TrayAppSettings:MonthlySelfOverrideLimit` (default 5). |
| Risk | TREND 30D | Peak in the window minus peak in the preceding window of equal length; "New" when the user had no earlier snapshot. |
| Risk | High band KPI | High **or** Critical (`RiskBands` thresholds: ≥ 65). |
| Shadow AI | ENDPOINTS, CATEGORY | Not in the findings' evidence — ENDPOINTS omitted; CATEGORY = the agent's finding type. |
| Shadow AI | Date range ("First seen") | The window filters the **scan** that reported the tool (`defensly_shadow_ai_runs.CreatedAt`), not the FIRST SEEN column — a tool in use since August is still a live detection in an October window, and filtering on first-seen would hide it. `dateField` says **"Detection date"** so the label names the filter. |
| Investigations | "Median time to close" KPI | Crunch rows have no close timestamp — replaced by **Closed cases**. |
| Investigations | Funnel stages "Findings confirmed / Action taken" | Not recorded — stages that exist are used (key finding identified, routed to human, escalated, still open). |
| Investigations | State filter in User view, Classification filter in User view | Runs have neither — those filters apply to the Incident view only. |
| Policy | STATE "Draft" | No draft concept — Active / Inactive from `Policy.IsActive`. |
| Inventory | ENGINE (AI/Rule) | Not stored for discovered files — omitted. |
| Inventory | Date range ("Last modified") | Snapshot dataset; the window is **not applied** (`supportsDateRange = false`). Applying it would empty the "stale over 12 months" tile. |
| Encryption | DISK TYPE SSD/HDD | `Volume.Kind` (Local / External / Mapped) is what the agent reports. |
| Encryption | Date range | Snapshot; not applied. |
| Tracing | KPIs in Chart View | The design hides the chart tab for tracing; the summary endpoint still answers (multi-path files, traced events, 3+ paths, reached removable media). |
| Tracing | "Deleted" movement | No delete events are captured today — not offered. Renames come from `dlp."ExtensionChangeLog"`. |
| All | "+ Filter" on reports without chips | Filters mapped 1:1 to grid columns are offered (policy, engine, outcome, band, department, status, severity, state, decision, type, classification, encryption). |
| Override | `type` / `policy` columns (**added beyond the design**, 2026-09-14) | The design draws seven columns and no Type or Policy, but the catalog published filters for both and every row already carried the values. Offering to filter on a dimension the grid never shows is the worse half of that trade, so the columns were added rather than the filters removed — "which policy did this override bypass" is a question a governance report should answer on its face. Both sort keys were already whitelisted. |
| Upload | `fileSize` filter renamed to `size` (2026-09-14) | The key now matches the column it filters, which is the convention every other filter follows. `fileSize` is kept as an alias (`ReportFilterDefinition.Aliases`, canonicalised by the normaliser and published in the catalog DTO) so nothing already sending it breaks; `appliedFilters` echoes the canonical key. |
| All | `user` / `endpoint` / `policy` filters on nine more reports (**added beyond the design**, 2026-09-14) | Requested by the UI team after the Upload funnels shipped: `user` on blocked, tracing, risk, investigation, override and encryption; `endpoint` on inventory and encryption; `policy` on policy. Same id-or-name rule as Upload's. Each report matches the id **its own rows carry** — `endpointUserId` on Upload's endpoint pivot, `endpointId` on the two discovery reports — because that is what a drill-down has to hand. Where the id is not a column on the filtered table (override requesters, investigation subjects) the name is resolved through a subquery on `sfm."User"` so the filter stays in SQL and the summary tiles and grid share it. |
| Tracing | `user` filter narrows events, not the scan | A chain is multi-path because of everything that happened to the file. Scoping the SQL scan to one person would make a file they uploaded and a colleague copied look single-movement and drop it from the trace entirely — the opposite of what tracing is for. So `user` joins `movement`, `destination` and `content` as a post-chain event filter. |
| All | Option lists take the report's **base** filter | A picklist is built from window + the report's own restriction (`BlockedOnly`, `InvestigationOnly`), never the caller's current selections, so it holds exactly the values that report's grid can show and does not shrink as the analyst narrows. Risk and override draw their populations from their own tables rather than the transfer log, because a user can be scored, or raise an override, in a window in which they moved nothing. |
| Upload | `user` / `endpoint` filters (**added beyond the design**, 2026-09-14) | The design draws four chips; the UI's drill-down funnels need to scope the event view to one person or machine, so two data-driven keys were published. Each value matches an id (`UserID` / `EndpointUserId` — what a pivot row carries) **or** a name (`Username`, `EmployeeCode` / `MachineName`), case-insensitively. A numeric value is matched against the id *and* the text columns rather than the id alone: treating digits as "must be an id" would silently drop a person whose name or code is numeric, and a superset can only over-match, never hide. Options are `{value: id, label: name}` so a picklist selection survives a rename or a duplicate display name. Consequence: the `(UserID, ReceivedTime)` / `(EndpointUserId, ReceivedTime)` composites cannot serve the OR, so the window index carries the query — acceptable because the window is capped at 366 days. On `blocked` / `investigation` / `tracing` the plumbing is in place (`UploadActivityFilter.Users` / `.Endpoints`) but no catalog key is published, so those reports are unchanged. |
| All | Time zone | Timestamps are returned and exported in UTC (as stored); the header line says so. |

---

## 4. Architecture

```
SFM.DLP.API
 ├── Controllers/ReportsController.cs          reports/catalog, {id}/definition|filters|summary|chart|grid
 └── Controllers/ReportsExportController.cs    reports/export/{id}/grid|chart|full/{format}

SFM.DLP.Application
 ├── DTOs/Reports/ReportDtos.cs                 request/response contracts
 ├── Reports/ReportCatalog.cs                   the whitelist: views, columns, sort keys, filters, KPIs
 ├── Reports/ReportQuery.cs                     normalised query + validation (one interpretation)
 ├── Reports/ReportExceptions.cs                400 / 404 signals
 └── Services/Reports/
      ├── ReportsService.cs                     IReportsService — dispatch by report id
      ├── ReportHandlerBase.cs                  IReportHandler, shared factories, in-memory sort/page
      ├── Handlers/*ReportHandler.cs            one per report (10)
      ├── ReportExportService.cs                IReportExportService — same query, streamed rows, cap
      └── ReportWorkbookBuilder.cs              ClosedXML workbook

SFM.DLP.Infrastructure
 ├── Repositories/Interfaces/IReportsRepository.cs   filters + row projections (Application-free)
 └── Repositories/ReportsRepository*.cs              EF queries per data source
```

Filter consistency is structural: `ReportQueryNormalizer.Normalize` runs once per request and
every handler read (summary, chart, grid, export rows) takes that `ReportQuery`. SQL-backed
handlers build one predicate (`UploadBase`, `OverrideBase`, `DiscoveryFileBase`, …) that all
their aggregates and pages share; derived handlers load one row set and answer everything
from it. `ReportsServiceAndExportTests.Summary_chart_and_grid_are_asked_the_same_question`
asserts it.

---

## 5. Query design

For each SQL-backed read:

```
crunch."LandingLogsCrunched"                          (Upload, Blocked, Incidents)
  WHERE ReceivedTime >= @from AND ReceivedTime < @to  → IX_LandingLogsCrunched_ReceivedTime
    [AND PolicyAction = 'Block'] [AND IsInvestigation] [AND lower(AppDeviceName) IN (...)]
    [AND (ContentType ILIKE ... OR ...)] [AND size band] [AND action/outcome predicate]
    [AND (Username ILIKE @p OR MachineName ILIKE @p OR ...)]
  → GROUP BY 1 (tiles) | GROUP BY date parts (trend) | GROUP BY ContentType (donut)
  → ORDER BY <whitelisted column>, EnrichedID  OFFSET @skip LIMIT @take   (event grid)
  → GROUP BY UserID / EndpointUserId / URL / AppDeviceName  … ORDER BY aggregate OFFSET/LIMIT (pivots)
     then, for the page's keys only: latest-row fields, directory, top content
```

Pivot groupings use the existing composites `(UserID|EndpointUserId|AppDeviceName, ReceivedTime)`;
the URL grouping (deliberately un-indexed upstream because URL values can exceed a btree entry)
relies on the window index, as `GetTransferLogs` does. The "latest row per group" pattern is run
only for the keys on the current page, so its cost is proportional to the page, not the window.

```
dlp."OverrideRequest"  WHERE DateRequested in window [status/type/policy/search] → GROUP BY 1 | LEFT JOIN sfm."User" ×2 ORDER BY … OFFSET/LIMIT
public.dlp_core_dailyrisksnapshot  WHERE SnapshotDate in window → GROUP BY user_id (peak, avg, sums, peak-day fields)
public.defensly_user_incident_investigator WHERE created_at in window [severity] → GROUP BY 1 | LEFT JOIN sfm."User" … ORDER BY CASE severity …
discovery."File" JOIN discovery."Volume"  WHERE IsSensitive AND NOT IsDeleted [Classifications ?| @labels → GIN] [volume status] [search] → GROUP BY 1 | ORDER BY … OFFSET/LIMIT
discovery."Volume" WHERE IsActive [status/kind/method]  (≈ endpoints × volumes rows; shaped in memory)
```

**No new indexes are needed.** Every window predicate hits an existing index; the discovery
classification filter hits the existing GIN index. If the URL pivot proves slow on a large
tenant, the upstream note in `AddGroupingIndexesToLandingLogsCrunched.sql` applies (URL is not
indexable as a btree key); narrowing the window is the mitigation.

In-memory work is bounded: risk (one row per scored user), policy (tens of policies), shadow
AI (one scan's findings), encryption (volumes ≈ 2–3 × endpoints), file tracing (thin rows,
capped at 100,000 events + 50,000 renames; a capped scan is reported in the summary). Windows
are capped at 366 days.

---

## 6. Security

* Tenant: the database is selected from the `ClientGUID` header before any controller runs.
  No request DTO has a client/tenant/connection property (`ReportsSecurityTests`).
* Auth: both controllers derive from `BaseController` (`[Authorize]`); no `[AllowAnonymous]`
  (tested). The `ClientId` claim is read only for Shadow AI tool approvals, which are keyed by it.
* Injection: sort keys, view keys, filter keys and static filter values are validated against
  the catalog before any query; free-text values only ever become parameters (`ILIKE @p`,
  `lower(col) IN (@list)`), with `%`/`_`/`\` escaped.
* Data exposure: rows carry grid-visible fields only — no `FileContent`, `SensitiveContent`,
  prompt text or override justification beyond what the design shows.

---

## 7. Excel export

* Library: **ClosedXML 0.104.2** (MIT), already in the solution. No new Excel dependency.
* Scopes: `grid` (Data sheet), `chart` (Summary + Chart Data), `full` (all three).
* Data sheet: context line (report, view, window, generated UTC, row count / cap notice),
  header row with **S.No**, one column per grid column plus one per secondary line, a numeric
  "(bytes)" twin for size columns; Excel **Table** with banded rows and **AutoFilter**; frozen
  header; typed cells (dates `yyyy-mm-dd hh:mm:ss`, integers `#,##0`, decimals, percentages);
  conditional-format colouring of tag values (Blocked / Allowed / Pending …); widths fitted then
  capped at 60 with wrap; landscape, fit to one page wide, header rows repeated, page x / y footer.
* Summary sheet: title, window, applied filters (or "None"), KPI table, chart table with
  **in-cell data bars** (conditional formatting). ClosedXML cannot author native chart objects;
  the chart therefore ships as live data + data bars rather than a picture, and the Chart Data
  sheet holds the full series for re-plotting.
* Cap: `Reports:MaxExportRows` (default 50,000; hard ceiling 1,048,573) with `Reports:ExportBatchSize`
  (default 2,000) streaming; the caller may lower the cap with `maxRows`. A capped file says
  "first N of M rows" in its header line, and the response carries `X-Report-Row-Count`,
  `X-Report-Total-Rows`, `X-Report-Truncated`. A background job (as `ExportTransferLogs` has)
  was not added: report windows are bounded to a year and the cap keeps a file to seconds; if a
  tenant needs full-history report files, the `ExportJob` flow can be reused unchanged.

---

## 8. Testing

`SFM.Platform.Storage.Tests/Reports/` (xUnit):

| File | Covers |
|---|---|
| `ReportCatalogTests` | ten reports, categories, every default/column sort whitelisted, unique keys, snapshot/no-chart flags, static value canonicalisation |
| `ReportQueryNormalizerTests` | default window, date-only end, timestamp end, UTC kinds, inverted / over-long ranges, paging bounds, sort whitelist, unknown filters, static value validation, trimming/dedupe, pivot/mode selection, aggregated errors, single-vs-array JSON binding, `WithPage`, user/endpoint accepted by id and by name |
| `ReportMappingTests` | verdict labels, size bands, agent status, outcomes, Shadow AI states, encryption posture, policy hierarchy + default fallback, movement classification, decision mapping, inventory/investigation helpers, export file names |
| `ReportsServiceAndExportTests` | same filter for summary/chart/grid/export, KPI values, server-side paging, filtering, search + window, zero-filled trend, blocked report, empty results, chart refusal, 404/400, filter options, file tracing chains, **the pivot → event drill-down funnel (id and name, user and endpoint, options carry ids)**; Excel: sheets, header, S.No, row counts, Table + AutoFilter, frozen rows, typed cells, size formatting, summary/chart sheets, cap + "first N of M", server cap, empty file, format/report rejection, single load for derived reports |
| `ReportsSecurityTests` | `[Authorize]` via `BaseController`, no `[AllowAnonymous]`, route prefixes, Excel only in the export controller, no tenant field on requests |

Database-dependent behaviour (EF translation of `ILike`, `JsonExistAny`, grouped subqueries)
is not covered by unit tests — the test project has no PostgreSQL provider — and is verified
against a tenant database on QA. The SQL shapes reuse patterns already in production
(`GetTransferLogs`, `GetIncidentSummaryCountsAsync`, `AgentRunRepository.GetRecentAsync`).

---

## 9. Configuration

Optional, all with defaults:

```json
"Reports": {
  "MaxExportRows": 50000,
  "ExportBatchSize": 2000
}
```

`TrayAppSettings:MonthlySelfOverrideLimit` (existing) drives the "n of m" quota column.

---

## 10. Status reports (Summary category) — added 2026-10-20

**UX reference:** "DLP Reports v2" design (Weekly Status Report with Chart View / Table
View, compared with the previous period). Three catalog entries: `daily`, `weekly`,
`monthly` under a new **Summary** category.

### Shape

A status report covers one whole period (UTC day / Monday-start week / calendar month)
and compares it with the period immediately before. `StatusReportPeriods` holds the
window arithmetic: dates absent = the last **complete** period (comparing a
half-elapsed period against a whole one reads as a drop that is not there); dates
present = the period containing the requested end. `currentLabel`/`previousLabel`
are relative for the latest complete period ("This week") and dated for any past
one ("Sep 14 – 20", "Sep 16", "September 2026") — added 2026-10-20 so the UI stops
building its own headers.

One build per request (`StatusReportHandlerBase.GetStatusAsync`) produces a
`StatusReportResponse`; the generic grid (the design's Table View), the KPI summary
and the Excel export are flattened from it, so the endpoints cannot disagree. The
Chart View's composite payload is served by **`POST reports/{id}/status`**
(`IStatusReportHandler`, dispatched by `ReportsService.GetStatusAsync`; 400 for a
non-status report). The catalog marks the three with **`layout: "status"`**
(`ReportLayouts`; everything else is `"grid"`) — the client's screen switch, added
2026-10-20 at the UI team's request so nothing hardcodes ids or the category name.
`HasChart` is true: the generic chart serves the one block its shapes can carry, the
daily events/blocked trend, which also fills the Excel `full` export's chart sheet.

### Sections → sources (current and previous window, same queries twice)

| Section | Source |
|---|---|
| Overview tiles | `GetUploadSummaryAsync` ×2, policy hits = Σ `GetPolicyActivityAsync`, risk bands, shadow tool counts, investigation counts, override totals, device rollup |
| Data activity (daily bars) | `GetUploadDailyTrendAsync`, zero-filled over the period |
| Activity / blocked by channel | **new** `GetUploadCountsByTransferTypeAsync` — grouped on `TransferTypeName` as stored (client data; no invented taxonomy); a channel active only last period still appears with zero |
| Policy enforcement (top 5 by hits) | `GetPolicyActivityAsync(window, coveredOnly: true)` ×2 — rows with a real verdict (`PolicyAction` ≠ `UnCovered`): the crunch stamps a PolicyName on every row, so counting by name alone equals data events. **Not** `IsAppliedToPolicy` — that flag marks logs an admin turned into policy rules via Log Review (a first attempt used it and QA's hits collapsed to 2). On a catch-all-policy tenant, hits = data events legitimately. Allowed = Total − Blocks − Warns |
| User risk bands + biggest increases | `GetRiskWindowsAsync` ×2, bands via `RiskBands.FromBaseline`; movers = users scored in **both** periods, top 5 by peak-score increase, department from the user directory |
| Shadow AI detections | `GetLatestFindingPerToolAsync` ×2 folded by `ShadowAiReportHandler.CanonicalToolKey` — noise names (blank / purely numeric, e.g. a QA finding literally named "10") dropped, variants ("DeepSeek" / "DeepSeek AI") merged (events sum, users = largest variant, any variant's standing decision wins) + `GetFirstDetectedByToolAsync` over all variants. One row per tool seen in **either** period — a tool that went quiet keeps its row with 0 current events and its `previousEvents`, matching the channel rows; the overview tile counts this period's detections only |
| Monthly trends (6 months) | **new** `GetMonthlyActivityCountsAsync` (events / blocked / policy hits / investigations per calendar month) + `GetRiskWindowsAsync` once per month for high-risk users |
| Monthly by-department | **new** `GetUploadCountsByDepartmentAsync` ×2 (crunch ⟕ sfm."User", grouped on DepartmentName) + the period's risk rows averaged per department via the directory |
| Incidents & investigations | **new** `GetInvestigationSeverityCountsAsync` ×2 — opened by severity, plus how many are closed **now** |
| Override & approval requests | `GetOverrideSummaryAsync` ×2 + **new** `GetOverrideDecisionHoursAsync` ×2 (median in memory; a period holds at most a few hundred decisions) |
| File inventory / disk encryption | snapshots (`CountDiscoveryFilesByClassificationAsync`, `GetDiscoveryVolumesAsync` with the Disk Encryption report's posture rule); `snapshot: true`, no previous column |

### Deviations (same omit-rather-than-invent rule as §3)

| Design element | Decision |
|---|---|
| Monthly "6-month trends" / "By department" | Served (added 2026-10-20 after UI review): `trends` + `departments` on the monthly report only; empty arrays on daily/weekly. |
| Web requests blocked / web filtering categories | No web-filtering event data exists — section omitted. |
| Tenant violations / tenant restrictions by app | No tenant-restriction event data exists — omitted. |
| Mean time to close | Crunch rows carry no close timestamp — omitted; the section note says so, and "Still open (of opened)" is served instead. |
| Channel taxonomy (Upload, Copy/Paste, Prompting …) | `TransferTypeName` is per-tenant master data; the stored names are served, not mapped onto the design's labels. |
| File inventory / encryption week-over-week change | Discovery is a now-snapshot; previous-period figures do not exist. |
| "Unencrypted devices" | Devices not fully protected — Partial and Suspended included — because a half-encrypted laptop is exposed. |

### Tests

`StatusReportTests`: period arithmetic pinned to the design's own dates (generated
Mon 2026-10-12 → current week Oct 5–11 vs Sep 28–Oct 4), the comparison maths and
change formatting, grid flattening + `section` filter, KPI tiles, and the 400 for a
non-status report. `ReportCatalogTests` updated for thirteen reports / four categories.
