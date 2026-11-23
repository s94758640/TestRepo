# Data Discovery & Classification — EXE Agent Integration (DLP API)

- **Audience:** EXE agent team
- **What this is:** the same four discovery routes you already have in `SFM.DLP.ExeApi`, now also served by the **DLP API** — the host you already call for `GetEndpointExeSettings`.
- **Contracts in code:** `SFM.DLP.Application/DTOs/DiscoveryAgentDTO.cs` · controller: `SFM.DLP.API/Controllers/DiscoveryAgentController.cs` · design: `DATA-DISCOVERY-IMPLEMENTATION-PLAN.md`

---

## 1. Read this first

**The payloads have not changed.** Routes, field names, JSON casing, status codes and the resume state machine are identical to the function-app version in `Data-Discovery-Agent-Integration.md`. If you already built against the ExeApi, porting is a **base-URL change and nothing else**.

What *is* different is the host and therefore the headers:

| | ExeApi (function app) | DLP API (this document) |
|---|---|---|
| APIM route | the function app's API | `/qadlp` — the same one as `GetEndpointExeSettings` |
| Tenant selection | derived from the agent credential | **`ClientGUID` request header** — required |
| Agent credential | `BasicAuth: {ClientGUID}_{serialNo}` or `Authorization: Bearer <agent jwt>` | **not checked** — these routes are anonymous, like the other agent routes on this host |
| Ownership check | body `endpointId` vs token, *and* vs the scan row | body `endpointId` vs the scan row |

**Say this part out loud so nobody is surprised later:** on the DLP API the agent bearer token is *not* validated, because the DLP API has no agent-auth middleware — `GetEndpointExeSettings` and the other agent routes there are anonymous today for the same reason. The protection that remains is real but narrower: the APIM subscription key, the `ClientGUID` header that picks the tenant database, and the server-side rule that **the `endpointId` in your body must match the machine that owns the scan**. Knowing an `endpointScanId` is not enough to write into it. But a caller inside the tenant can still name another machine's `endpointId` here, which the ExeApi's token scoping would have stopped.

Consequence for your rollout: **if you already ship against the ExeApi, keep using it.** Move to the DLP API when you want discovery on the same host as your other DLP calls, or in on-premises installs where the function app is not deployed. Do not point half the fleet at each — the two write to the same tables and there is no reason to.

### Base URL and headers

```
Base:  https://qa-email.azure-api.net/qadlp

Ocp-Apim-Subscription-Key: <your APIM key>          (hosted only; absent on-prem)
ClientGUID:                <the tenant GUID>        REQUIRED on every call
Content-Type:              application/json         on the three POSTs
```

Miss the `ClientGUID` header and you will land on the default database, not your tenant's — you will get `204` forever from the poll and `404` from everything else. It is the single most likely cause of "it works in Postman for someone else but not for me".

---

## 2. What the agent has to do

Data-at-rest discovery: on request, walk the machine's disks, classify file contents with the **same DLP content engine you already run for transfers**, and report results in batches.

You do **not** upload file content, matched keywords, or samples — metadata and class labels only. That is a hard product boundary, not an optimisation.

The flow rides your existing patterns: you already poll `V1/GetEndPointCommand`; discovery adds one more poll target and three upload calls.

```
  poll  ──►  GET  V1/GetDiscoveryScanJob        204 → sleep, 200 → you have a scan
             │
             ├─ every ≤300s ──►  POST V1/DiscoveryScanProgress    (heartbeat)
             ├─ per ≤500 files ─►  POST V1/DiscoveryFilesBatch     (results)
             └─ once, always ──►  POST V1/DiscoveryScanComplete    (finalise)
```

---

## 3. The four endpoints

### 3.1 `GET V1/GetDiscoveryScanJob?serialNumber=<serial>` — or `?endpointId=<id>`

Poll on your normal command cadence. **One identity is enough**:

- `serialNumber` — preferred; the hardware serial you already send on every other call.
- `endpointId` — fallback for machines without a usable BIOS serial (VMs, clones, "To be filled by O.E.M."); you know it after registration and from every previous poll response.
- Sending both is allowed (serial is tried first). Sending neither is a **400**. `endpointId=0` counts as not sent.
- `endpointUserId` — **optional, and not an identity on its own.** An EndpointUserId is unique per machine, not across the tenant, so sending it alone is still a 400. Send it alongside one of the two identities and it does two useful things: it picks the exact endpoint row when one serial maps to several signed-in accounts (without it the primary user wins), and it is recorded against the scan so the dashboard can show which agent instance ran it.
- Either identity must resolve to an **active** endpoint row; an inactive or unknown machine gets **204**, not an error.

Responses:

- **204 No Content** — nothing to do. This is the common case; treat it as normal, not as a failure.
- **200** — a scan to run:

```json
{
  "success": true,
  "data": {
    "endpointScanId": 5012,
    "scanJobId": 12,
    "endpointId": 37,
    "endpointUserId": 2,
    "scanType": "Incremental",
    "volumes": ["C", "D", "E", "Mapped"],
    "includeMapped": true,
    "includeExternal": true,
    "volumeCursors": [
      { "driveLetter": "C", "usnCursor": 1000003, "snapshotVersion": null },
      { "driveLetter": "D", "usnCursor": null,    "snapshotVersion": null }
    ],
    "batchSize": 500,
    "heartbeatSeconds": 300,
    "maxFileSizeBytes": 524288000,
    "fileTypes": [
      { "fileTypeMasterId": 3, "fileType": "csv",  "fileTypeDescription": "Comma separated values" },
      { "fileTypeMasterId": 1, "fileType": "pdf",  "fileTypeDescription": "Portable Document Format" },
      { "fileTypeMasterId": 7, "fileType": "xlsx", "fileTypeDescription": "Excel workbook" }
    ]
  }
}
```

Rules:

- The first read of a Queued scan **claims it** — the server marks it Running before replying. If you poll again, e.g. after a reboot, you get the **same `endpointScanId`** back: resume it, don't restart from zero.
- `scanType` is what was *requested*. Per volume, if `usnCursor`/`snapshotVersion` is null, or your USN journal wrapped, or the volume isn't NTFS, **upgrade that volume to full** and report `executedType` accordingly at the end.
- `volumes` is the list of **fixed drive letters**, and may list letters the machine doesn't have — skip them silently.
- **`includeMapped` / `includeExternal` are how the non-fixed volumes are requested**, because their letters are not knowable when the job is queued — they depend on who is signed in and what is plugged in. Resolve them on the machine at scan time:
  - `includeMapped: true` → also enumerate and scan **mapped network drives** (`DriveType` 4), whatever letters they have, and report each one in `DiscoveryScanComplete` with `"kind": "Mapped"`.
  - `includeExternal: true` → also enumerate and scan **removable / external volumes** (`DriveType` 2 and external fixed disks), reported with `"kind": "External"`.
  - Either flag false → skip that class of volume entirely.
  - These two flags are what fill the console's **Mapped** and **External** columns; they stay at 0 until the agent honours them.
  - `volumes` still also carries the tokens `"Mapped"` / `"External"` when the matching flag is set, so older builds are unaffected. Read the flags; treat non-letter entries in `volumes` as noise.
- `endpointUserId` comes back in the response — **echo it on the three POSTs**. It may be null, which is not an error: the scan is per machine, and null just means the server could not pin a user context to it.
- `batchSize` and `heartbeatSeconds` are server-set (500 / 300s today). Read them from the response rather than hard-coding, so we can tune them without an agent release.
- `fileTypes` is every **active** row of the tenant's file-type master — the types to classify. Same shape `GetAllFileTypeMaster` returns, so you can parse it with the code you already have, and it ships with the job so a scan needs no second call before it starts. Match `fileType` case-insensitively and tolerate the leading dot being absent (`pdf` and `.pdf` both occur). Retired types are omitted, so treat the list as the whole allowed set and re-read it on every poll rather than caching it across runs.
- `maxFileSizeBytes` is the size ceiling, in **bytes** — **skip** anything larger, do not fail it and do not report it as an error. Server-set for the same reason as `batchSize`: read it, don't hard-code it. It defaults to 500 MB (`524288000`), matching the limit the transfer pipeline already applies.

### 3.2 `POST V1/DiscoveryScanProgress` — the heartbeat

At least every `heartbeatSeconds` while scanning:

```json
{ "endpointScanId": 5012, "endpointId": 37, "endpointUserId": 2, "filesScanned": 18230, "sensitiveFound": 411, "currentVolume": "C" }
```

Miss it for 15 minutes and the server marks the scan **Interrupted**. That is not fatal: your next `GetDiscoveryScanJob` poll returns the **same scan again** (flipped back to Running) — resume from your last acked batch. An Interrupted scan only stops being resumable when a newer scheduled job supersedes it.

A `DiscoveryFilesBatch` call also counts as a heartbeat, so an agent that is steadily uploading will not be swept. Send the heartbeat anyway during long stretches with no sensitive hits — a full scan of a clean 2 TB volume can go a long time without producing a single batch.

> **`endpointId` is REQUIRED on all three POST bodies** (progress, batch, complete) — use the value from `GetDiscoveryScanJob`. The server checks it against the scan row: omit it or send `0` and you get **400**; send someone else's and you get **404**.
>
> **`endpointUserId` is OPTIONAL on the same three bodies** and is deliberately **not** part of that check — an Interrupted scan can legitimately be resumed under a different signed-in account on the same machine, so a mismatch is recorded, never rejected. Omit it and whatever is already recorded is kept, so an older agent never blanks a value a newer one wrote.

### 3.3 `POST V1/DiscoveryFilesBatch` — results, ≤500 per call

Send **only sensitive and unclassified** files (decision 8.1). Clean files are never sent as rows — they ride the counters.

```json
{
  "endpointScanId": 5012,
  "endpointId": 37,
  "endpointUserId": 2,
  "batchSeq": 1,
  "files": [
    {
      "volume": "D",
      "directory": "D:\\Finance\\Payroll\\2026",
      "fileName": "Payroll_Master_2026.xlsx",
      "extension": "xlsx",
      "sizeBytes": 50537240,
      "lastModified": "2026-09-02T10:11:00Z",
      "classifications": ["PFI", "PII"],
      "confidence": 87.5,
      "sensitiveContent": { "PFI": ["4111 2222 3333 4444"], "PII": ["ravi.kumar@acme.co.in"] },
      "risk": "HIGH",
      "severity": "CRITICAL",
      "severityReason": "Health insurance + Email + Phone",
      "dpdpFlag": "Protect",
      "changeKind": "New"
    },
    {
      "volume": "C",
      "directory": "C:\\Users\\x\\Documents",
      "fileName": "Scan_Corrupt.tmp",
      "extension": "tmp",
      "sizeBytes": 120400,
      "classifications": [],
      "changeKind": "New"
    }
  ]
}
```

- `classifications`: your engine's DlpType labels (PHI/PII/PFI/PCI/CODE/CREDS…). **Empty array = unclassified** — the file was scanned but unreadable or unsupported. Send those too; they are a dashboard bucket, not an error.
- `sensitiveContent` (optional): what tripped detection, as a JSON **object** — the same shape you already send as `sensitiveContent` on a transfer. **The server masks it before storing it**, so `4111 2222 3333 4444` is written as `#### 2### 3### 4###`: the shape survives, the value does not. The masking is one-way and there is no setting that turns it off, so do not send this expecting it to be readable back. Anything that is not a JSON object is stored as a placeholder recording only its length. Omit it and the file's detail panel is simply empty.
- `risk`, `severity`, `dpdpFlag`, `severityReason` (all optional): your verdict for the file, stored **exactly as you send it** — the server validates and rewrites nothing, so a label outside the usual vocabulary reaches the grid rather than being dropped or normalised into one that looks official. `severityReason` is free text ("Health insurance + Email + Phone"): name the **classes** that fired, never the matched values, because unlike `sensitiveContent` this field is **not** run through the masker — it is stored and displayed as sent.
- Omitting any of these four stores `NULL`, which the grid reads as "not recorded" and renders as an empty chip. `NULL` is never treated as a benign verdict, so there is no harm in sending nothing until your build computes them.
- `changeKind`: `New` | `Modified` | `Deleted`. Incremental scans send `Deleted` for files that disappeared; full scans send New/Modified only. A `Deleted` for a file the server has never seen is silently ignored and not counted in `processed`.
- `extension` is normalised server-side (leading dot stripped, lowercased) — send it however you have it.
- `volume` is matched case-insensitively and trimmed. A drive letter the server hasn't seen gets a volume row created on the spot; the full details arrive with scan-complete.
- `batchSeq`: 1-based, strictly increasing per scan. Keep a local journal of acked batches; after a crash, resume from the first un-acked one. Server-side replay dedup is **not** enforced — re-sending an acked batch is harmless because every write is an upsert keyed on (volume, directory, fileName), but don't rely on it for counting.
- Over 500 files in one call is a **400**, not a truncation.

Response:

```json
{ "success": true, "data": { "endpointScanId": 5012, "batchSeq": 1, "processed": 500 } }
```

`processed` can be lower than `files.length` when the batch contained `Deleted` entries for files the server never had. That is expected; don't treat it as a partial failure.

### 3.4 `POST V1/DiscoveryScanComplete` — always the last call

```json
{
  "endpointScanId": 5012,
  "endpointId": 37,
  "endpointUserId": 2,
  "executedType": "Incremental",
  "filesScanned": 96410,
  "sensitiveFound": 2418,
  "errorDetail": null,
  "volumes": [
    {
      "driveLetter": "C", "kind": "Local", "fileSystem": "NTFS", "sizeGb": 512,
      "encryptionStatus": "Encrypted", "encryptionMethod": "XTS-AES 256",
      "suspendedSince": null,
      "usnCursor": 1000451, "snapshotVersion": null,
      "filesScanned": 48220, "sensitiveCount": 911, "unclassifiedCount": 610
    },
    {
      "driveLetter": "E", "kind": "External", "fileSystem": "NTFS", "sizeGb": 2048,
      "encryptionStatus": "Unencrypted",
      "usnCursor": null,
      "filesScanned": 13384, "sensitiveCount": 303, "unclassifiedCount": 94
    }
  ]
}
```

- `executedType` = what actually ran: `Full` when you upgraded, `Incremental` otherwise. **Required** — omitting it is a 400.
- **`usnCursor`** = the new journal position after this scan. The server hands it straight back on your next poll; it *is* your incremental state, so a scan that completes without one forces the next scan of that volume to full. Volumes scanned by inventory-diff report `snapshotVersion` instead.
- **Encryption**: read BitLocker per volume via WMI `Win32_EncryptableVolume` at scan end. `encryptionStatus`: `Encrypted` | `Partial` | `Suspended` | `Unencrypted`; include `encryptionMethod` (e.g. `XTS-AES 256`) when protected and `suspendedSince` when protection is paused.
- A volume you scanned last time but that is now absent — unplugged external, unmapped share — **just omit it**. The server deactivates it. Sending it with zeroes would wipe its cursor and force a full rescan when it comes back.
- Reporting an **empty** `volumes` array is allowed and means "I enumerated nothing" — typically a scan that died early. It does **not** deactivate the endpoint's known volumes; only a non-empty list prunes the ones it omits. So a crashed scan does not cost you your cursors.
- Fatal failure: **still call complete**, with `errorDetail` set. The scan records as Failed instead of hanging until the Interrupted sweep, and the dashboard shows a reason instead of a silence.
- `filesScanned` counts **everything** you enumerated, clean files included — this is where clean files are accounted for. The per-volume `filesScanned` should sum to roughly the top-level figure; the dashboard reads both.

Response: `{ "success": true, "data": { "endpointScanId": 5012 } }`

---

## 4. Status codes and what to do about them

| Code | Meaning | Your move |
|---|---|---|
| **200** | Accepted | Continue; on a batch, mark that `batchSeq` acked |
| **204** | No scan pending (poll only) | Sleep until the next poll. Not an error |
| **400** | Contract violation — missing `endpointScanId` / `endpointId`, missing `executedType`, >500 files | **A bug to fix, not to retry.** Retrying sends the same invalid body forever |
| **404** | The scan is not in a writable state — superseded by a newer job, or swept Interrupted while you were away | Abandon the call and re-poll `GetDiscoveryScanJob`. Interrupted → you get the **same** `endpointScanId` back and resume. Superseded → you get the newer job, or 204 |
| **500** | Server-side failure | Retry with backoff. The body carries the message; it is logged server-side too |

Every response, success or failure, is the standard envelope: `{ "success": bool, "message": string|null, "data": …, "errors": string[]|null }`. On success `errors` is null and `data` carries the payload; on failure `data` is null and `message` carries the reason. The only exception is **204**, which has no body at all — don't try to parse one.

> **Tenant stop switch.** An admin can stop the DDC scanner for the whole
> tenant. To your agent it looks like a cancel of everything: in-flight scans
> 404 on their next write, and the poll returns 204 until the scanner is
> started again. The switch is also served to you as the
> `DiscoveryScannerEnabled` key in `GetEndpointExeSettings` (`"0"`/`"false"` =
> stopped; absent = running) — honor it if you want to stop scanning without
> waiting for the next refused write, but the server enforces it either way.

Network failures and 500s are the ones to retry. Keep the retry local to the call — never restart a scan because a single heartbeat failed.

---

## 5. Behaviour rules (agreed product decisions)

| Rule | Decision |
|---|---|
| Files to classify | **Identical to the transfer engine** — same file-type list, same zip handling, same max-size guard. Everything else counts as clean without content reading (8.7) |
| Rows to send | Sensitive + unclassified only; clean = counters (8.1) |
| Throttling | Low-priority I/O + configurable CPU / files-per-second cap; the scan runs while the user works and the machine must stay responsive (8.8) |
| Offline at scan time | Nothing special to do: the job stays claimable until the next scheduled run supersedes it — you pick it up on the next poll after boot (8.6) |
| First scan of a volume | Always full — there is nothing to be incremental against |
| One scan at a time | Never run two discovery scans concurrently on a machine |
| Content leaving the device | Never. Metadata, class labels and confidence only |

---

## 6. Test checklist

Run these against QA before piloting. Each one exercises a branch that has bitten someone.

1. **Wrong / missing `ClientGUID`** → poll returns 204 forever. Confirm your agent logs this distinctly from "no scan pending", or you will chase it for a day.
2. **Neither identity** — `GET V1/GetDiscoveryScanJob` with no query at all → **400**. Same with `?endpointUserId=2` alone — it is not an identity.
3. **Multi-user machine** — on a laptop with two signed-in accounts, poll with `serialNumber` + `endpointUserId` → the response echoes the `endpointUserId` you sent; poll without it → you get the primary user's.
4. **Claim and resume** — poll once (200, note `endpointScanId`), poll again immediately → **same** `endpointScanId`, still Running.
5. **Happy path** — full scan on a pilot machine → rows appear in the dashboard's endpoint detail; `filesScanned` matches your own count.
6. **Incremental with no changes** — re-run right after → 0 new/modified rows, scan completes, cursor advances.
7. **Change detection** — modify 3 files, delete 1 → incremental sends exactly 3 upserts + 1 `Deleted`, and `processed` is 4.
8. **Batch cap** — send 501 files → **400** with "A batch carries at most 500 files."; confirm your batching never generates it.
9. **Missing `endpointId`** — send a progress body without it → **400**, not a silent success.
10. **Wrong `endpointId`** — send another machine's id → **404**.
11. **Interruption** — kill the agent mid-scan → server shows Interrupted after 15 min → restart → poll returns the same scan → resume from the last acked batch, and completion counters are still correct.
12. **Fatal error path** — force a failure → you still call complete with `errorDetail` → scan shows Failed with the reason, not Interrupted.
13. **External volume removed** — complete a scan omitting a previously-reported drive → it disappears from the dashboard; plug it back in and the next scan runs full for it.

---

## 7. Deployment notes

- The four routes are **new in the DLP API** and need APIM operations registered on `/qadlp` before they are reachable: `GET V1/GetDiscoveryScanJob`, `POST V1/DiscoveryScanProgress`, `POST V1/DiscoveryFilesBatch`, `POST V1/DiscoveryScanComplete`. A gateway `404 {"statusCode":404}` — note the bare shape, not our envelope — means the operation isn't registered yet, **not** a wrong URL.
- The discovery tables (`discovery."ScanJob"`, `"EndpointScan"`, `"Volume"`, `"File"`) come from `scripts/create_discovery_schema.sql` and must exist in the tenant database.
- `endpointUserId` needs one extra column on an existing database: run `scripts/add_discovery_endpointuserid_column.sql` once per tenant DB. It is `ADD COLUMN IF NOT EXISTS`, so it is safe to re-run, and the column is nullable — agents that do not send `endpointUserId` keep working exactly as before.
- The ExeApi routes stay exactly as they are. Nothing about this change requires you to move.
- The QA database currently holds mock discovery data (marker `RequestedBy = -1`) so the dashboard works before your integration. We'll clean it out when your pilot starts.
