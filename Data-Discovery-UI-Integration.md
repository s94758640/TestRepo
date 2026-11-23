# Data Discovery & Classification — UI Integration Guide

**Audience:** UI / integration team · **Screens:** Overview tab, By User tab (Users + Endpoints views), Run Discovery Scan modal, Re-scan modal
**Status:** endpoints code-complete on branch `feature/data-discovery-classification`; QA availability requires deploy + APIM registration
**Backend contacts:** DLP API team · Design reference: DATA-DISCOVERY-IMPLEMENTATION-PLAN.md

---

## 1. Basics

- **Base URL:** the DLP API gateway (same as the dashboard APIs), e.g. `https://qa-email.azure-api.net/qadlp`
- **Auth:** every request needs the normal `Authorization: Bearer <JWT>` **and** the `ClientGUID` header (tenant routing) — identical to the existing dashboard calls.
- **Casing:** responses are camelCase JSON, wrapped in the standard envelope:

```json
{ "success": true, "message": null, "data": { ... }, "errors": null }
```

- **Admin-only actions:** `POST /discovery/scans` and `POST /discovery/schedule` return **403** for non-admin roles. The UI should hide/disable those buttons for non-admins, but the server enforces it regardless.
- Until the EXE agent integrates, all data comes from the mock seeder — every widget renders, values are synthetic.

---

## 2. Endpoint map (what each screen element calls)

| Screen element | Method + route |
|---|---|
| Header strip (Scanned … / Next scan in … / Weekly · 02:00 IST) | `GET /discovery/scans/latest` |
| Run Discovery Scan modal → Start scan | `POST /discovery/scans` |
| Re-scan modal (single endpoint) → Start scan | `POST /discovery/scans` (scope `SingleEndpoint`) |
| Cancel ✕ on a row · Cancel scan button | `POST /discovery/scans/cancel` |
| Stop/Start DDC scanner option | `GET /discovery/scanner` · `POST /discovery/scanner` |
| "Found in" scan picker | `POST /discovery/scan-history` |
| ALL Overview tiles + charts | `POST /discovery/overview` (one call) |
| By User tab — Users view grid | `POST /discovery/users` |
| By User tab — Endpoints view grid | `POST /discovery/endpoints` (same rows/shape) |
| Right-hand detail panel (tiles + volume cards) | `POST /discovery/endpoint-detail` |
| Discovered Files grid (Sensitive / All / Unclassified) | `POST /discovery/files` |
| "Found in" scan dropdown on the files grid | `POST /discovery/scan-history` |
| Schedule settings read / save | `GET /discovery/schedule` · `POST /discovery/schedule` |

---

## 3. Header strip — `GET /discovery/scans/latest`

```json
{
  "lastCompletedAt": "2026-09-09T21:10:44Z",
  "nextScheduledAt": "2026-09-16T20:30:00Z",
  "cadence": "Weekly",
  "runAtLocal": "02:00",
  "timeZone": "Asia/Kolkata",
  "scanInProgress": false,
  "runningScanJobId": null,
  "averageDurationMinutes": 42,
  "runningMachineCount": 0,
  "queuedMachineCount": 0,
  "scannerEnabled": true
}
```

- "Scanned 1 day ago" ← `lastCompletedAt` · "Next scan in 6 days" ← `nextScheduledAt` · "Weekly · 02:00 IST" ← `cadence` + `runAtLocal` + `timeZone`.
- `averageDurationMinutes` is the real per-endpoint average — use it for the "~42 min average" line in both modals.
- While a scan runs, `scanInProgress` is true; poll this endpoint to refresh the strip.
- `scannerEnabled` is the tenant's Stop/Start DDC scanner option (§7.5) — when false, show the scanner-stopped state and disable the Run Scan buttons (the server refuses them with 400 anyway).

---

## 4. Run Discovery Scan / Re-scan — `POST /discovery/scans`

Request (Run Discovery Scan modal):

```json
{
  "scope": "All",
  "scanType": "Full",
  "volumes": ["C", "D", "E"],
  "includeMapped": true,
  "includeExternal": true
}
```

- `scope`: `All` | `HighRisk` | `NeverScanned` | `SingleEndpoint`
  (maps to the modal options: All managed endpoints / High-risk users only / Never scanned)
- `scanType`: `Full` | `Incremental`
- `volumes`: the **fixed drive letters** ticked in the modal — any of `C`, `D`, `E`; omit/empty = all three.
- `includeMapped` (bool): the modal's **Mapped drives** tick. Mapped network drives are scanned whatever letters they have on each machine.
- `includeExternal` (bool): the modal's **External drives** tick — USB sticks and portable disks attached when the scan runs.

> **Why these two are not letters.** A mapped share or a USB stick has no letter anyone can name when the job is queued: it depends on who is signed in and what is plugged in at scan time. So the modal must send them as flags. Sending `volumes: ["C"]` alone means **C and nothing else** — that is why the **External** and **Mapped** columns on the grids read 0.

- Both flags are optional. When omitted the server reads them from the old pseudo-volume tokens in `volumes` (`"Mapped"`, `"External"`), so an older console build keeps working unchanged — but new UI should send the booleans and keep `volumes` to real letters.
- Re-scan modal sends: `{ "scope": "SingleEndpoint", "targetEndpointId": 4482, "scanType": "Full", "volumes": ["C","D","E"], "includeMapped": false, "includeExternal": true }`
- The agent receives both flags on its next `GET V1/GetDiscoveryScanJob` poll and reports each volume it found back with `kind: "Local" | "Mapped" | "External"` — which is what fills those two columns.

Response:

```json
{ "scanJobId": 12, "endpointsQueued": 27, "superseded": 3 }
```

- `superseded` = older queued scans replaced by this one (informational).
- **400** with a message when the scope matches no endpoints; **403** for non-admin callers.
- The "Never scanned (18)" count for the modal label comes from `overview.coverage.neverScanned`.

---

## 5. Overview — `POST /discovery/overview` (empty body `{}`)

One call returns every tile and chart:

```json
{
  "scanJobId": 12,
  "scanStatus": "Running",
  "scanInProgress": true,
  "dataAsOf": "2026-09-09T21:42:10Z",
  "lastScanAt": "2026-09-09T21:10:44Z",
  "filesScanned": 4281530,
  "sensitiveFiles": 96412,
  "unclassifiedFiles": 1204,
  "unencryptedDisks": { "count": 192, "total": 2240 },
  "filesOnUnencryptedDisks": 7412,
  "endpointsScanned": { "count": 782, "total": 800 },
  "classification": [
    { "label": "PHI", "files": 32780, "sharePct": 34.0, "deltaPct": 6.0 },
    { "label": "PII", "files": 25070, "sharePct": 26.0, "deltaPct": 3.0 }
  ],
  "classByVolumeKind": {
    "PHI": { "Local": 14530, "External": 1240, "Mapped": 780 },
    "PII": { "Local": 13120, "External": 1980, "Mapped": 1120 }
  },
  "newSensitiveByWeek": [
    { "weekStart": "2026-08-03T00:00:00Z", "total": 38000, "byClass": { "PHI": 20000, "PFI": 11000, "PCI": 7000 } }
  ],
  "byExtension": [
    { "extension": "xlsx", "files": 4120, "sensitiveBytes": 13743895347, "sensitiveSize": "12.8 GB" }
  ],
  "hotspotFolders": [
    { "path": "C:\\Users\\mock.user\\Downloads", "topClass": "PHI", "files": 12480, "endpoints": 618 }
  ],
  "topUsers": [
    { "userId": 100, "userName": "Mac Ferrari", "emailId": "macferrari.m@company.com", "sensitiveFiles": 2418 }
  ],
  "ageBuckets": { "0-6m": 38420, "6-12m": 26110, "1-3y": 23742, "3y+": 8140 },
  "coverage": { "fullyScanned": 742, "partialInterrupted": 40, "inProgress": 0, "neverScanned": 18, "totalEndpoints": 800 }
}
```

### 5.1 While a scan is running

`scanJobId` is the newest scan that was not cancelled — **the running one while a scan
is in flight**, not the last completed one. So the Overview fills in as the agents
upload, including on a tenant's very first scan, which used to show nothing at all
until the job closed.

- `scanInProgress` is true while that job is Queued or Running. Everything else is
  then a partial picture that grows — label the panel accordingly rather than
  presenting it as a finished scan.
- `scanStatus` is the job's own status: `Queued` / `Running` / `Completed` /
  `CompletedWithErrors`. Null when the tenant has never scanned.
- `lastScanAt` remains the last scan that **finished** — null while the first one is
  still running. Use `scanStatus` for what is happening now.
- `classification[].deltaPct` is null for the whole run: half a scan measured against
  a completed one reads as a collapse in every class, which is an artefact of the
  timing. Deltas come back when the job closes.
- `dataAsOf` stamps **the charts only** — `classification`, `classByVolumeKind`,
  `byExtension`, `ageBuckets`, `hotspotFolders`, `newSensitiveByWeek`. They are rebuilt
  at most once a minute during a run, so they can trail the tiles by up to that much.
  Polling faster than that is free but changes nothing.
- Every tile — `filesScanned`, `sensitiveFiles`, `unclassifiedFiles`,
  `unencryptedDisks`, `endpointsScanned`, `topUsers`, `coverage` — is counted live on
  each call and is **not** subject to `dataAsOf`. So `sensitiveFiles` agrees with
  `topUsers` at all times; expect it to run ahead of what the charts below add up to
  while a scan is in flight.

`filesScanned` is the largest of three figures per endpoint, not their sum: the
per-disk counters (written only when an endpoint finishes), the agent's progress
heartbeat (live, but only as current as the last beat), and the number of file rows we
actually hold. Taking the largest keeps it monotonic — it climbs as a rescan overtakes
the previous total and never goes backwards mid-scan.

The row count matters because the other two lag. Batches of files arrive faster than
heartbeats do, so a scan that had already delivered 1,469 rows was still reporting 23
scanned. **`filesScanned` is therefore guaranteed to be at least
`sensitiveFiles + unclassifiedFiles`** — you will never see the tile claim fewer files
scanned than the sensitive count beside it.

It is still a floor, not a precise total, while a scan runs: clean files are counted
but never stored as rows (§9.1), so the true figure is higher than what we can prove
from rows alone until the endpoint reports its volumes at completion.

`filesOnUnencryptedDisks` is floored the same way, per volume — file rows carry their
volume, so they can be attributed. The heartbeat cannot be: it is per endpoint, with no
way to split it across disks.

`endpointsScanned.count` is endpoints that have **ever finished a scan, or are
scanning now** — not only completed ones, which read 0 all through a first scan.
Triggering a tenant-wide rescan does not reset it.

Coverage during a run: a machine currently being scanned lands in
`coverage.inProgress`. The four buckets are disjoint and add up to `totalEndpoints`.
`totalEndpoints` counts distinct machines — a machine with several endpoint users is
one, matching the By User / Endpoints grid.

Tile bindings:

| Tile / chart | Field |
|---|---|
| Files Scanned | `filesScanned` |
| Sensitive Files | `sensitiveFiles` |
| Unencrypted Disks 192/2,240 | `unencryptedDisks.count` / `.total` |
| Files on Unencrypted Disks | `filesOnUnencryptedDisks` |
| Endpoints Scanned 782/800 | `endpointsScanned.count` / `.total` |
| Classification Breakdown donut + list | `classification[]` (`deltaPct` is the red/green %; `null` on the first scan and throughout a run → hide the badge) |
| Where Each Class Lives matrix | `classByVolumeKind` — columns are `Local` / `External` / `Mapped` (C: and D: are both `Local`; render per design) |
| Newly Discovered Sensitive Files | `newSensitiveByWeek[]` — 6 entries, oldest→newest, stack by `byClass` keys |
| Sensitive Volume by File Type | `byExtension[]` — see limitation §9.1 |
| Hotspot Folders | `hotspotFolders[]` (`topClass` = the chip) |
| Users Holding Most Sensitive Data | `topUsers[]` |
| Age of Sensitive Data | `ageBuckets` (fixed keys `0-6m`, `6-12m`, `1-3y`, `3y+`) |
| Endpoint Scan Coverage | `coverage` (Fully scanned / Partial — interrupted / In progress / Never scanned) |

Performance note: heavy widgets are cached per scan server-side — calling this per page
load is fine. During a run that cache is rebuilt at most once a minute (see §5.1), so
polling the Overview while a scan is in flight is also fine.

---

## 6. By User grids — `POST /discovery/users` and `/discovery/endpoints`

Both routes return the same row shape (one row per machine, with its primary user); the two views just present different columns. Request body is the standard grid `QueryParameters`:

```json
{
  "page": 1, "pageSize": 20,
  "sortBy": "sensitiveFiles", "sortDescending": true,
  "likeFilters": { "userName": ["mac"] },
  "exactFilters": { "encryptionStatus": ["Unencrypted", "Partial"] }
}
```

- Multi-value filters work like the other grids: values within a key OR'ed, keys AND'ed; keys are the camelCase response field names.
- `sortBy` accepted: `userName`, `machineName`, `filesScanned`, `lastScanAt`, `encryptionStatus`, `riskBand`, `classifications`, `agentStatus`, `scanStatus`, `scanProgressPct`, `queuePosition` (default: `sensitiveFiles` desc).
- **An unknown filter key is now a 400**, naming the key, exactly as `/discovery/files` has always behaved. It used to be swallowed, so a typo returned the full unfiltered grid and nothing said so. Allowed keys:
  - `likeFilters`: `userName`, `emailId`, `designationName`, `machineName`, `osName`, `encryptionDetail`
  - `exactFilters`: `classifications`, `disks`, `agentStatus`, `encryptionStatus`, `scanStatus`, `riskBand`, `userId`, `endpointId`, `userName`, `emailId`, `designationName`, `machineName`, `osName`, `runningScanJobId`
- **Volumes filter** → `exactFilters.disks`: a row matches when **any** of its disks is listed. `"C"` and `"C:"` are the same drive, so sending back exactly what the column renders works.

Response:

```json
{
  "totalCount": 800,
  "data": [
    {
      "userId": 100,
      "userName": "Mac Ferrari",
      "emailId": "macferrari.m@company.com",
      "designationName": "Finance Controller",
      "endpointId": 4482,
      "machineName": "IN-LT-4482",
      "osName": "Windows 11 Pro",
      "disks": ["C", "D", "E"],
      "encryptionStatus": "Partial",
      "encryptionDetail": "2 of 3 · E: open",
      "filesScanned": 96410,
      "sensitiveFiles": 2418,
      "classifications": ["PCI", "PFI", "PHI"],
      "riskBand": "High",
      "volumeScannedGb": 412.00,
      "lastScanAt": "2026-09-06T20:44:02Z",
      "scanStatus": "Running",
      "agentStatus": "Active",
      "runningScanJobId": 12,
      "scanStartedAt": "2026-09-09T21:02:11Z",
      "scanProgressPct": 61.4,
      "queuePosition": null,
      "scanStatusReason": null
    }
  ],
  "encryptionCounts": [
    { "value": "Encrypted", "label": "Encrypted", "count": 540 },
    { "value": "Partial",   "label": "Partial",   "count": 192 },
    { "value": null,        "label": "Not reported", "count": 18 }
  ],
  "riskCounts":   [ { "value": "Critical", "label": "Critical", "count": 24 } ],
  "statusCounts": [ { "value": "Running",  "label": "Running",  "count": 31 } ],
  "agentCounts":  [ { "value": "Active",   "label": "Active",   "count": 702 },
                    { "value": "Offline",  "label": "Offline",  "count": 80 },
                    { "value": "Unknown",  "label": "Unknown",  "count": 18 } ]
}
```

- **Disk Encryption column:** render `encryptionStatus` as the badge (`Encrypted` | `Partial` | `Suspended` | `Unencrypted`) and `encryptionDetail` as the subtitle ("XTS-AES 256", "2 of 3 · E: open", "Paused 9 days", "No volume encryption").
  **Those four are the only values, and `null` is the only other thing you can
  get.** The agent reports a volume's encryption once, in the scan-complete
  payload, so a machine with no completed scan has nothing to show: `null`,
  rendered "Not reported", with `encryptionDetail` saying "Reported when the
  scan completes". Null is not a verdict — don't colour it as exposure, and
  don't count it as protected either.
  *(Changed — there used to be an `Unknown` state, for an agent that could not
  read BitLocker. The agent no longer sends it: an unreadable volume now goes
  out as `Unencrypted`. `Unknown` is gone from `encryptionCounts` and from
  `exactFilters.encryptionStatus`, and any legacy row still holding the word is
  served as `null`. A mixed machine's subtitle says "· 1 not reported yet"
  where it used to say "· 1 unreadable".)*
- `classifications` = the class chips. `riskBand` = Critical/High/Medium/Low/Safe —
  the **average** verdict of the endpoint's sensitive files (per file, the higher of
  the agent's `risk` and `severity`; unrated files are excluded). It is NOT the single
  worst file — one critical file on a mostly-benign machine will not read Critical.
  `Safe` = a completed scan found nothing sensitive; `null` = never scanned, or no
  file carries a verdict.
- `scanStatus`: `Completed` | `Running` | `Queued` | `Interrupted` | `Failed` | `TimedOut` | `Cancelled` | `Superseded` | `Never` — the last-scan cell.
  **It is the scan the agent is on**, not the newest job that named the machine:
  a machine that is scanning reads `Running` even when a later job also listed it,
  and `runningScanJobId` names that same scan.
  *(Fixed — the row used to report the machine's newest scan row while the agent
  claims its oldest pending one, so endpoint 15 read `Queued`, `queuePosition: 1`,
  `runningScanJobId: 53` — a job its own scan history did not list, because it had
  never run — while the agent was ten hours into job 42 and the server was acking
  its batches. The ✕ would have cancelled job 53 and left the running scan alone.
  The panel header, the grid and `POST /discovery/scan-history` now agree.)*
  `TimedOut` means the scheduler gave up waiting: the agent never claimed the scan,
  or claimed it and never came back, for longer than the tenant's
  `DiscoveryQueueTimeoutHours` (default 168h / one week). Treat it like `Failed`
  for colour — it is a run that did not happen — and show `scanStatusReason`,
  which names the limit it blew ("Timed out after 168h — the agent never picked
  the scan up"). It appears in the scan-status facet list like any other value.
  *(New — a queued row for an offline agent had no timeout at all, so its job
  could never close and `scanInProgress` stayed true for ever: QA job 41 held 36
  queued machines, 31 of them offline.)*
- `lastScanAt` is **when the machine last FINISHED a scan, whatever a newer job is
  doing**. Queueing a rescan over a machine does not blank it. It is null **only**
  when the machine has never completed one — that, and only that, is
  "Not scanned yet". Superseded, Cancelled and TimedOut rows don't date a scan
  (their timestamp is when the row was closed, not when a scan ran); a Failed run
  does, because it still walked the disk. What a newer job is up to is in `scanStatus`,
  `queuePosition`, `scanStartedAt` and `scanProgressPct`.
  *(Fixed — it used to be read off the newest scan row, so a machine with 97,656
  scanned files and a fresh job queued returned `null`.)*

### 6.1 The Agent column — `agentStatus`

`Active` | `Offline` | `Unknown`, from the machine's newest `dlp."EndpointDailyStatus"`
row. Anything but `Offline` is Active (Online and Locked both mean the agent
reported in). **A machine that has never written a status row is `Unknown`, not
Offline** — "we have never heard from it" and "it told us it was off" are
different claims, and only one blames the machine. Same derivation as the Upload
Activity report's AGENT column, so the two screens cannot disagree.

Filter with `exactFilters.agentStatus`; counts come back in `agentCounts`.

### 6.2 Running %, queue position, status sub-line

| Field | Meaning |
|---|---|
| `runningScanJobId` | The job this machine is running, waiting for, or was interrupted on — the one the **agent** is on. **Non-null is the test for "this row has a cancellable scan"** — feed it to `/discovery/scans/cancel`. Null once nothing is pending. |
| `scanStartedAt` | When the agent actually started. Null while still Queued. |
| `scanProgressPct` | `0`–`99`, or **null for "running, no estimate"**. |
| `queuePosition` | 1-based place among the machines of **its own job** that are still waiting; null otherwise. See below — it is not a turn. |
| `scanStatusReason` | The sub-line, e.g. `"Agent offline"`, `"Cancelled by admin"`, `"Waiting for the agent to pick it up"`. Null when the state needs no explanation. |

**`scanProgressPct` is an estimate and the cap at 99 is deliberate.** A scan cannot
report a true percentage — the agent discovers how much work there is as it does
it — so this is the live file counter measured against what the **same machine's
last completed scan** scanned. It never reaches 100: only the agent's
scan-complete call says a scan is done. `null` means there is no completed scan to
measure against (a first scan) — render an indeterminate bar, **not** 0%. Queued
rows read 0.

**`queuePosition` is ordering, not a turn and not an ETA. The server runs no queue
of its own.** It does not scan one machine at a time: each agent polls, is handed
its own row, and starts — so every machine in a job can be scanning at once, an
offline machine at position 1 does not hold up position 2, and the number does not
count down as other machines finish. Position 1 on two different jobs is normal.
`Queued` means one thing: **that machine's agent has not asked for its work yet** —
either it has not polled since the job was created, or it is offline (then
`agentStatus` is `Offline` and the sub-line says so). A fleet sitting at `Queued`
with live agents is an agent-side symptom, not a server backlog; the counterpart
check is `GET /discovery/scanner` — while the scanner is stopped no agent is handed
anything, and everything stays queued.

Branch your rendering on `scanStatus`, never on `scanStatusReason` — that string is
display copy and may be reworded.

### 6.3 Filter-menu counts

`encryptionCounts`, `riskCounts`, `statusCounts` and `agentCounts` are **faceted**
like `riskCounts` on `/discovery/files`: each list is counted with every *other*
filter applied but not its own, so ticking one encryption state does not zero out
the three options you are still looking at.

Each entry is `{ value, label, count }`. Known options are always present even at
count 0, so a checkbox never disappears mid-interaction. `value: null` is the
unrecorded bucket (`"Not reported"` / `"Not scored"`) and has no filter value —
it is a count, not a tick box.

---

## 7. Detail panel + Discovered Files

### 7.1 `POST /discovery/endpoint-detail` — `{ "endpointId": 4482 }`

```json
{
  "endpointId": 4482,
  "machineName": "IN-LT-4482",
  "osName": "Windows 11 Pro",
  "userId": 100,
  "userName": "Mac Ferrari",
  "emailId": "macferrari.m@company.com",
  "designationName": "Finance Controller",
  "encryptionStatus": "Partial",
  "encryptionDetail": "2 of 3 · E: open",
  "scanStatus": "Interrupted",
  "lastScanAt": "2026-09-06T20:44:02Z",
  "filesScanned": 96410,
  "sensitiveFiles": 2418,
  "unclassifiedFiles": 1204,
  "volumeOnDiskGb": 3584.00,
  "scannedGb": 3584.00,
  "volumes": [
    {
      "driveLetter": "C", "kind": "Local", "sizeGb": 512, "scannedGb": 118.40,
      "encryptionStatus": "Encrypted", "encryptionMethod": "XTS-AES 256",
      "filesScanned": 48220, "sensitiveCount": 911, "sensitivePctOfFiles": 1.9
    }
  ]
}
```

Maps 1:1 to the panel header + tiles + per-volume cards (`sensitivePctOfFiles` =
the "19% of files" figure). `emailId` / `designationName` are the header
subtitle; `scanStatus` is the "Scan status" chip (`Never` when the endpoint has
no scan row); `encryptionDetail` is the text behind the Encryption value, same
wording as the grid's `encryptionDetail`.

`lastScanAt` follows the same rule as the grid (§6): the last scan the machine
**finished**, whatever a newer job is doing, and null only when it has never
completed one.

`volumeOnDiskGb` and each card's `sizeGb` are **capacity only** — what the
agent measured off the disk (`DriveInfo.TotalSize`) and sent with the
completion payload. Both are `null` until a scan completes and reports it, and
are never derived from anything else. The grid's `diskSizeGb` (§6) is the same
figure.

`scannedGb`, on the panel and on each card, is the other number: the bytes of
the inventory rows we hold, floored by reported capacity. It moves as a scan
uploads and says nothing about how big the disk is. It is the grid's
`volumeScannedGb` (§6), to the same decimal, and is `null` only when there is
no reported capacity **and** nothing stored — the genuine Never case.

> Until 2026-11-06 `sizeGb` carried the floored figure, so an endpoint with no
> completed scan reported the size of what it had uploaded as though it were
> the disk's capacity — INN274-SHAIKHMO showed `23.31` for a 474.7 GiB C:
> drive. A UI binding `sizeGb` to a disk-capacity label should now expect
> `null` before the first completed scan, and bind `scannedGb` wherever it
> wants the old behaviour.

### 7.2 `POST /discovery/files`

```json
{
  "endpointId": 4482,
  "tab": "Sensitive",
  "search": "payroll",
  "scanJobId": 2104,
  "page": 1, "pageSize": 15,
  "sortBy": "lastModified", "sortDescending": true
}
```

- `tab`: `Sensitive` | `All` | `Unclassified` — see limitation §9.2 for what "All" means.
- `search` matches file name OR path, case-insensitive contains.
- `scanJobId` (optional) — the "Found in" dropdown. A value from
  `POST /discovery/scan-history` (§7.3) narrows the grid to the files **first
  detected** by that scan; rows, `totalCount` and the facet counts all narrow
  together. Omit it for "All scans". A job that never ran on the endpoint is a
  400.
- `sortBy`: `fileName`, `size`, `firstSeenAt`, `confidence`, `risk`, `severity`, default `lastModified`.
  `risk` and `severity` sort by **severity, not alphabetically** (CRITICAL → SAFE,
  case-insensitively), with unrecorded and unrecognised values last on a
  descending sort.
- `exactFilters` keys: `classifications`, `risk`, `severity`, `dpdpFlag`, `extension`
  and **`driveLetter`** (the C:/D: disk filter — `"C"` and `"C:"` both work).
  `likeFilters`: `fileName`, `directory`, `extension`, `severityReason`. Unknown
  key → 400, naming it.
- The "Found in" filter is the **top-level `scanJobId`**, not an exact filter —
  see §7.3.

Row:

```json
{
  "fileId": 991,
  "fileName": "Payroll_Master_2026.xlsx",
  "directory": "D:\\Finance\\Payroll\\2026",
  "driveLetter": "D",
  "extension": "xlsx",
  "sizeBytes": 50537240,
  "size": "48.2 MB",
  "classifications": ["PFI", "PII"],
  "lastModified": "2026-09-02T10:11:00Z",
  "firstSeenAt": "2026-08-28T02:14:00Z",
  "foundInScanJobId": 2104,
  "foundInScanAt": "2026-07-12T02:00:14Z",
  "foundInScanType": "Full",
  "confidence": 87.5,
  "risk": "HIGH",
  "severity": "CRITICAL",
  "severityReason": "Health insurance + Email + Phone",
  "dpdpFlag": "Protect",
  "sensitiveContent": {
    "PFI": ["#### 2### 3### 4###"],
    "PII": ["xxxx.xxxxx@acme.co.in"]
  }
}
```

#### The "Found in" column

`foundInScanJobId` / `foundInScanAt` / `foundInScanType` name the scan that
**first detected** the file — render as "12/07/26 · Full scan". `foundInScanType`
is `Full` or `Incremental` (what actually ran); append the word "scan" yourself.
All three are null when the endpoint has no scan history to place the file in.
Clicking the cell = resend the request with that `foundInScanJobId` as
`scanJobId`.

### 7.3 `POST /discovery/scan-history` — `{ "endpointId": 4482 }`

The "Found in" dropdown: every scan that ran on the endpoint, newest first.

```json
{
  "endpointId": 4482,
  "scanCount": 4,
  "firstScanAt": "2026-05-03T02:00:09Z",
  "totalFiles": 10,
  "scans": [
    {
      "scanJobId": 2231, "scanDate": "2026-09-06T02:00:11Z",
      "completedAt": "2026-09-06T02:38:40Z", "scanType": "Incremental",
      "status": "Completed", "durationSeconds": 2280, "durationDisplay": "38 min",
      "newFiles": 2, "isBaseline": false, "isLatest": true
    },
    {
      "scanJobId": 2001, "scanDate": "2026-05-03T02:00:09Z",
      "completedAt": "2026-05-03T04:46:12Z", "scanType": "Full",
      "status": "Completed", "durationSeconds": 9960, "durationDisplay": "2 h 46 min",
      "newFiles": 4, "isBaseline": true, "isLatest": false
    }
  ]
}
```

- Header line = `scanCount` + `firstScanAt` + `totalFiles` ("All scans · 4 scans
  since 03/05/26 · 10 files total").
- `newFiles` = stored files that scan first detected ("+2 new files"); on the
  baseline row it is everything the first scan found ("4 found · baseline").
  **First-detected, not files-seen.** A re-scan of a machine whose files are all
  already on record reports `newFiles: 0` while happily uploading thousands of
  rows — that is the right answer to "what did this scan find that we did not
  have?", and picking that scan in the files grid correctly shows nothing. Label
  the column "New files", never "Files scanned"; the files the run is walking
  are the row's `filesScanned` in §6, which updates as batches are acked.
- The `newFiles` figures add up to `totalFiles` — a file is first-detected by
  exactly one scan.
- `isLatest` drives the "Latest" badge; `durationDisplay` is pre-formatted,
  null while the scan is still running.
- Only scans the endpoint's agent **actually started** appear — a job that was
  queued and never claimed has nothing to show. Status is not consulted, so a scan
  that ran and was later superseded or cancelled still appears and still owns the
  files it found.
  *(Fixed — Superseded rows used to be excluded. An agent that uploads files and
  then goes silent gets its row swept to Interrupted and then superseded by the
  next job, which erased the scan from the history: QA endpoints 25 and 19 showed
  11,215 and 440 files with `scanCount: 0`.)*
- **`newFiles` across all scans sums to `totalFiles`** — every stored file belongs
  to exactly one scan, because the oldest window is open-ended and catches
  anything older than it. If the two ever disagree, the machine holds files from a
  scan that is not in the list, which is a bug worth reporting.
- A scan appearing here does **not** imply `lastScanAt` is set: a run that was
  interrupted or superseded ran but never *finished*, and `lastScanAt` only counts
  finished scans (§6). A machine can legitimately show scan history, files, and no
  scan date — it means no scan on it has ever completed.

#### The risk columns

All five are **nullable, and null means "not recorded" — never a benign verdict**.
Render null as an empty cell or a muted dash; do not substitute `LOW`, `Safe` or
`0`. An older agent build, or a row written before these columns existed, sends
none of them.

| Field | Values | Notes |
|---|---|---|
| `confidence` | `0`–`100`, two decimals | Detection confidence. A number, not a percentage string — format it yourself. |
| `risk` | agent's label, e.g. `"HIGH"` | **Verbatim, NOT normalised.** |
| `severity` | agent's label, e.g. `"CRITICAL"` | Verbatim. |
| `severityReason` | free text, e.g. `"Health insurance + Email + Phone"` | Names the classes that fired. Shown as sent — render as text, and escape it like any other server string. |
| `dpdpFlag` | e.g. `"Protect"` | DPDP disposition. Verbatim. |

These four are stored exactly as the agent sent them — the server validates and
rewrites nothing — so both the casing and the vocabulary are the agent's, and a
value you have never seen is a real possibility. A mismatched agent build
surfaces here rather than being silently rewritten, so render an unknown value
as-is instead of falling through to a default chip.

Note this is **not** the same field as the endpoint grid's `riskBand` (§6): that
one is the normalised (`Critical`/`High`/…) average of these per-file verdicts
across the endpoint — don't feed both into one chip component expecting matching
casing.

#### `sensitiveContent` — the detection-detail panel

What tripped detection on the file, the discovery equivalent of the transfer-log
grid's `incidentInfoJson`. Render it as the same "Sensitive Info" panel.

- **The values arrive masked and there is no unmasked form.** They were masked
  before they were ever written, so `4111 2222 3333 4444` is stored and returned as
  `#### 2### 3### 4###`. Digits become `#`, letters `x`, separators survive, and
  emails keep their domain. Do not build an "unmask" or "reveal" affordance — there
  is nothing behind it, and the shape is the whole point: a card still looks like a
  card.
- **`null` means "not shown to you", and you cannot tell why.** It is null when the
  tenant's `Discovery:ShowSensitiveContent` setting is off, when the row predates
  the column, and when the agent sent no detail — deliberately indistinguishable.
  Hide the panel on null; don't show an error or an empty-state that implies data
  was expected.
- The setting is a **ClientSetting row, not a UI toggle**. There is no endpoint to
  flip it; it is provisioned per tenant. It is now **opt-OUT**: unset tenants get
  the panel, and only `"0"` or `"false"` turns it off. (It was opt-in, which meant
  every tenant saw `"sensitiveContent": null` with no way to switch it on — if you
  built around the panel never appearing, it appears now.)
- The shape is whatever the agent sent, masked in place — treat it as an arbitrary
  JSON object and render generically rather than binding to fixed keys.

---

## 7.4 Cancel a scan — `POST /discovery/scans/cancel`

```json
{ "scanJobId": 12 }                      // Cancel scan — the whole run
{ "endpointId": 4482 }                   // Cancel ✕ — stop this machine
{ "scanJobId": 12, "endpointId": 4482 }  // same as above; the job id is ignored
```

Response:

```json
{
  "scanJobId": 12,
  "cancelledJobIds": [12],
  "cancelledRunning": 3,
  "removedQueued": 11,
  "jobClosed": true,
  "jobsClosed": 1,
  "jobStatus": "CompletedWithErrors"
}
```

- **Admin only (403 otherwise)** — same rule as starting a scan. Stopping a
  tenant-wide run is as consequential as launching one.
- **A scan does not have to have started.** Queued rows are cancelled exactly
  like Running ones — most of a fleet job is work that has not started, and a
  stop that only reached a live scan would stop almost nothing. Interrupted rows
  go too: they are resumable (the laptop-switched-off case), so leaving them
  would have the run quietly restart days later on a scan somebody stopped.
- **Naming a machine stops the machine, not one row of it.** `endpointId`
  cancels everything that endpoint has pending, across every job, and
  `cancelledJobIds` lists what was touched. `scanJobId` is optional once you
  send `endpointId`, and is ignored when you send both — send either, or both,
  and the machine stops.
  *(Fixed — a machine holding a Running row with a newer Queued one behind it
  was cancelled only in the job the ✕ named, then claimed the other on its next
  poll: the button reported success and the disk kept being crawled.)*
- **400 when there is nothing left to cancel** — the run already finished, or
  somebody else cancelled it first. Show the message; it is not a server fault.
- `jobStatus` after a full cancel is `Cancelled` if the run delivered nothing, or
  `CompletedWithErrors` if some machines had already finished — their results are
  real and the dashboard still reads them.
- **Agents are not pushed to.** A cancelled scan stops because the agent's next
  poll returns nothing and its next upload is refused, so a machine mid-file may
  keep working for up to one heartbeat (5 min) before it notices. Treat the ✕ as
  "stop requested", and let the row's `scanStatus` → `Cancelled` confirm it.
  A Queued row stops the moment you call this — there is no agent to notice.
- Enable the ✕ on a row whenever `runningScanJobId` is non-null — which includes
  `Queued` and `Interrupted` rows, not just `Running` ones.

---

## 7.5 Stop the DDC scanner — `GET` / `POST /discovery/scanner`

The tenant-wide off switch, distinct from cancelling one run: while the scanner
is stopped, **no** scans run, queue, or reach agents — the scheduler skips its
slots, `POST /discovery/scans` returns 400, and agent polls return nothing —
until it is started again.

`GET /discovery/scanner` (any console role):

```json
{ "enabled": true, "running": 3, "queued": 11 }
```

`running`/`queued` are the endpoint scans a stop would end / remove — use them
in the confirmation dialog ("Stop the scanner? 3 machines are scanning now and
11 are queued.").

`POST /discovery/scanner` (admin only, 403 otherwise):

```json
{ "enabled": false }
```

Response:

```json
{ "enabled": false, "cancelledRunning": 3, "removedQueued": 11, "jobsClosed": 1 }
```

- **Stopping cancels every in-flight endpoint scan across all jobs** and closes
  the open jobs (same closing rule as a cancel: a run that completed some
  machines reads `CompletedWithErrors` and keeps those results; one that
  delivered nothing reads `Cancelled`).
- **Agents are not pushed to** — same as a cancel, a machine mid-file may keep
  working for up to one heartbeat (5 min) before it notices. The switch also
  reaches the EXE as the `DiscoveryScannerEnabled` key in
  `GetEndpointExeSettings`, so agent builds that honor it stop without even
  polling.
- `{ "enabled": true }` starts the scanner again. Nothing cancelled is
  resurrected — the next schedule slot or a manual run queues fresh scans.
- The header strip (`GET /discovery/scans/latest`) now carries
  `scannerEnabled`, so the dashboard can show the state without an extra call.
- The schedule itself is untouched by the switch: stopping does not flip
  `isActive` on `/discovery/schedule`, it just gates it. Starting again resumes
  the existing cadence.

---

## 8. Schedule — `GET` / `POST /discovery/schedule`

```json
{
  "cadence": "Weekly",
  "runAtLocal": "02:00",
  "timeZone": "Asia/Kolkata",
  "defaultScanType": "Incremental",
  "defaultVolumes": ["C", "D", "E", "Mapped"],
  "isActive": true
}
```

POST takes the same shape (admin-only, 403 otherwise). `runAtLocal` must be `HH:mm`. `cadence`: `Daily` | `Weekly` | `Monthly`.

---

## 9. Known limitations / behaviours to design for

1. **Sensitive Volume by File Type:** the backend stores rows only for sensitive + unclassified files (clean files are counted, not stored — design decision 8.1). So `byExtension[]` carries **sensitive** bytes per type; the "total volume scanned per type" bar from the mock-up is not derivable. Size the bars on `sensitiveBytes` / `sensitiveSize`.
2. **"All files" tab:** same reason — "All" lists the stored rows (sensitive + unclassified). Clean files appear in the counts (`filesScanned`) but have no rows to list. If the tab must show a number, use the volume card counts.
3. **Weekly cadence has no day-of-week** — the next run is 7 days after the previous scheduled run, at `runAtLocal` tenant-local time. If product wants "every Monday", request it as a backend change.
4. **`deltaPct` is `null` on the first scan** (nothing to compare against) — hide the trend badge, don't show 0%.
5. **Mock data phase:** until the EXE agent integrates, numbers are synthetic (seeded per tenant). Shapes and behaviours are final.
6. **403 handling:** scan/schedule POSTs return the standard envelope with `success:false` and a message — show it as a permission toast.
7. New routes 404 from the gateway until the APIM operations are registered — a `{"statusCode": 404, "message": "Resource not found"}` body means APIM, not a wrong URL in your code. **`/discovery/scans/cancel`, `/discovery/scan-history` and `GET`/`POST /discovery/scanner` are new and need registering.**
8. **`scanProgressPct` is an estimate, never 100** (§6.2). There is no true percentage to report; design the bar for "61%" and for "running, unknown".
9. **Cancel is not instant** (§7.4) — there is no push channel to the agent, so a machine can keep scanning for up to one heartbeat after the ✕.
10. **A fleet stuck at `Queued` is not a server backlog** (§6.2). Nothing serialises scans: the server hands every agent its own row the moment that agent polls. So `Queued` on a machine you believe is online means its agent has not polled `/discovery/scan` — check, in this order: `GET /discovery/scanner` (`enabled:false` → nothing is handed out at all), the row's `agentStatus`, and then the agent build on the machine (a DLP agent without the discovery poll never claims anything). Rows nobody ever claims are closed as `TimedOut` after `DiscoveryQueueTimeoutHours`.

---

## 10. Quick test recipe (Postman)

1. Log into the QA dashboard as a **DLP Admin**, copy `Authorization` + `ClientGUID` from any request in the Network tab.
2. `POST /discovery/overview` with `{}` → all tiles populated (mock data).
3. `POST /discovery/users` with `{"page":1,"pageSize":10}` → grid rows.
4. `POST /discovery/endpoint-detail` with an `endpointId` from step 3 → tiles + volume cards.
5. `POST /discovery/files` with the same `endpointId`, `tab:"Sensitive"` → file rows.
6. `POST /discovery/scans` `{"scope":"All","scanType":"Full"}` → job queued (verify 403 with a non-admin token).
7. `POST /discovery/users` `{"exactFilters":{"disks":["C"]}}` → only machines with a C: drive; `{"exactFilters":{"volumes":["C"]}}` → **400** naming the bad key.
8. `POST /discovery/scan-history` `{"endpointId":4482}` → the scan list; feed a `scanJobId` back as `{"endpointId":4482,"scanJobId":2231}` on `/discovery/files`.
9. `POST /discovery/scans/cancel` `{"scanJobId":<the job from step 6>}` → counts of what was stopped; call it twice and the second is a **400** ("nothing left to cancel").
10. `POST /discovery/scanner` `{"enabled":false}` → everything in flight cancelled; `POST /discovery/scans` now returns **400** ("scanner is stopped") and `GET /discovery/scans/latest` shows `scannerEnabled: false`. Finish with `{"enabled":true}` to put the tenant back.
