# Cloud discovery — API integration guide

For the UI team building the **Cloud** screens (Integrations and Exposure tabs)
and the **Connect a cloud app** wizard.

Everything here is live on DLP.API. v1 covers **SharePoint and OneDrive**, which
are one Microsoft Graph integration. Google Drive, Box and Dropbox are not
implemented yet — their tiles should render as "available to add".

---

## Every element on the screens, and whether it can be served

Read this before building anything. Some of what the designs show cannot be
produced from the data any cloud provider gives us, and binding a component to
one of those means it renders empty for ever — which a customer reads as *zero*,
not as *unknown*.

### Exposure tab

| Element | | Comes from |
|---|---|---|
| Blast Radius matrix | ✅ | `cloud/overview` → `blastRadius` |
| Exposure of Sensitive Objects | ✅ | `cloud/overview` → totals per level |
| Where Sensitive Data Sits | ✅ | `cloud/overview` → `hotspots` (top 10); the "All N" link → `cloud/locations` |
| Service Board | ✅ | `cloud/services` |
| Public Links in Circulation (preview) | ✅ | `cloud/links`, except the open counts |
| Just Scanned feed | ⚠️ | approximable from `cloud/files` sorted by `firstSeenAt` — but it is "recently **discovered**", not "scanned as it lands" |
| **LIVE banner** — Uploads scanned today, Flagged sensitive, Time to verdict, 24-hour histogram | ❌ | needs change webhooks; we poll and crawl |
| **Restricted on detection** — "83 sharing revoked" | ❌ | remediation write-back; needs write permissions no customer has consented to |

### Public Links tab

| Element | | Comes from |
|---|---|---|
| Public links · with sensitive content | ✅ | `summary.publicLinksWithSensitiveContent` |
| Outside companies | ✅ | `summary.outsideCompanies` |
| **Anonymous opens** | ❌ | each provider's audit feed — see below |
| **Places shared** — channels and threads | ❌ | collaboration-tool integration — see below |
| Table · File, Owner, Sensitive data, Source | ✅ | `cloud/links` row |
| Table · Who can open it — **the badge** | ✅ | `scope` + `externalIdentityCount` |
| Table · Who can open it — *"96 opened without signing in"* | ❌ | audit feed |
| Table · Link created by | ❌ | always empty — `roles` works, `createdByName` and `ageDays` do not; use `seenDays` |
| **Table · Destination** — Slack / Teams / Email / Wiki | ❌ | collaboration-tool integration |
| Panel · file, owner, classes, link URL, "Public for 41 days" | ✅ | `cloud/links` row |
| Panel · Employees / Outsiders / companies | ✅ | `internalIdentityCount`, `externalIdentityCount`, `externalDomains` |
| Panel · **Anyone — 96 opens** | ❌ | audit feed |
| Panel · **Where the link has been shared** (whole section) | ❌ | collaboration-tool integration |

### Why the ❌ rows are not simply unbuilt

**Not "not implemented" — not collected.** Two different pieces of work sit
behind them, and neither is a setting.

**Open counts** exist, but in a *different API per provider*: Microsoft Purview,
Google's Drive Activity API, Box enterprise events, Dropbox's team log. Each
returns a stream of events rather than a number, needs its own consent, and
needs somewhere to store and aggregate those events. Roughly 1–2 weeks per
provider. Microsoft alone would cover SharePoint and OneDrive.

**Link spread** — *Destination*, *Places shared*, *Where the link has been
shared* — is not in any storage provider at all. Google does not know a Drive
link was pasted into Slack; Slack knows. Producing it means integrating Slack,
Teams, Confluence and mail, reading message content, extracting URLs and
matching them back to files. That is a separate product, and a privacy
conversation.

**What to do with them meanwhile:** hide, or label as coming later. Do not render
them as `0`. "0 anonymous opens" reads as *nobody has opened it*, and we do not
know that either way.

The badge — *Anyone / Outsiders / Org-wide* — is the part that needs a decision
from whoever reads the screen. The open count says how bad it already is; it
does not change what you do about it.

---

## The basics

**Host:** DLP.API (`qadlp` through APIM).
**Auth:** `Authorization: Bearer <web app jwt>` on every call, plus the usual
`ClientGUID` header — that header selects the tenant database, so no tenant id
appears in any payload.
**Verb:** POST for everything except the two GETs noted below, including reads.
That is this API's house style, not an oversight.
**Envelope:** every response is `{ success, message, data, errors }`. All the
shapes below are what sits in `data`.

**Admin-only routes** — these return **403** for a non-admin, so hide the
controls rather than letting the call fail: create/test/delete a connector,
queue a scan, and change what is excluded.

---

## 1 · Integrations tab

### Grid operations — the same contract as every other grid

`POST /cloud/connectors/list` and `POST /cloud/services` take the shared
`QueryParameters` body — the identical shape `GetUserReport` and `GetTransferLogs`
take, so the console needs no cloud-specific grid code:

```jsonc
POST /cloud/connectors/list
{
  "page": 1,
  "pageSize": 20,
  "sortBy": "Status",
  "sortDescending": true,
  "likeFilters":  { "Provider": ["box"] },
  "exactFilters": { "Status": ["Connected", "Failed"] }
}
```

```jsonc
{ "totalCount": 5, "data": [ … ] }     // same envelope GetUserReport returns
```

- **Filter and sort keys are the response's own field names**, matched
  case-insensitively — `Provider`, `DisplayName`, `Status`, `FileCount`,
  `ContainerCount`, `LastSyncAt`, `DateAdded` for connectors; plus `TotalFiles`,
  `SensitiveFiles`, `PublicLinkFiles` for services.
- `likeFilters` is CONTAINS, `exactFilters` is equality; several values on one
  key are OR'ed.
- **A key that names no field is ignored, not an error** — an out-of-date UI
  cannot break the grid.
- `pageSize: 0` returns everything unpaged. `totalCount` is always the count
  *before* paging.

`GET /cloud/connectors` is unchanged and still returns the plain unpaged array —
it is what the connect wizard and the provider pickers use. The grid uses
`/list`, because `POST /cloud/connectors` already creates a connector.

### The service list

```
GET /cloud/connectors
```

```jsonc
[
  {
    "cloudConnectorId": 2,
    "provider": "SharePoint",            // SharePoint | OneDrive
    "displayName": "Innovly SharePoint",
    "externalTenantId": "…", "clientId": "…",
    "status": "Connected",               // Pending | Connected | Failed | Disabled
    "lastError": null,
    "lastValidatedAt": "2026-09-23T09:12:00Z",   // last connectivity TEST
    "lastSyncAt": "2026-09-23T11:40:00Z",        // last completed CRAWL  <- "Last sync"
    "containerCount": 14,
    "excludedContainerCount": 4,
    "fileCount": 1203,
    "isActive": true,
    "dateAdded": "2026-09-22T07:00:00Z"
  }
]
```

Mapping to the screen:

| column | field |
|---|---|
| Service | `provider`, with `displayName` underneath |
| Status | `status` — `Failed` shows `lastError` in a tooltip |
| Scope | static per provider: SharePoint → "Sites, libraries", OneDrive → "Personal libraries" |
| Last sync | **`lastSyncAt`**, not `lastValidatedAt` — a connector can test fine and never have been crawled, and the two dates will differ |

The three status tiles come from counting this list: **Connected** = `status == "Connected"`,
**Error** = `status == "Failed"`, **Not connected** = 5 minus the number of
distinct providers present.

### Connect a service — the wizard

**Step 1 · Prepare** is static content. The admin must, in their own Microsoft
tenant: register an app, add **Application** (not Delegated) permissions
`Sites.Read.All`, `Files.Read.All` and — **for OneDrive** — `User.Read.All`,
then click **Grant admin consent**. Missing that last click is the single most
common failure, and it does not surface until step 2.

**Step 2 · Authorize** — save, then test:

```jsonc
POST /cloud/connectors
{ "provider": "SharePoint", "displayName": "Innovly SharePoint",
  "externalTenantId": "<Directory (tenant) ID>",
  "clientId": "<Application (client) ID>",
  "clientSecret": "<secret value>" }
```

The secret is encrypted before storage and **is never returned by any endpoint**.
There is no "show secret" call to build.

```jsonc
POST /cloud/connectors/test
{ "cloudConnectorId": 2 }

→ { "success": true,
    "containers": [ { "cloudContainerId": 33, "name": "Test / Documents",
                      "ownerUpn": null, "filesScanned": 0, "sensitiveCount": 0,
                      "lastScanAt": null, "hasDeltaCursor": false,
                      "isExcluded": false } ] }
```

**Read `data.success`, not the HTTP status.** A wrong secret or missing consent
returns **HTTP 200** with `success: false` and an `error` written for an admin to
act on — it is a result the wizard renders, not an exception. Only an unexpected
server fault returns 5xx.

**Step 3 · Configure scan** — the container list above is the checkbox list:

```jsonc
POST /cloud/containers
{ "cloudConnectorId": 2 }          // every library, including excluded ones

POST /cloud/containers/exclude
{ "cloudContainerId": 122, "isExcluded": true,
  "reason": "Management account - temporary" }
```

Exclusion is reversible and non-destructive: the container, its resume cursor and
any files already found all survive, so re-including resumes rather than
restarts. Encourage a reason — an unexplained gap in coverage is
indistinguishable from a bug months later.

### Other connector actions

```jsonc
POST /cloud/connectors/delete   { "cloudConnectorId": 2 }   // soft delete
POST /cloud/scans               { "cloudConnectorId": 2, "scanType": "Incremental" }
```

`scanType`: **`Incremental`** (default) resumes each library from its cursor —
fast, and what a "Sync now" button should send. **`Full`** clears every cursor
and re-reads the whole tenant.

```
GET /cloud/scans/latest?cloudConnectorId=2
```

```jsonc
{ "cloudScanJobId": 1, "status": "Running",
  // Queued | Running | Cancelling | Completed | Failed | Cancelled
  "scanType": "Full", "filesSeen": 155, "filesExtracted": 110, "filesFailed": 0,
  "startedAt": "…", "completedAt": null, "errorDetail": null }
```

Poll this while a scan runs.

### Cancelling a scan

```jsonc
POST /cloud/scans/cancel
{ "cloudScanJobId": 1 }
```

**Cancellation is not instant, and the UI has to show that.** The crawl runs in
a separate worker process, so the API records the intent and the worker stops
itself at its next safe point — where everything is already saved. Expect:

```
Running → Cancelling → Cancelled
```

`Cancelling` can last up to ~30 seconds. Show the button as pressed and disabled
during it rather than assuming an immediate stop, and keep polling
`/cloud/scans/latest` until `Cancelled`.

A **queued** job cancels immediately (no worker involved). A job that already
finished returns its existing status rather than an error — the caller wanted it
stopped and it is.

Nothing is thrown away by cancelling: files already discovered keep their rows,
libraries keep their resume cursors, and the next scan continues from there.

### If a worker dies

A crawl whose worker is killed — a deploy, a crash, a restart — is picked up
again automatically: its heartbeat goes stale, the next worker returns it to the
queue and resumes it from the cursors. Recovery takes up to ~10 minutes, so a
job can sit in `Running` for that long with nothing happening. Do not present
that as an error; it self-heals.

---

## 2 · Exposure tab

### Activity strip — counters and the 24-hour chart

```jsonc
POST /cloud/activity
{ "cloudConnectorId": 2, "hours": 24 }   // both optional; omit the body for all services, 24h
```

```jsonc
{
  "scannedToday": 4130,
  "sensitiveToday": 325,
  "buckets": [
    { "hourUtc": "2026-09-29T08:00:00Z", "scanned": 312, "sensitive": 21, "isCurrentHour": false },
    { "hourUtc": "2026-09-29T09:00:00Z", "scanned": 0,   "sensitive": 0,  "isCurrentHour": false },
    // ... one entry per hour, oldest first, 24 in total ...
    { "hourUtc": "2026-09-30T07:00:00Z", "scanned": 96,  "sensitive": 8,  "isCurrentHour": true }
  ]
}
```

- **Every hour is present**, including empty ones — never fill gaps client-side.
  Exactly one bucket carries `isCurrentHour`, and it is incomplete by
  definition, which is why the design shows that bar in a different colour.
- `hours` is clamped to **1–168**.
- `scannedToday` is **midnight UTC to now** — a different window from the
  buckets, which is why it is not the sum of them.

> **Label this "scanned", not "uploads".** Crawls are **polled**: a worker
> claims a queued job and walks each container's delta cursor. Nothing
> subscribes to provider change notifications, so these buckets show when files
> were *read by us*, not when someone uploaded them. Expect tall bars while a
> scan ran and flat ground between — not a smooth daily curve. A tile headed
> "Uploads scanned today" over this number is an honest figure telling a lie,
> and "time to verdict, webhook to result" has no underlying event to measure.

### Tiles, Blast Radius and hotspots

```jsonc
POST /cloud/overview
{ "cloudConnectorId": 2 }     // optional — omit the body for all services
```

```jsonc
{
  "totalFiles": 1203, "sensitiveFiles": 214, "publicLinkFiles": 12,
  "totalBytes": 8123456789,

  "byClass": [ { "label": "PII", "files": 140, "sharePct": 52.4 } ],

  "blastRadius": [
    { "label": "PII", "restricted": 120, "orgWide": 14,
      "external": 0, "publicLink": 6, "total": 140 }
  ],

  "byExtension": [ { "name": "pdf", "count": 410 } ],

  "hotspots": [
    { "directory": "/sites/Finance/Shared Documents",
      "containerName": "Finance / Documents", "provider": "SharePoint",
      "sensitiveFiles": 92 }
  ],
  "hotspotLocationCount": 102,   // what the ten are a slice OF — "Top 10 of 102"

  "byExtractionOutcome": [ { "name": "Ok", "count": 1100 },
                           { "name": "Empty", "count": 103 } ]
}
```

- **`blastRadius`** is the matrix, one row per class. A file carrying two labels
  counts under both — that is what "every sensitive object by what it holds"
  means, so the rows deliberately sum to more than `sensitiveFiles`.
- **`external` is now populated.** It used to be permanently 0: exposure was read
  off the delta response's sharing facet, which says *that* something is shared
  and not *with whom*, so "shared with two colleagues" and "shared with a
  competitor" both landed in Restricted. Exposure is now derived from the
  provider's actual permissions, and an identity is external when its domain is
  not one the tenant owns.
- **`hotspots[].sharedBeyondOwnerPct`** is the bar on *Where Sensitive Data
  Sits* — the share of that folder's sensitive files reaching past their owner,
  meaning anything not `Restricted`.
- **`byExtractionOutcome` is a coverage story, not a footnote.** A large `Empty`
  count means files were read but yielded no text — most often because OCR is
  unavailable on the host, so every scanned PDF and image is invisible to
  classification. Worth surfacing rather than burying.

### All locations — the hotspot panel's "All N" link

```jsonc
POST /cloud/locations
{
  "page": 1, "pageSize": 25,
  "sortBy": "sensitiveFiles",      // directory | containerName | provider | sensitiveFiles | sharedBeyondOwnerPct
  "sortDescending": true,
  "likeFilters": { "directory": "Finance" },     // optional, CONTAINS
  "exactFilters": { "provider": ["SharePoint"] } // optional
}
```

```jsonc
{ "totalCount": 102,
  "data": [
    { "directory": "/sites/Finance/Shared Documents",
      "containerName": "Finance / Documents", "provider": "SharePoint",
      "sensitiveFiles": 92, "sharedBeyondOwnerPct": 34.8 }
  ] }
```

Every folder holding sensitive data, not just the overview's ten — `totalCount`
equals `hotspotLocationCount`, and rows are shaped exactly like `hotspots[]`.
Standard grid contract; an empty body (or no `sortBy`) returns the same
most-sensitive-first order the panel shows, so page 1 unsorted *is* the top ten.

### Service Board

```
POST /cloud/services
```

```jsonc
[ { "cloudConnectorId": 2, "provider": "SharePoint", "displayName": "…",
    "status": "Connected", "totalFiles": 1203, "sensitiveFiles": 214,
    "publicLinkFiles": 12, "lastSyncAt": "…",
    "byClass": [ { "label": "PII", "files": 140 } ] } ]
```

One card per connected service — the counts and the class-split bar.

### The file grid

```jsonc
POST /cloud/files
{
  "tab": "Sensitive",              // Sensitive (default) | All | Unclassified
  "cloudConnectorId": 2,           // optional
  "cloudContainerId": 33,          // optional
  "classifications": ["PII"],      // optional, any match
  "extensions": ["pdf","docx"],    // optional
  "ownerUpn": "user@acme.com",     // optional
  "exposureLevel": "PublicLink",   // optional
  "extractionOutcome": "Empty",    // optional — "what could not be read"
  "search": "invoice",             // file name and folder path
  "page": 1, "pageSize": 25,
  "sortBy": "modified",            // name | size | owner | confidence | firstseen | modified
  "sortDescending": true
}
```

```jsonc
{ "totalCount": 214,
  "data": [ {
    "cloudFileId": 5012, "cloudContainerId": 33,
    "containerName": "Test / Documents", "provider": "SharePoint",
    "fileName": "Payroll_2026.xlsx", "directory": "/sites/Finance/Shared Documents",
    "extension": "xlsx", "sizeBytes": 20480, "size": "20 KB",
    "lastModified": "…", "ownerUpn": "…", "webUrl": "https://…",
    "classifications": ["PII","PFI"], "isSensitive": true, "confidence": 92.0,
    "exposureLevel": "Restricted",
    "sensitiveContent": null,
    "extractionOutcome": "Ok", "extractedChars": 18422, "isTruncated": false,
    "firstSeenAt": "…"
  } ] }
```

Field notes:

- **`webUrl`** opens the file in the provider — the natural row action.
- **`exposureLevel`**: `Restricted` | `OrgWide` | `External` | `PublicLink`.
  Never null in responses; rows crawled before this existed read as `Restricted`.
- **`confidence`** is 0–100, and is the **highest single match** in the file, not
  an average — one strong match makes a file sensitive.
- **`isTruncated: true`** means only the head of a long document was read, so its
  verdict is based on part of the file. Worth a marker in the detail panel.
- **`sensitiveContent`** is the detection detail, and is **null unless the tenant
  has `Cloud:ShowSensitiveContent` enabled**. Build the panel to handle null as
  the normal case.

  Two different settings govern this, and they are worth keeping straight:

  | setting | decides | default |
  |---|---|---|
  | `Cloud:IsProtectSensitiveContent` | whether values are **masked before they are stored** | mask |
  | `Cloud:ShowSensitiveContent` | whether the stored panel is **returned by the API** | off |

  `Cloud:IsProtectSensitiveContent` is the only key cloud reads — it does **not**
  fall back to the product-wide `IsProtectSensitiveContent`, which governs
  LandingLog alone. A tenant with no cloud row gets masked values regardless of
  what the product-wide flag says.

  Masking happens **on write and is one-way**: turning protection off does not
  reveal anything already stored, and turning it on does not hide anything
  already stored in the clear. Only a re-scan changes existing rows. For a tenant
  with protection off, `Cloud:ShowSensitiveContent` is the only thing standing
  between a real card number and the response.

Tab meanings: **Sensitive** = something was found. **Unclassified** = read, but
nothing found — this is where unreadable and OCR-less files land, so it is the
coverage tab, not an empty one. **All** = everything discovered.

---

## 3 · Users tab

```jsonc
POST /cloud/users
{ "page": 1, "pageSize": 20,
  "sortBy": "SensitiveFiles", "sortDescending": true,
  "likeFilters":  { "UserName": ["mac"] },
  "exactFilters": { "Risk": ["High"] } }
```

Same `QueryParameters` contract as every other grid. Filter and sort keys are the
row's own field names: `UserName`, `Department`, `SensitiveFiles`,
`PublicLinks`, `ExternalShares`, `Risk`, `OwnerUpn`.

```jsonc
{
  "totalCount": 1284,
  "summary": {
    "usersWithSensitiveFiles": 1284,
    "usersScanned": 1903,
    "highRiskUsers": 46,
    "usersWithPublicLinks": 188,
    "topTenSharePct": 31.0
  },
  "data": [{
    "ownerUpn": "macferrari.m@company.com",
    "userName": "Mac Ferrari", "department": "Finance",
    "services": [ { "provider": "SharePoint", "files": 2180 }, … ],
    "sensitiveFiles": 3430,
    "classifications": [ "PFI", "PCI", "PII" ],
    "publicLinks": 14, "externalShares": 62,
    "risk": "High"
  }]
}
```

**`summary` describes the whole population, not the page** — it does not change
as you page.

### What each number means

| field | definition |
|---|---|
| `sensitiveFiles` | classified, non-deleted files owned by this UPN |
| `publicLinks` | **links**, not files — anonymous links on their files |
| `externalShares` | **files**, not links — files whose exposure is `External` |
| `usersScanned` | everyone we saw any file for — the tile's denominator |
| `topTenSharePct` | the ten largest holders' share of all sensitive files |

The two share-counts use different units deliberately: *"how many public links
has this person got out"* and *"how many of their files reach outside the
company"* are both asked on that screen, and one unit answers neither.

### Risk

```
High    a public link on a PHI / PCI / PFI file, or more than 10 public links
Medium  any public link, or any externally shared file
Low     holds sensitive files, none reaching beyond the owner
```

Computed per request, never stored — a stored band is wrong the moment a link is
revoked. `Low` is a real answer and never null: "no sharing" and "no data" must
not render alike.

> **Users we cannot name still appear.** When a file's owner matches no user
> record, `userName` falls back to the UPN and `department` is null. Dropping
> them would quietly understate every tile above.

### The detail panel

```jsonc
POST /cloud/users/detail   { "ownerUpn": "macferrari.m@company.com" }
```

Returns the same figures plus `byService` (the panel's bars) and `files` — their
**most exposed** files, worst first: public link, external, org-wide, restricted.
Capped at 50. Each carries `cloudFileId`, so the panel links straight to
`GET /cloud/files/{id}/link`.

`404` means the user holds no sensitive files — a real answer, not a failure.

---

## 4 · Public Links tab

```
POST cloud/links
{
  "scope": "anonymous",        // anonymous | organization | users. Default anonymous
  "cloudConnectorId": 3,       // optional — omit for every service
  "classifications": ["PHI"],  // optional, any match
  "sensitiveOnly": true,       // optional — only links on classified files
  "ownerUpn": null,
  "search": null,              // matches file name and folder path
  "page": 1, "pageSize": 25,
  "sortBy": null,              // default: newest link first
  "sortDescending": true
}
```

Returns `{ totalCount, summary, data[] }`.

**Scope defaults to `anonymous`** because the screen is called Public Links, and a
link anyone signed in can use is a different question. Pass `scope` explicitly
for the others, or `null` for all three.

**Revoked links never appear.** They are kept in the database — "this was public
for six weeks and is not any more" is what an investigation needs — but a list
headed *in circulation* must not carry them.

### The four tiles

| Tile | Field | Notes |
|---|---|---|
| Public links · with sensitive content | `summary.publicLinksWithSensitiveContent` | |
| — | `summary.publicLinks` | all anonymous links, classified or not |
| Outside companies | `summary.outsideCompanies` | distinct external domains |
| — | `summary.externalPeople` | named outsiders across the filtered set |
| **Anonymous opens** | ❌ **not available** | see below |
| **Places shared** | ❌ **not available** | see below |

Summary counts describe the whole filtered set, not the page being shown.

### The table

| Column | Field |
|---|---|
| File | `fileName`, `directory` |
| Owner | `ownerUpn` — the file's owner, which is often **not** the link's creator |
| Sensitive data | `classifications[]` |
| Source | `provider`, `containerName` |
| Who can open it | `scope` + `externalIdentityCount` — see below |
| Link created by | ❌ **always empty** — see below |
| Age | ❌ **always empty**; use `seenDays`, labelled "first seen" |
| **Destination** | ❌ **not available** |

**`createdByName` and `ageDays` are structurally empty on anonymous links.**
Measured on a live tenant: 0 of 1,955 links carried a creator or a creation
date. It is not a permission you can grant — Graph withholds `createdBy` and
`createdDateTime` from application tokens exactly as it withholds
`link.webUrl`. Box, Dropbox and Google report no creation date either, and of
the three only a Box *collaboration* names who invited whom.

`creatorRole` derives from the same field and is empty for the same reason.

Use **`seenDays`** for the age column — days since the crawl first saw the link.
Label it *first seen*, not *age*: a link created a year ago on a tenant
connected last week reads 7, and putting that under an "Age" heading reports a
number nobody measured.

**"Who can open it"** is composed, not a single field:

| Show | When |
|---|---|
| **Anyone** | `scope === "anonymous"` |
| **Outsiders** · *"Sent to N outside companies"* | `externalIdentityCount > 0`, N = `externalDomains.length` |
| **Org-wide** | `scope === "organization"` |
| Named people | `scope === "users"`, count from `internalIdentityCount` |

The mock's *"96 opened without signing in"* underneath is the part we cannot
supply — the badge above it is real, the count is not.

### The detail panel

`linkUrl`, `seenDays` ("First seen 41 days ago" — `ageDays` and `createdByName`
are always empty, see above),
`internalIdentityCount` / `externalIdentityCount` / `externalDomains`
("Employees 206 · Outsiders 7 people · 4 companies"), plus `hasPassword` and
`expiresAt`, which the mock does not show and probably should — a link with an
expiry is a materially smaller problem than one without.

**"Where the link has been shared"** is the whole lower half of that panel, and
none of it can be served. See below.

### What cannot be served, and why

Design these away rather than binding them to fields that stay at zero.

**Anonymous open counts** — *"326 opens"*, *"96 opened without signing in"*,
*"viewed 1.2k times"*. These are access events, and no provider reports them
alongside the file. They live in each provider's separate audit feed — Purview,
Drive Activity API, Box events, Dropbox team log — four different APIs with
different retention and permissions, returning event streams rather than a
number. It is an ingestion subsystem, not a field.

**Destination chips and "where the link has been shared"** — Slack, Teams,
Email, Wiki, and the *Places shared · 19 channels and threads* tile. Knowing a
Drive link was pasted into `#clinical-ops` means integrating Slack, Teams,
Confluence and mail and correlating URLs across them. That is a
link-propagation product, larger than all five storage connectors together.

The badge — *who can open it* — is the finding worth acting on. "This file is
reachable by anyone on the internet" is what needs a decision; the open count
tells you how bad it already is, not whether to act.

### Post-share access revoke (what the policy did about it)

Every row of `cloud/links` now also carries what Policy V2 made of that share.
`DLPProcessLaterService` resolves the policy governing the **file's owner**,
evaluates the policy's **Upload** rules against the share (the Cloud channel,
the file's type, what the scanner found in it), and revokes the link at the
provider when that evaluation says Block.

**Two jobs, because noticing and deciding are different problems.** A
`CloudFileLink` row used to be written only by the full crawl, and a crawl runs
when somebody asks for one — so a link created at 02:00 was not decided until
the next crawl, however often the decider ran.

| Job | Every | Does |
|---|---|---|
| Share watcher | 60s | walks each container's change feed, reads permissions for items reported as shared, writes the link rows, hands them straight to the sweep |
| Policy sweep | 15 min | decides anything the watcher missed, and re-decides links whose policy or classification has changed |

The watcher downloads nothing and classifies nothing — it is the cheap half of
a crawl, run often. **It therefore cannot match a content rule on a file the
crawl has never read.** A file that already existed and is newly shared carries
its classifications, so "no card data shared outside the company" fires within
the minute; a file created and shared in the same minute has none yet, so only
file-type and name rules can catch it until the crawl reaches it and the sweep
re-decides.

**60 seconds is a floor, not a promise.** The watcher walks
`CloudShareWatchContainersPerTick` containers per tick (seeded 200). A tenant
with fewer containers than that sees all of them every tick; a tenant with more
sees them in rotation, so 5,000 OneDrive drives at 200 a minute is a 25-minute
round trip. The worker logs the round trip it is actually achieving, hourly,
per tenant — it does not leave anyone to infer it. Containers the crawl has
never been through are skipped, because a watcher with no cursor would
enumerate the whole drive inside a one-minute loop.

| Field | Meaning |
|---|---|
| `policyDecision` | what the **policy says**: `Allow` · `Warn` · `Block` · `NoPolicy` |
| `policyStatus` | where the link **stands**: `Validated` · `Monitored` · `Revoked` · `Failed` · `Skipped` |
| `policyStatusReason` | one sentence, ready to show — the rule that matched and what was done |
| `policyName` / `policyRuleName` | which policy and which of its rules |
| `policyValidatedAt` | when it was last evaluated |
| `policyStatusChangedAt` | when the status last **changed** — not when it was last confirmed |
| `revokedAt`, `policyError` | when it came down, or what the provider said |

All of these are **null when the sweep has not reached the link yet**, and null
`policyDecision` is not "allowed" — one is "nobody has looked", the other is "a
policy was evaluated and permits it". Rendering both as a tick claims an
assurance nobody gave.

Two columns rather than one because they answer different questions.
`policyDecision` is fixed by the policy; `policyStatus` is the one that moves — a
`Failed` link becomes `Revoked` when a retry succeeds, and a `Monitored` one the
moment the tenant switches to Enforce.

**`Monitored` is the one worth surfacing loudest.** It means the policy blocks
this link and the tenant has not switched enforcement on — the link is still
live. The tenant's `dlp."ExeSettings"` decide that:

| Key | Values | Seeded |
|---|---|---|
| `CloudSharePolicyMode` | `Off` · `Monitor` · `Enforce` | **`Monitor`** |
| `CloudSharePolicyMinExposure` | `OrgWide` · `External` · `PublicLink` | `External` |
| `CloudSharePolicyMaxPerSweep` | provider calls per tenant per sweep | `500` |

Nothing is ever revoked until somebody sets that tenant to `Enforce`, and even
then only links reaching at least `CloudSharePolicyMinExposure`. The floor
exists because a Policy V2 rule describes *content* and *channel* and says
nothing about *audience*: without it, "no payroll data in SharePoint" would cut
off the two people in Finance who maintain the file.

Two conditions a rule can carry that a stored cloud file cannot answer, and
which therefore never cause a revoke: a **keyword** (cloud file text is not
kept, so the rule is reported as unchecked rather than matched) and the
**transfer size cap** (it holds an upload for admin approval, and the file is
already there).

Behind those fields are two tables, and the split matters if you query them
directly:

- `discovery."CloudSharePolicyValidation"` — **one row per link, updated in
  place.** The present. There is no `Pending` status: "not validated yet" is the
  absence of a row, and "needs re-validating" is `ValidatedAt` older than the
  link's `LastSeenAt`.
- `discovery."CloudShareEnforcement"` — **one row per decision, never
  rewritten.** The past. Every decision including every Allow, with the policy
  name, rule name, mode and the exposure as they stood at the time. This is what
  answers "why did this stop being shared on the 14th" however many times the
  status has moved since.

---

## Coverage — the part that stops a screen lying

Every container now records **why** it produced what it did:

```
POST cloud/containers   { "cloudConnectorId": 3 }
```

| Field | Meaning |
|---|---|
| `lastScanOutcome` | `Ok` · `Empty` · `Denied` · `NotFound` · `Locked` · `Failed` |
| `lastScanError` | the provider's own words |
| `lastScanItemsSeen` | items returned before filtering |

**Anything other than `Ok` or `Empty` is a coverage gap and should be visible.**
On a real tenant there are always a few — a site on legal hold, an unlicensed
account, a library the app was never consented for. Before this existed, a drive
we could not read and a drive with nothing in it produced the same zero, and the
screens reported the tenant as clean either way.

Worth a line on the Integrations tab: *"12 of 61 drives could not be read"*, with
the reasons behind it.

---

## Two things to design around

**Classification is regex-based and matches the endpoint agent exactly** — same
package, same patterns, so a file in SharePoint and the same file on a laptop
produce the same labels. Classes are `PII`, `PHI`, `PFI`, `BESD`.

**Nothing here is real-time.** The screens' "scanned as it lands" and the
response/remediation actions (restrict link, notify owner) are a later phase —
they need provider webhooks and write permissions that customers have not
consented to. Today a file appears after the next crawl of its library.
