# Data Discovery & Classification — EXE Agent Integration Guide

**Audience:** EXE agent team
**Server side:** live in code (SFM.DLP.ExeApi, same function app as your existing `V1/...` routes); reachable in QA after deploy + APIM registration
**Contracts in code:** `SFM.DLP.ExeApi/DTO/DiscoveryDTO.cs` · design: `DATA-DISCOVERY-IMPLEMENTATION-PLAN.md`

---

## 1. What the agent has to do (new capability)

Data-at-rest discovery: on request, walk the machine's disks, classify file contents with the **same DLP content engine you already run for transfers**, and report results in batches. You do **not** upload file content, matched keywords, or samples — metadata + class labels only.

The whole flow rides your existing patterns: you already poll `V1/GetEndPointCommand`; discovery adds one more poll target and three upload calls, same base URL, same agent auth (bearer token via the existing auth middleware).

## 2. The four endpoints (flow order)

### 2.1 `GET V1/GetDiscoveryScanJob?serialNumber=<serial>` — or `?endpointId=<id>`

Poll on your normal command cadence. **One identity is enough**:
- `serialNumber` — preferred; the hardware serial you already send on every other call.
- `endpointId` — fallback for machines without a usable BIOS serial (VMs, clones, "To be filled by O.E.M."); you know it after registration and from every previous poll response.
- Sending both is allowed (serial is tried first); sending neither is a **400**.

Responses:
- **204** — nothing to do (the common case).
- **200** — a scan to run:

```json
{
  "success": true,
  "data": {
    "endpointScanId": 5012,
    "scanJobId": 12,
    "endpointId": 37,
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
- The first read of a Queued scan **claims it** (server marks it Running). If you poll again — e.g. after a reboot — you get the **same** `endpointScanId` back: resume it, don't restart from zero.
- `scanType` is what was requested. Per volume, if `usnCursor`/`snapshotVersion` is null (or your USN journal wrapped / the volume isn't NTFS), **upgrade that volume to full** and later report `executedType` accordingly.
- `volumes` is the list of **fixed drive letters**, and may list letters the machine doesn't have — skip them silently.
- **`includeMapped` / `includeExternal` are how the non-fixed volumes are requested.** A mapped share or a USB stick has no letter anybody can name when the job is queued — it depends on who is signed in and what is plugged in — so these are flags you resolve **on the machine at scan time**, not entries in `volumes`:
  - `includeMapped: true` → also enumerate and scan **mapped network drives** (`DriveType` 4), whatever letters they have, and report each one in `DiscoveryScanComplete` with `"kind": "Mapped"`.
  - `includeExternal: true` → also enumerate and scan **removable / external volumes** (`DriveType` 2 and external fixed disks — USB sticks, portable drives), reported with `"kind": "External"`.
  - Either flag false → leave that whole class of volume alone, even if one is mounted on a letter that appears in `volumes`.
  - These flags are what fill the console's **Mapped** and **External** columns. They read 0 for as long as the agent ignores them.
- For backward compatibility `volumes` still also carries the tokens `"Mapped"` / `"External"` whenever the matching flag is set. Read the **flags**; treat any entry in `volumes` that is not a drive letter as noise.
- `fileTypes` is every **active** row of the tenant's file-type master — the types to classify. Same shape `GetAllFileTypeMaster` returns, and it ships with the job so a scan needs no second call before it starts. Match `fileType` case-insensitively and tolerate a missing leading dot (`pdf` and `.pdf` both occur). Retired types are omitted, so treat it as the whole allowed set and re-read it each poll rather than caching across runs.
- `maxFileSizeBytes` is the size ceiling, in **bytes** — **skip** anything larger; do not fail it or report it as an error. Server-set like `batchSize`, so read it rather than hard-coding. Defaults to 500 MB (`524288000`).

### 2.2 `POST V1/DiscoveryScanProgress` — the heartbeat

At least every `heartbeatSeconds` (300 s) while scanning:

```json
{ "endpointScanId": 5012, "endpointId": 37, "filesScanned": 18230, "sensitiveFound": 411, "currentVolume": "C" }
```

Miss it for 15 minutes and the server marks the scan **Interrupted**. That is not fatal: your next `GetDiscoveryScanJob` poll returns the **same scan again** (flipped back to Running) — resume from your last acked batch. An Interrupted scan only stops being resumable when a newer scheduled job supersedes it.

**`endpointId` is REQUIRED on all three POST bodies** (progress, batch, complete) — use the value from `GetDiscoveryScanJob`. It is how the auth layer scopes your agent token to your own machine: the middleware compares it to the token, and the server verifies it matches the scan. Omitting it = **400**; a wrong value = **404/403**.

### 2.3 `POST V1/DiscoveryFilesBatch` — results, ≤500 per call

Send **only sensitive and unclassified** files (decision 8.1). Clean files are never sent as rows — they ride the counters.

```json
{
  "endpointScanId": 5012,
  "endpointId": 37,
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

- `classifications`: your engine's DlpType labels (PHI/PII/PFI/PCI/CODE/CREDS…). **Empty array = unclassified** (file was scanned but unreadable/unsupported) — send those too.
- `sensitiveContent` (optional): what tripped detection, as a JSON **object**, same shape you send on a transfer. **Masked server-side before it is stored** — `4111 2222 3333 4444` becomes `#### 2### 3### 4###`. One-way, with no setting that turns it off, so don't send it expecting to read it back. Non-object payloads are stored as a length-only placeholder; omit it and the detail panel is empty.
- `risk`, `severity`, `dpdpFlag`, `severityReason` (all optional): your verdict for the file, stored **exactly as you send it** — the server validates and rewrites nothing, so a label outside the usual vocabulary reaches the grid rather than being dropped or normalised into one that looks official. `severityReason` is free text ("Health insurance + Email + Phone"): name the **classes** that fired, never the matched values, because unlike `sensitiveContent` this field is **not** run through the masker — it is stored and displayed as sent.
- Omitting any of these four stores `NULL`, which the grid reads as "not recorded" and renders as an empty chip. `NULL` is never treated as a benign verdict, so there is no harm in sending nothing until your build computes them.
- `changeKind`: `New` | `Modified` | `Deleted`. Incremental scans send `Deleted` for files that disappeared; full scans send New/Modified only.
- `batchSeq`: 1-based, strictly increasing per scan. Keep a local journal of acked batches; after a crash, resume from the first un-acked one. (Server-side replay dedup is not enforced yet — re-sending an acked batch is harmless because writes are upserts, but don't rely on it for counting.)
- Response echoes `{ endpointScanId, batchSeq, processed }`.

### 2.4 `POST V1/DiscoveryScanComplete` — always the last call

```json
{
  "endpointScanId": 5012,
  "endpointId": 37,
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

- `executedType` = what actually ran (Full when you upgraded).
- **`usnCursor`** = the new journal position after this scan — the server hands it back on the next poll; it is your incremental state. Volumes scanned by inventory-diff report `snapshotVersion` instead.
- **Encryption**: read BitLocker per volume via WMI `Win32_EncryptableVolume` at scan end. `encryptionStatus`: `Encrypted` | `Partial` | `Suspended` | `Unencrypted`; include `encryptionMethod` (e.g. `XTS-AES 256`) when protected and `suspendedSince` when protection is paused.
- A volume you scanned last time but that is now absent (unplugged external, unmapped share): just omit it — the server deactivates it.
- Fatal failure: still call complete, with `errorDetail` set — the scan is recorded Failed instead of hanging until the Interrupted sweep.
- `filesScanned` counts **everything** you enumerated, clean files included — this is where clean files are accounted for.

### 2.5 Error handling

- **404** on progress/batch/complete = the scan is not Running right now (superseded by a newer job, or swept Interrupted while you were away). Abandon the call and re-poll `GetDiscoveryScanJob`: if the scan was merely Interrupted you get the **same** `endpointScanId` back and resume; if it was superseded you get the newer job (or 204).
- **400** = contract violation (missing id, >500 files) — a bug to fix, not to retry.

## 3. Behaviour rules (agreed product decisions)

| Rule | Decision |
|---|---|
| Files to classify | **Identical to the transfer engine** — same file-type list, same zip handling, same max-size guard. Everything else counts as clean without content reading (8.7) |
| Rows to send | Sensitive + unclassified only; clean = counters (8.1) |
| Throttling | Low-priority I/O + configurable CPU / files-per-second cap; scan runs while the user works — the machine must stay responsive (8.8) |
| Offline at scan time | Nothing special to do: the job stays claimable until the next scheduled run supersedes it — you pick it up on the next poll after boot (8.6) |
| First scan of a volume | Always full (nothing to be incremental against) |
| One scan at a time | Never run two discovery scans concurrently on a machine |

## 4. Suggested rollout

1. Build against QA (after APIM ops are registered — a gateway `404 {"statusCode":404}` means the operation isn't registered yet, not a wrong URL).
2. Pilot on 2–3 endpoints: full scan → verify rows in the dashboard's endpoint detail → immediate incremental with no changes → expect 0 new/modified.
3. Change-detection test: modify 3 files, delete 1 → incremental sends exactly 3 upserts + 1 Deleted.
4. Kill the agent mid-scan → server shows Interrupted after 15 min → restart agent → poll returns the same scan → resume from last acked batch.
5. We then run the P4 load test together before rolling to all endpoints.

**Note:** the QA database currently holds mock discovery data (marker `RequestedBy = -1`) so the dashboard works before your integration. We'll clean it out when your pilot starts.
