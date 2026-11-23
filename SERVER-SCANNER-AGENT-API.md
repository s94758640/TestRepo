# Server Scanner — EXE ⇄ API contract

Everything needed to build the server scanner executable. The API side is
implemented and deployed from this repository; this document is the contract
the EXE must speak.

---

## 1. What this is, and how it differs from the endpoint agent

A standalone executable that an operator runs **on a server** (file server, NAS
gateway, application server). It walks the folders it is pointed at,
classifies what it finds, and pushes metadata and verdicts back to the DLP API.

**There is no trigger.** Nothing in the console starts a server scan and nothing
is queued waiting for one. This is the single structural difference from the
endpoint (DDC) agent, and almost every other difference follows from it:

| | Endpoint agent (DDC) | **Server scanner (this)** |
|---|---|---|
| What starts a scan | Console queues a job; agent polls `GET V1/GetDiscoveryScanJob` | **A person runs the EXE** |
| Opening a run | `GET` claims a pre-made row | **`POST V1/ServerScanStart` creates it** |
| Identity | `sfm."Endpoint"` row, created by enrolment | **`discovery."ServerAgent"`, created by the EXE on first register** |
| Scan unit | Whole drives, keyed by letter | **Named roots — `D:\Shares\Finance`, `\\nas01\archive`** |
| Stopped how | Console cancels the job | **Console sets a flag; the EXE reads it and stops itself** |
| Swept stale after | 8 hours | **30 minutes** (a server that goes quiet is a dead process, not a closed lid) |

**The EXE drives the whole exchange:**

```
POST V1/ServerScanRegister     → ServerAgentId + settings        (always first)
POST V1/ServerScanStart        → ServerScanId + resume state
POST V1/ServerScanProgress     ← cancelRequested                 (every ~60 s)
POST V1/ServerScanFilesBatch   ← ack + cancelRequested           (≤500 files)
POST V1/ServerScanComplete     → final status                    (always last)
```

### What must never leave the server

File **contents**, matched **keywords** and **samples**. These routes carry
metadata and classification verdicts only. The one field that does carry
detection detail — `sensitiveContent` — is masked by the API before it is
stored, but it is on the wire until then, so send the least you can.

---

## 2. Base URL and headers

```
Base:  https://qa-email.azure-api.net/qadlp          (QA; production differs)

Ocp-Apim-Subscription-Key: <APIM key>        hosted only; absent on-prem
ClientGUID:                <tenant GUID>     REQUIRED on every call
Content-Type:              application/json  on every call
```

**Miss `ClientGUID` and you land on the default database, not the tenant's.**
Register will appear to succeed against the wrong tenant and nothing will ever
show up in the customer's console. Log this header's value at startup — it is
the single most likely cause of "it ran fine but the console is empty".

### Authentication

These are agent routes and are **anonymous**, like `GetEndpointExeSettings` and
the other agent routes on this host. What protects them is the APIM subscription
key, the `ClientGUID` header, and the server-side rule that the
`serverAgentId` in your body must own the scan you are writing to.

Where that is not enough — servers scanned by hand on an untrusted network — the
tenant sets the `ServerScannerEnrollmentToken` ExeSettings row, and
`ServerScanRegister` starts requiring `enrollmentToken` in the body. The EXE
should accept that value as a command-line argument or config entry and send it
whenever it has one; sending it when the tenant has not set one is harmless and
ignored.

---

## 3. Identity: `machineKey`

The EXE computes **one stable string per server** and sends it on every
register. Get this wrong and the inventory grows by one server per run.

Recommended, in order:

1. The BIOS/system UUID — `(Get-CimInstance Win32_ComputerSystemProduct).UUID`.
2. If that is absent, all-zeroes or the known-bad `FFFFFFFF-FFFF-FFFF-FFFF-FFFFFFFFFFFF`:
   a GUID generated **once** and written beside the executable (or to
   `HKLM\SOFTWARE\<vendor>\ServerScanner\MachineKey`), then reused for ever.

**Do not use the hostname.** Servers get renamed, and two servers in different
domains share short names — either one silently splits or merges a server's
whole history.

Max 128 characters.

---

## 4. The five calls

### 4.1 `POST V1/ServerScanRegister`

Announces the server and returns its id and the rules to scan by. Idempotent —
the first call creates, every later one updates. **Always the EXE's first call**,
including on a resume.

```jsonc
{
  "machineKey":      "4C4C4544-0037-5A10-8044-C7C04F485A33",  // REQUIRED
  "hostName":        "FS01",                                   // REQUIRED
  "fqdn":            "fs01.corp.local",
  "ipAddress":       "10.12.4.21",
  "osVersion":       "Microsoft Windows Server 2022 Standard (10.0.20348)",
  "agentVersion":    "1.0.3",
  "timeZone":        "India Standard Time",
  "description":     "Finance file server",      // free text, from the command line
  "initiatedBy":     "CORP\\svc_dlpscan",        // the OS account running the EXE
  "enrollmentToken": "…"                         // only when the tenant set one
}
```

**200:**

```jsonc
{
  "success": true,
  "data": {
    "serverAgentId": 7,
    "isNew": false,
    "scannerEnabled": true,          // ← CHECK THIS
    "settings": {
      "batchSize": 500,
      "heartbeatSeconds": 60,
      "interruptedAfterMinutes": 30,
      "maxFileSizeBytes": 524288000,
      "excludedPaths": ["\\Windows\\", "\\$Recycle.Bin\\"],
      "fileTypes": [
        { "fileTypeMasterId": 3, "fileType": "pdf", "fileTypeDescription": "PDF document" },
        { "fileTypeMasterId": 4, "fileType": ".xlsx", "fileTypeDescription": "Excel workbook" }
      ]
    }
  }
}
```

- **`scannerEnabled: false`** — the tenant has switched server scanning off.
  Exit cleanly without scanning; `ServerScanStart` would refuse anyway.
- **`fileTypes`** — the extensions this tenant classifies. Compare
  case-insensitively and tolerate the leading dot being absent: the master holds
  both spellings, as the example shows.
- **`maxFileSizeBytes`** — files above this are **skipped, not failed**. Count
  them in `filesSkipped`.
- **`excludedPaths`** — advisory directory fragments. The EXE applies them; the
  API has never seen the filesystem.

**401** — the enrolment token is missing or wrong. **400** — `machineKey` or
`hostName` missing.

---

### 4.2 `POST V1/ServerScanStart`

Opens a run. **Write `scanKey` to your journal before you call this.**

```jsonc
{
  "serverAgentId": 7,                                     // REQUIRED, from register
  "machineKey":    "4C4C4544-0037-5A10-8044-C7C04F485A33",// REQUIRED, must match
  "scanKey":       "9f3c1a84-6d2e-4f77-9b51-2c0a7e5d1b33",// REQUIRED, your GUID
  "scanType":      "Incremental",                         // Full | Incremental
  "rootPaths":     ["D:\\Shares\\Finance", "\\\\nas01\\archive"],
  "initiatedBy":   "CORP\\svc_dlpscan",
  "agentVersion":  "1.0.3"
}
```

**200:**

```jsonc
{
  "success": true,
  "data": {
    "serverScanId": 41,
    "serverAgentId": 7,
    "resumed": false,
    "lastAckedBatchSeq": 0,
    "cursors": [
      { "serverPathId": 12, "rootPath": "D:\\Shares\\Finance", "cursor": "ts:2026-11-04T18:22:05Z", "lastScanAt": "2026-11-04T18:40:11Z" },
      { "serverPathId": 13, "rootPath": "\\\\nas01\\archive",  "cursor": null, "lastScanAt": null }
    ],
    "settings": { /* same shape as register, re-read in case it changed */ }
  }
}
```

**Idempotency is the point of `scanKey`.** If the response is lost and you
resend the same key, you get the **same `serverScanId`** back with
`"resumed": true` and the true `lastAckedBatchSeq`. Resume from
`lastAckedBatchSeq + 1`; do not restart the walk.

**Cursors.** `cursor` is whatever string you sent for that root last time —
the API stores it opaquely and never parses it. Use an ISO timestamp watermark
(`ts:2026-11-04T18:22:05Z`), a USN position, anything you can act on. A root
absent from the list, or carrying `null`, **must be walked full** regardless of
the `scanType` you asked for.

A root you discover mid-run does not need to be declared here — the first batch
mentioning it creates it, and it simply scans full that once.

**409** — either the tenant has the scanner switched off (`outcome:
"ScannerDisabled"` — exit), or that `scanKey` belongs to a run that already
finished (`outcome: "AlreadyFinished"` — generate a new key).
**403** — `serverAgentId` and `machineKey` do not agree. Re-register.

---

### 4.3 `POST V1/ServerScanProgress`

Heartbeat and running counters. Call at least every `heartbeatSeconds`.

```jsonc
{
  "serverScanId":  41,      // REQUIRED
  "serverAgentId": 7,       // REQUIRED — verified against the scan row
  "filesScanned":  120345,  // running totals for the whole run, NOT deltas
  "sensitiveFound": 412,
  "filesSkipped":  88,
  "bytesScanned":  91234567890,
  "currentPath":   "D:\\Shares\\Finance\\Payroll\\2026"
}
```

**200:**

```jsonc
{ "success": true, "data": { "serverScanId": 41, "cancelRequested": false, "status": "Running" } }
```

**Read `cancelRequested` on every response.** True means somebody asked the
console to stop this run, or the tenant switched the scanner off mid-run. Stop
at the next batch boundary and call `ServerScanComplete` with
`"cancelled": true`.

Going quiet for `interruptedAfterMinutes` (30) marks the run `Interrupted`.
That is **recoverable, not fatal**: start again with the same `scanKey` and carry
on from `lastAckedBatchSeq`.

---

### 4.4 `POST V1/ServerScanFilesBatch`

Findings, at most 500 per call. **Sensitive and unclassified files only** —
clean files are never sent as rows, they ride the counters.

```jsonc
{
  "serverScanId":  41,
  "serverAgentId": 7,
  "batchSeq":      17,          // 1-based, strictly increasing within a run
  "files": [
    {
      "rootPath":  "D:\\Shares\\Finance",              // REQUIRED, as declared at start
      "directory": "D:\\Shares\\Finance\\Payroll\\2026",// REQUIRED, the file's own folder
      "fileName":  "March-salaries.xlsx",
      "extension": "xlsx",
      "sizeBytes": 184320,
      "lastModified": "2026-03-31T09:12:44Z",
      "owner":     "CORP\\jsmith",
      "classifications": ["PII", "PFI"],
      "sensitiveContent": "{\"PAN\":[\"ABCDE1234F\"],\"Email\":[\"j@corp.com\"]}",
      "confidence": 92.5,
      "risk":       "HIGH",
      "severity":   "CRITICAL",
      "severityReason": "PAN + Email + Bank account",
      "dpdpFlag":   "Protect",
      "changeKind": "New"                              // New | Modified | Deleted
    }
  ]
}
```

**200:**

```jsonc
{
  "success": true,
  "data": { "serverScanId": 41, "batchSeq": 17, "processed": 500,
            "lastAckedBatchSeq": 17, "cancelRequested": false }
}
```

Field notes:

- **`rootPath` vs `directory`** — `rootPath` is the tree you were pointed at;
  `directory` is where the file actually sits. Both are required and they are
  usually different.
- **`owner`** — send it whenever the ACL can be read. On a share this is the
  only thing that says whose data a finding is; without it a thousand sensitive
  files have nobody to send them to. This is the field the server scanner exists
  for and the endpoint agent has no equivalent of.
- **`classifications`** — empty array means *scanned but unreadable or
  unsupported* (the Unclassified bucket). That is a legitimate, useful result —
  send the row. Labels are stored as class codes whatever form you send.
- **`sensitiveContent`** — a JSON **object** as a string. The API masks it
  before storing, so `4111 2222 3333 4444` lands as `#### 2### 3### 4###`.
  Anything that is not a JSON object is stored as a placeholder recording only
  its length. Optional.
- **`risk` / `severity` / `severityReason` / `dpdpFlag`** — stored **verbatim**,
  validated against nothing. Omit them and the columns stay null, which the
  console reads as "not recorded" and never as a benign verdict.
  `severityReason` is **not masked** — name the classes that fired, never the
  matched values.
- **`changeKind: "Deleted"`** marks an existing row gone. Only meaningful on an
  incremental run; a full run sends `New`/`Modified`. A deletion for a file we
  never had is accepted and not counted.

**Writes are upserts** keyed on the file's path, so replaying an acked
`batchSeq` is harmless and resuming never duplicates a row. A batch also counts
as a heartbeat — a scanner uploading steadily will not be swept.

**400** if the batch carries more than 500 files.

---

### 4.5 `POST V1/ServerScanComplete`

**Always the EXE's last call** — including after a fatal error and including
after a cancellation.

```jsonc
{
  "serverScanId":  41,
  "serverAgentId": 7,
  "executedType":  "Incremental",   // what actually ran
  "filesScanned":  482001,
  "sensitiveFound": 1904,
  "filesSkipped":  312,
  "bytesScanned":  912345678901,
  "errorDetail":   null,            // set → the run records Failed
  "cancelled":     false,           // true → records Cancelled
  "paths": [
    {
      "rootPath":  "D:\\Shares\\Finance",
      "kind":      "Local",                 // Local | Network | Removable
      "fileSystem": "NTFS",
      "sizeGb":    4096.00,
      "freeGb":    812.50,
      "encryptionStatus": "Encrypted",      // Encrypted|Partial|Suspended|Unencrypted|Unknown
      "encryptionMethod": "XTS-AES 256",
      "cursor":    "ts:2026-11-11T04:55:02Z",
      "filesScanned": 301422,
      "sensitiveCount": 1502,
      "unclassifiedCount": 88
    }
  ]
}
```

**200:**

```jsonc
{ "success": true,
  "data": { "serverScanId": 41, "status": "Completed", "durationSeconds": 5321, "storedFileCount": 1992 } }
```

- **`status`** is `Completed`, `Failed` (when `errorDetail` is set) or
  `Cancelled` (when `cancelled` is true).
- **`storedFileCount`** is what the API now holds for this server. A cheap check
  that what you think you sent is what landed.
- **`encryptionStatus`** — send **`Unknown`**, never `Unencrypted`, when the
  state could not be read. A scanner without local administrator rights cannot
  query BitLocker, and this feeds a compliance figure: "nobody checked" must not
  read as "confirmed plaintext".
- **`cursor`** — `null` leaves the stored cursor alone (a run that could not
  produce one must not wipe a good marker). An **empty string** is the explicit
  "force the next run to be full".
- **Roots you omit are marked gone** — but only if you reported at least one. A
  run that died before reaching any root reports `"paths": []`, and the server's
  inventory and cursors are left intact rather than being blanked.

**Do not just exit on an error.** A run left open sits `Running` until the sweep,
and the console spends half an hour reporting a scan that is not happening. Call
this with `errorDetail`.

---

## 5. Failures

Every refusal carries a machine-readable reason in `data`:

```jsonc
{ "success": false,
  "message": "That scan is already Completed. Start a new run.",
  "data": { "outcome": "AlreadyFinished", "retryable": true } }
```

| HTTP | `outcome` | Meaning | What the EXE does |
|---|---|---|---|
| 400 | `Invalid` | Malformed request | Fix the payload. Do not retry as-is |
| 401 | `NotAuthorised` | Enrolment token missing/wrong | Stop; this is a config problem |
| 403 | `WrongAgent` | Ids do not identify your server | Re-register, then start again |
| 404 | `NotFound` | No such scan | Re-register and start a new run |
| 409 | `AlreadyFinished` | That run is over | New `scanKey`, new run |
| 409 | `ScannerDisabled` | Tenant switched the scanner off | Exit cleanly |
| 500 | — | Server fault | Retry with backoff; the writes are idempotent |

`retryable` is the one bit worth branching on: true means re-register, re-start
and resend is the right move.

**Transient network failures** (timeouts, 502/503/504) — retry with exponential
backoff. Every write on these routes is idempotent, so a retried batch cannot
duplicate a row and a retried start cannot open a second run.

---

## 6. Reference flow

```text
settings = REGISTER(machineKey, hostName, …)
if not settings.scannerEnabled: exit 0

scanKey = journal.readPending() ?? newGuid()      // write BEFORE starting
journal.writePending(scanKey)

run = START(serverAgentId, machineKey, scanKey, scanType, rootPaths)
seq = run.lastAckedBatchSeq                       // 0 on a fresh run

batch = []
lastBeat = now()

for root in rootPaths:
    cursor = run.cursors[root]?.cursor            // null ⇒ walk this root FULL
    for file in walk(root, cursor, settings.excludedPaths):

        if file.size > settings.maxFileSizeBytes:
            skipped++; continue                   // SKIPPED, not failed

        verdict = classify(file)                  // local; contents never leave
        scanned++
        if verdict.isClean: continue              // clean files ride the counters

        batch.append(toItem(root, file, verdict))

        if batch.size == settings.batchSize:
            ack = FILES_BATCH(run.id, agentId, ++seq, batch)
            journal.writeAcked(ack.lastAckedBatchSeq)
            batch.clear()
            if ack.cancelRequested: goto stopping

        if now() - lastBeat > settings.heartbeatSeconds:
            ctl = PROGRESS(run.id, agentId, scanned, sensitive, skipped, bytes, file.dir)
            lastBeat = now()
            if ctl.cancelRequested: goto stopping

if batch.notEmpty():
    FILES_BATCH(run.id, agentId, ++seq, batch)    // the final partial batch

COMPLETE(run.id, agentId, executedType, counters, cancelled: false, paths)
journal.clearPending()
exit 0

stopping:
    if batch.notEmpty(): FILES_BATCH(run.id, agentId, ++seq, batch)
    COMPLETE(run.id, agentId, executedType, counters, cancelled: true, paths)
    journal.clearPending()
    exit 0

on fatal error e:
    COMPLETE(run.id, agentId, executedType, counters, errorDetail: e.toString(), paths)
    journal.clearPending()
    exit 1
```

### The EXE's local journal

Two values, written to disk beside the executable:

| Key | Written when | Read when |
|---|---|---|
| `pendingScanKey` | before `ServerScanStart` | on startup — a value here means resume |
| `machineKey` | on first ever run | every run |

You do **not** need to journal acked batch sequences: `ServerScanStart` returns
`lastAckedBatchSeq` from the server, which is authoritative and survives the
machine going down with its journal.

---

## 7. Tenant settings the EXE is affected by

All are rows in the tenant's `dlp."ExeSettings"`. Every one is absent by default
and every default is permissive.

| Key | Default | Effect |
|---|---|---|
| `ServerScannerEnabled` | on | `"0"`/`"false"` stops server scanning tenant-wide. Register returns `scannerEnabled: false`, start returns 409, and a run in flight gets `cancelRequested` on its next heartbeat |
| `ServerScannerEnrollmentToken` | unset | When set, `ServerScanRegister` requires a matching `enrollmentToken` |
| `ServerScannerExcludedPaths` | empty | Comma/semicolon separated directory fragments, handed to the EXE to apply |
| `ServerScannerHeartbeatSeconds` | 60 | Heartbeat cadence |
| `ServerScannerInterruptedAfterMinutes` | 30 | Silence before a run is swept `Interrupted` |

`Discovery:MaxFileSizeBytes` (app configuration, not ExeSettings) sets
`maxFileSizeBytes`, defaulting to 500 MB. It is shared with endpoint discovery
on purpose — one answer per tenant to "how big is too big".

---

## 8. Run states

```
                      ┌──────────────── start (same scanKey) ──────────┐
                      │                                                │
   START ──→ Running ─┼─→ COMPLETE (no error)          ──→ Completed   │
                      ├─→ COMPLETE (errorDetail)       ──→ Failed      │
                      ├─→ COMPLETE (cancelled: true)   ──→ Cancelled   │
                      └─→ 30 min of silence            ──→ Interrupted ┘
```

`Completed`, `Failed` and `Cancelled` are terminal — writes against them return
409. `Interrupted` is **not** terminal: writes are still accepted and the first
one takes the run back to `Running`, which is what makes a crashed scanner
resumable.

---

## 9. Server-side reference

| Piece | Where |
|---|---|
| Agent routes | `SFM.DLP.API/Controllers/ServerScannerAgentController.cs` |
| Agent logic | `SFM.DLP.Application/Services/ServerScannerAgentService.cs` |
| Wire contracts | `SFM.DLP.Application/DTOs/ServerScannerAgentDTO.cs` |
| Tenant switches | `SFM.DLP.Application/Services/ServerScannerSettings.cs` |
| Console routes | `SFM.DLP.API/Controllers/ServerScannerController.cs` |
| Tables | `SFM.DLP.API/Migrations/Dlp/V0028__server_scanner.sql` |
| Tests | `SFM.Platform.Storage.Tests/ServerScannerTests.cs` |

Console-side routes, for whoever builds the UI — all JWT-authenticated, all
using the house `{ TotalCount, Data }` grid envelope:

```
POST server-scanner/servers             grid of scanned servers
POST server-scanner/scans               grid of runs (optionally one server)
POST server-scanner/paths               roots scanned on one server
POST server-scanner/files               the findings grid
POST server-scanner/overview            tiles + breakdowns
POST server-scanner/scans/cancel        ask a run to stop
POST server-scanner/servers/set-active  retire / restore a server
```

There is deliberately **no start endpoint**. The console cannot launch a process
on a customer's server. If a "Scan now" button is ever wanted, the EXE has to be
installed as a polling service — a different feature, and one that should be
built as one rather than bolted onto this.
