# Data Discovery & Classification — Implementation Plan

**Status:** Draft for review · **Owner:** DLP API team · **Date:** 2026-09-10
**UX reference:** "Data Discovery & Classification" designs (Overview + By User tabs, Run Discovery Scan / Re-scan modals)

---

## 1. What this feature is

Today the DLP platform classifies **data in motion**: the agent inspects a file at the moment a
user transfers it (upload, copy to device, share), and the result lands in
`dlp.LandingLogs` → `crunch.LandingLogsCrunched`.

Data Discovery adds **data at rest**: on a schedule or on demand, the agent walks the disks of
each managed endpoint, classifies every file's content with the **same classification engine
and taxonomy** (PHI / PII / PFI / PCI / Code / Credentials), and reports an **inventory** of
what sensitive data exists, where it sits, who holds it, and whether the disk it sits on is
encrypted.

The two datasets stay **separate**:

| | Transfer DLP (existing) | Discovery (new) |
|---|---|---|
| Tables | `dlp.LandingLogs`, `crunch.LandingLogsCrunched` | new **`discovery`** schema |
| Nature | Event stream, append-only; a file can appear many times | Inventory, upserted; one row per file currently on disk |
| Question | What did users try to move, and what did policy do? | What sensitive data exists on disks right now? |
| Growth | Grows with activity | ≈ stable at the number of files on disks |

Shared by reference, not by table: the classification taxonomy (same DlpType names, same
agent engine), Endpoint / User / tenant keys (so reports can join, e.g. *"user holds 2,418
sensitive files and had 72 violations"*), and the masking rule (no raw matched content ever
leaves the agent).

---

## 2. Concepts and definitions

### 2.1 Scan types

| | Full scan | Incremental |
|---|---|---|
| Behaviour | Enumerates **every file** on the selected volumes and classifies each from scratch | Processes **only files changed since the last completed scan** — new, modified, deleted |
| When | First scan of a volume; after classification-rule changes; integrity rebuild | Routine weekly / manual runs |
| Endpoint cost | Heavy (the modal's "~42 min average") | Minutes; typically <5 % of files change per week |

**Change detection for incremental — two mechanisms, chosen per volume:**

1. **NTFS USN Change Journal** (primary, for fixed NTFS volumes C:/D:).
   The agent stores the last-read USN cursor per volume after every completed scan; the next
   incremental reads only journal records after that cursor. Reliable for deletes and renames.
   Fallback to full scan when: journal wrapped (cursor too old), journal disabled, or volume
   is not NTFS.
2. **Inventory diff** (fallback, for external / FAT32 / mapped volumes).
   The agent keeps a local snapshot (`path + size + lastModified`) per volume and compares on
   the next run. Slower than USN, works anywhere.

**Agent-enforced rules:** the first scan of a volume is always full; an incremental request is
silently upgraded to full when no valid cursor/snapshot exists. The scan that ran is reported
back (`executedType`), so the UI can show what actually happened.

### 2.2 Scan scopes (Run Discovery Scan modal)

| Scope | Resolution |
|---|---|
| All managed endpoints | every active endpoint with a DLP agent |
| High-risk users only | endpoints of users from the existing daily-risk-snapshot query (same rule as the High-Risk Users widget: peak `weighted_risk_score_v2_d ≥ 65` in the window) |
| Never scanned (N) | endpoints with no row in `discovery.EndpointScan` with status `Completed` |

### 2.3 Disk encryption states (agent reads BitLocker via WMI `Win32_EncryptableVolume`)

| State | Meaning | UX example |
|---|---|---|
| `Encrypted` | all selected volumes protected; method captured (e.g. XTS-AES 256) | "Encrypted · XTS-AES 256" |
| `Partial` | some volumes unprotected | "Partial · 2 of 3 · E: open" |
| `Suspended` | protection paused; `suspendedSince` captured | "Suspended · Paused 9 days" |
| `Unencrypted` | no volume encryption | "Unencrypted" |

### 2.4 Endpoint scan coverage states

| State | Rule |
|---|---|
| Fully scanned | latest `EndpointScan` completed all requested volumes |
| Partial — interrupted | latest scan sent progress but no completion within the heartbeat timeout (laptop shut down mid-scan), or completed with volumes skipped |
| Never scanned | no completed `EndpointScan` ever |

---

## 3. Architecture and flow

```
 SFM.DLP.LogTrigger TimerTrigger (15-min tick,          Admin clicks "Run Discovery
 EndpointStatusMonitor pattern) → per tenant:            Scan" / "Re-scan"
 POST /discovery/run-scheduled (ClientGUID) —
 API checks tenant ScanSchedule, creates job when due
        │                                                    │
        ▼                                                    ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │ SFM.DLP.API  POST /discovery/scans                                   │
 │   creates ScanJob + one EndpointScan row (Queued) per in-scope agent │
 └──────────────────────────────────────────────────────────────────────┘
        │  (agents are behind NAT — no push; they poll)
        ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │ Agent polls SFM.DLP.ExeApi  GET V1/GetDiscoveryScanJob               │
 │   (same pattern as the existing V1/GetEndPointCommand channel)       │
 └──────────────────────────────────────────────────────────────────────┘
        │
        ▼  per selected volume
   full walk  ──or──  USN-delta / inventory diff
        │
        ▼  classify locally (existing DLP content engine — content never leaves the laptop)
        │
        ▼  batched uploads to SFM.DLP.ExeApi
   POST V1/DiscoveryScanProgress    (heartbeat + running counters)
   POST V1/DiscoveryFilesBatch      (≤500 file results per call, upsert)
   POST V1/DiscoveryScanComplete    (final counters, per-volume USN cursors, encryption info)
        │
        ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │ On completion the API computes discovery.ScanSummary rows.           │
 │ EVERY dashboard widget reads summaries — never COUNT(*) on the       │
 │ file inventory (lesson from GetTransferLogs / GetIncidentSummary).   │
 └──────────────────────────────────────────────────────────────────────┘
        │
        ▼
 Dashboard endpoints on SFM.DLP.API (Overview tiles, charts, By-User grid,
 per-endpoint file inventory with paging/filters)
```

Tenant routing: all new routes resolve the tenant DLP database from the `ClientGUID`
header/claim exactly like the existing pipeline (`ConnectionStringResolver`).

---

## 4. Schema — new `discovery` schema in the tenant DLP database

> DDL below is the review sketch; final DDL lands as a migration script.
> Naming follows existing conventions (`PascalCase` columns, `IsActive`-style flags).

### 4.1 `discovery."ScanSchedule"` — one row per tenant

```sql
ScanScheduleId   smallserial PK
Cadence          text        -- 'Weekly' | 'Daily' | 'Monthly'
RunAtLocal       time        -- 02:00
TimeZone         text        -- 'IST'
DefaultScanType  text        -- 'Incremental'
DefaultVolumes   text[]      -- {'C','D','E','Mapped'}
IsActive         bool
DateAdded / DateUpdated / UpdatedBy
```

### 4.2 `discovery."ScanJob"` — one row per triggered run

```sql
ScanJobId        bigserial PK
Trigger          text        -- 'Scheduled' | 'Manual' | 'Rescan'
Scope            text        -- 'All' | 'HighRisk' | 'NeverScanned' | 'SingleEndpoint'
TargetEndpointId int NULL    -- set for Rescan
ScanType         text        -- 'Full' | 'Incremental'  (requested)
Volumes          text[]
RequestedBy      int         -- UserId
CreatedAt / StartedAt / CompletedAt
Status           text        -- 'Queued' | 'Running' | 'Completed' | 'CompletedWithErrors' | 'Cancelled'
```

### 4.3 `discovery."EndpointScan"` — one row per endpoint per job

```sql
EndpointScanId   bigserial PK
ScanJobId        bigint  FK → ScanJob
EndpointId       int     FK → sfm.Endpoint
Status           text    -- 'Queued' | 'Running' | 'Completed' | 'Interrupted' | 'Failed' | 'Superseded'
                         -- Interrupted: heartbeat silent >15 min; RESUMABLE — the agent's next poll
                         --   returns the same scan flipped back to Running (catch-up, §8.6).
                         -- Superseded: still Queued or Interrupted when the next scheduled job
                         --   covered the endpoint (§8.6).
ExecutedType     text    -- 'Full' | 'Incremental' (what actually ran; may differ from requested)
FilesScanned     int
SensitiveFound   int
StartedAt / LastHeartbeatAt / CompletedAt
DurationSeconds  int     -- feeds the "~42 min average"
ErrorDetail      text NULL
-- INDEX (EndpointId, Status), INDEX (ScanJobId)
```

`Interrupted` is derived by a periodic sweep: `Running` with `LastHeartbeatAt` older than the
heartbeat timeout (proposed: 15 min).

### 4.4 `discovery."Volume"` — per endpoint volume (the incremental state lives here)

```sql
VolumeId          bigserial PK
EndpointId        int FK
DriveLetter       text     -- 'C', 'D', 'E' or UNC root for mapped
Kind              text     -- 'Local' | 'External' | 'Mapped'
FileSystem        text     -- 'NTFS' | 'FAT32' | ...
SizeGb            numeric(10,2)
EncryptionStatus  text     -- 'Encrypted' | 'Partial' | 'Suspended' | 'Unencrypted'
EncryptionMethod  text NULL          -- 'XTS-AES 256'
SuspendedSince    timestamptz NULL
UsnCursor         numeric NULL       -- last-read USN journal position
SnapshotVersion   bigint NULL        -- inventory-diff fallback version
FilesScanned      int                -- last completed scan
SensitiveCount    int
LastScanAt        timestamptz
IsActive          bool               -- volume disappeared → false
-- UNIQUE (EndpointId, DriveLetter)
```

### 4.5 `discovery."File"` — the inventory (the scale-sensitive table)

```sql
FileId          bigserial PK
EndpointId      int FK
VolumeId        bigint FK
Directory       text          -- 'D:\Finance\Payroll\2026'
FileName        text          -- 'Payroll_Master_2026.xlsx'
Extension       text          -- 'xlsx' (lowercased)
SizeBytes       bigint
LastModified    timestamptz   -- feeds Age of Sensitive Data buckets
Classifications jsonb         -- ["PFI","PII"]; [] = scanned clean; NULL = unclassified/unreadable
IsSensitive     bool          -- Classifications non-empty
Confidence      numeric(5,2) NULL
FirstSeenAt     timestamptz   -- set once — feeds "Newly Discovered Sensitive Files"
LastSeenAt      timestamptz   -- stamped every scan that sees the file
IsDeleted       bool          -- file gone at last scan (kept for trend, purged by retention)
-- UNIQUE (EndpointId, VolumeId, Directory, FileName)
-- INDEX (EndpointId, IsSensitive) INCLUDE (Extension, SizeBytes, LastModified)
-- GIN INDEX on Classifications
-- PARTITION BY HASH (EndpointId)  — see 4.7
```

**No file content, no matched keywords, no samples are stored** — only metadata plus the
classification verdict. Matching detail stays on the agent (consistent with the transfer-log
masking rules).

**Row scope (decided, §8.1):** only **sensitive and unclassified** files are stored as rows
(~2–3 M/tenant). Clean files are counted in `Volume.FilesScanned` / `ScanSummary` but never
stored row-by-row — the agent reports them in batch counters, not as file entries. The UX
"All files" tab therefore shows totals, and its grid lists sensitive + unclassified rows.

### 4.6 `discovery."ScanSummary"` — precomputed aggregates (what the dashboard reads)

One row per completed `EndpointScan`, plus a tenant-level rollup row per `ScanJob`:

```sql
ScanSummaryId     bigserial PK
ScanJobId         bigint FK
EndpointId        int NULL          -- NULL = tenant rollup row
FilesScanned      int
SensitiveFiles    int
UnclassifiedFiles int
VolumeScannedBytes  bigint
SensitiveVolumeBytes bigint
ByClass           jsonb   -- {"PHI":{"files":32780,"deltaPct":6}, ...}
ByClassByVolumeKind jsonb -- the "Where Each Class Lives" matrix
ByExtension       jsonb   -- {".xlsx":{"scannedTb":74,"sensitiveTb":12.8}, ...}
AgeBuckets        jsonb   -- {"0-6m":38420,"6-12m":26110,"1-3y":23742,"3y+":8140}
HotspotFolders    jsonb   -- top N paths: [{"path":"C:\\Users\\*\\Downloads","class":"PHI","files":12480,"endpoints":618}]
NewSensitiveByWeek jsonb  -- stacked weekly counts for the trend chart
CreatedAt         timestamptz
```

**Rule (from hard experience with `GetTransferLogs` / `GetIncidentSummary`): every Overview
widget is served from `ScanSummary` only. `discovery."File"` is touched by exactly one
endpoint — the per-endpoint Discovered Files grid — always filtered by `EndpointId` and
paged in SQL.**

### 4.7 Scale & retention

- With the sensitive-only decision (§8.1), `discovery."File"` stays **unpartitioned**
  (~2–3 M rows/tenant); the hash-partitioning design is parked in case the scope ever widens.
- Batch upserts via `INSERT ... ON CONFLICT (EndpointId,VolumeId,Directory,FileName) DO UPDATE`.
- Retention: `IsDeleted` rows purged after 90 days (configurable); `ScanJob`/`EndpointScan`
  history kept 12 months; `ScanSummary` kept indefinitely (it is small and feeds trends).

---

## 5. API surface

> ⚠ **Every route below needs an APIM operation registered before it is reachable in QA** —
> a deployed route without one returns the gateway's 404 (see the GetAppliedPolicies incident).

### 5.1 Agent-facing — SFM.DLP.ExeApi (Azure Functions, `V1/…`, same auth as existing agent routes)

| Route | Purpose |
|---|---|
| `GET  V1/GetDiscoveryScanJob?serialNumber=&endpointUserGuid=` | Agent poll. Returns the queued/running `EndpointScan` for this device (scan type, fixed-letter `volumes`, `includeMapped`/`includeExternal`, batch size, heartbeat interval) or 204. Mirrors `V1/GetEndPointCommand`. |
| `POST V1/DiscoveryScanProgress` | Heartbeat: `{endpointScanId, filesScanned, sensitiveFound, currentVolume}`. Updates `LastHeartbeatAt` + counters. |
| `POST V1/DiscoveryFilesBatch` | ≤500 results per call: `{endpointScanId, files:[{volume, directory, fileName, ext, sizeBytes, lastModified, classifications[], confidence, changeKind:'New'|'Modified'|'Deleted'}]}`. Upserts inventory; `Deleted` sets `IsDeleted`. Idempotent per batch (agent sends `batchSeq`, API ignores replays). |
| `POST V1/DiscoveryScanComplete` | Final counters + per-volume state: `{endpointScanId, executedType, volumes:[{letter, kind, fileSystem, sizeGb, encryptionStatus, encryptionMethod, suspendedSince, usnCursor, filesScanned, sensitiveCount}]}`. Marks Completed, upserts `Volume`, triggers `ScanSummary` computation. |

### 5.2 Dashboard — SFM.DLP.API (JWT, `BaseController`)

| Route | Serves (UX element) |
|---|---|
| `POST /discovery/scans` | Run Discovery Scan modal → creates `ScanJob` (+ per-endpoint rows). Body: `{scope, scanType, volumes[], includeMapped?, includeExternal?, targetEndpointId?}`. `volumes[]` is fixed drive letters; mapped shares and removable media are the two flags, because their letters are not knowable until the scan runs |
| `GET  /discovery/scans/latest` | Header strip: last scanned, next scheduled, cadence; job in progress if any |
| `POST /discovery/overview` | All Overview tiles + charts in ONE call, from `ScanSummary`: filesScanned, sensitiveFiles, unencryptedDisks x/y, filesOnUnencryptedDisks, endpointsScanned x/y, classificationBreakdown (+delta %), classByVolumeKind matrix, newSensitiveByWeek, sensitiveVolumeByFileType, hotspotFolders, topUsers, ageBuckets, coverage {fully, partial, never} |
| `POST /discovery/users` | By User grid (Users view): user, endpoint, disks, encryption, files, sensitive, class chips, risk (joins existing risk bands), lastScan. `QueryParameters` paging/sort/multi-value filters — same contract as `GetTransferLogs`/`GetDlpUsers` |
| `POST /discovery/endpoints` | Same grid, Endpoints view (adds volumes GB scanned, agent status/version) |
| `POST /discovery/endpoint-detail` | Right-hand panel: tiles (filesScanned, sensitive, unclassified, volumeOnDisk, lastScan) + per-volume cards |
| `POST /discovery/files` | Discovered Files grid: `{endpointId, tab:'Sensitive'|'All'|'Unclassified', search, page, pageSize, sortBy}` → pages `discovery."File"` in SQL |
| `GET  /discovery/schedule` / `POST /discovery/schedule` | Read/update the scan schedule |
| `POST /discovery/run-scheduled` | Called by the LogTrigger timer per tenant. Reads `ScanSchedule` in the tenant TZ; creates the due `ScanJob` or replies "not due". Idempotent — at most one job per schedule slot. |

### 5.3 UI element → field mapping (integration handoff)

| UX element | Endpoint | Field |
|---|---|---|
| Files Scanned 4.28M | `/discovery/overview` | `filesScanned` |
| Sensitive Files 96,412 | 〃 | `sensitiveFiles` |
| Unencrypted Disks 192/2,240 | 〃 | `unencryptedDisks.count` / `.total` |
| Files on Unencrypted Disks | 〃 | `filesOnUnencryptedDisks` |
| Endpoints Scanned 782/800 | 〃 | `endpointsScanned.count` / `.total` |
| Classification Breakdown (+6 %) | 〃 | `classification[] {label, files, sharePct, deltaPct}` |
| Where Each Class Lives | 〃 | `classByVolumeKind` matrix |
| Newly Discovered (weekly stack) | 〃 | `newSensitiveByWeek[]` |
| Sensitive Volume by File Type | 〃 | `byExtension[]` |
| Hotspot Folders | 〃 | `hotspotFolders[]` |
| Users Holding Most Sensitive Data | 〃 | `topUsers[]` |
| Age of Sensitive Data | 〃 | `ageBuckets` |
| Endpoint Scan Coverage | 〃 | `coverage {fully, partialInterrupted, neverScanned}` |
| By User / Endpoints grids | `/discovery/users`, `/discovery/endpoints` | row fields per §5.2 |
| Volume cards + Discovered Files | `/discovery/endpoint-detail`, `/discovery/files` | per §5.2 |

---

## 6. Agent responsibilities (for the agent team)

1. Poll `GetDiscoveryScanJob` on the existing command cadence; honour one active scan at a time.
2. Resolve volumes: the fixed letters in `volumes`, plus — only when the job's `includeExternal` / `includeMapped` flags are set — the removable/external volumes and mapped network drives found on the machine, whatever letters they have. Report kind (`Local`/`External`/`Mapped`) + file system.
3. Full walk or USN-delta per §2.1; maintain USN cursor / snapshot per volume locally.
4. Classify with the existing content engine; produce class list + confidence per file.
   **Never** upload content, matched keywords, or samples.
5. Batch, throttle (CPU/IO caps — scan must not degrade the user's machine; proposed:
   low-priority IO, configurable max files/sec), heartbeat every ≤5 min.
6. Read BitLocker status per volume via WMI at scan start and report it on completion.
7. Resume: if the machine reboots mid-scan, resume from the last acked batch (agent keeps a
   local journal of acked `batchSeq`).

---

## 7. Security & privacy

- Metadata only leaves the endpoint (path, name, size, dates, class labels, confidence).
- File *names/paths* are themselves sensitive — the new tables live in the tenant DLP DB behind
  the same tenant routing; the dashboard routes sit behind the existing JWT scheme.
- Agent routes use the existing agent auth (and inherit the JWT/HMAC migration planned in
  AGENT-SECURITY-PLAN.md when that lands — no new shared-secret surface).
- Discovery detail endpoints (file paths per user) should be treated like
  `GetSensitiveContentById`: an authorised drill-down, logged via `ApiLog` like other actions.

---

## 8. Decisions

| # | Decision | Outcome |
|---|---|---|
| 8.1 | `discovery."File"` row scope | **DECIDED (b): sensitive + unclassified rows only** (~2–3 M/tenant). Clean files are counted (per volume / per scan) but not stored row-by-row; the "All files" tab shows counts, its grid lists sensitive + unclassified. Partitioning deferred — unpartitioned is fine at this volume. |
| 8.2 | Mapped-drive dedup | count per endpoint in v1; label the matrix column "Mapped" honestly |
| 8.3 | Heartbeat timeout for `Interrupted` | 15 min |
| 8.4 | Schedule execution | **DECIDED: TimerTrigger in the existing `SFM.DLP.LogTrigger` Functions app** — the per-tenant fan-out already exists there (`ApiClient` + `ClientGuidProvider`), one scheduling mechanism across features, and API deploys/restarts don't affect schedules. Pattern copied from `EndpointStatusMonitor`: the timer fires every 15 min and calls `POST /discovery/run-scheduled` once per active tenant (ClientGUID header); the API reads that tenant's `ScanSchedule` (RunAtLocal + TimeZone), and either returns "not due" or creates the ScanJob — idempotent, at most one job per schedule slot, so the frequent tick is safe and catches each tenant's local 02:00 without per-TZ cron lines. |
| 8.5 | Retention windows | §4.7 defaults |
| 8.6 | Offline endpoints at schedule time | **DECIDED: catch up until next run.** An `EndpointScan` stays claimable (`Queued`) until the next scheduled `ScanJob` is created, which supersedes it (`Status='Superseded'`). A laptop off at 02:00 picks the job up whenever it next polls — coverage converges to 100 % without double scans. |
| 8.7 | File coverage for content classification | **DECIDED: identical to the transfer engine** — same file-type list, same archive (zip) handling, same max-file-size guard (`FileProcessingGuard`). One taxonomy, one behaviour: a file classed PHI on transfer is classed PHI at rest. Everything outside that coverage is counted clean without content reading. |
| 8.8 | Scan throttling on active machines | **DECIDED: low-priority background.** Scans run any time with low-priority I/O and a configurable CPU/files-per-second cap — no idle-only gating. |
| 8.9 | Who may trigger manual scans | **DECIDED: admin roles only** — the same roles that manage policies. `POST /discovery/scans` and Re-scan enforce the role server-side (not just hidden buttons). |

---

## 9. Delivery phases

| Phase | Contents | Exit criteria |
|---|---|---|
| **P1 — Plumbing** | `discovery` schema migration; ExeApi routes (poll, progress, batch, complete); manual `POST /discovery/scans`; APIM operations | An agent build can complete a full scan of one endpoint; rows land in `File`/`Volume`; `EndpointScan` reaches Completed |
| **P2 — Dashboard** | `ScanSummary` computation; `/discovery/overview`; By-User/Endpoints grids; files grid; schedule CRUD + `DiscoveryScanScheduler` TimerTrigger in SFM.DLP.LogTrigger | Overview renders entirely from summaries; grid filters/paging pass the same QA patterns as GetTransferLogs; scheduled job fires at each tenant's local time |
| **P3 — Incremental + coverage** | USN cursor path, inventory-diff fallback, executedType, Interrupted sweep, Re-scan modal | Incremental scan on a changed volume touches only changed files; coverage tile shows Fully/Partial/Never correctly |
| **P4 — Hardening** | retention purge, partitioning (if 8.1 = all files), load test at 800 endpoints × weekly, docs | dashboard p95 < 1 s on summary endpoints at target volume |

### QA acceptance highlights

- Full scan then immediate incremental with no changes → incremental reports 0 new/modified.
- Modify 3 files, delete 1 → incremental upserts exactly 3, flags 1 `IsDeleted`; tiles move accordingly.
- Kill agent mid-scan → endpoint shows **Partial — interrupted** after the heartbeat timeout; re-scan resumes.
- BitLocker suspended on one volume → endpoint shows `Partial`/`Suspended` per volume as designed.
- Overview totals reconcile: Σ per-class files ≥ `sensitiveFiles` (multi-class files count once per class — same documented rule as `data-classification`).
- No dashboard endpoint issues `COUNT(*)`/scan over `discovery."File"` (verified via query logging).

---

## 10. Explicitly out of scope (v1)

- Remediation actions (delete/quarantine/encrypt from the dashboard)
- Content preview or matched-keyword display for discovered files
- Cloud-storage discovery (OneDrive/SharePoint crawling) — endpoint disks only
- Cross-tenant/organisation rollups
