# Database Scanner — UI ↔ API Contract

For the UX designs of the **Scanner → Database Connector** screens (Overview /
Connections / Sensitive tables / Connect wizard / connection drawer).

**Envelope**: every response is the house `{ success, message, data, errors }`;
grids put `{ TotalCount, Data }` inside `data`. **Auth**: console JWT + the
usual `ClientGUID` tenant header. **APIM**: every route below must be
registered before QA — an unregistered route answers the gateway's 404.

One concept to hold onto: a **connection** is either a *cloud connector*
(`source: "Cloud"` — the platform signs in directly) or an *agent instance*
(`source: "Agent"` — the on-site server scanner holds the credential and pushes
results). Every id you send back for a connection must travel WITH its
`source`; the two id spaces are different tables.

---

## 1. Overview tab

`GET database/overview` — everything in one call.

| Screen element | Field |
|---|---|
| "214 out of 1,486 tables read in 7 connections" | `sensitiveTables`, `tablesRead`, `connectionCount` |
| Category bar + counts (PII 92, PHI 61, …) | `byClass` — JSON string `{"PII":92,"PHI":61,...}` |
| "Did we read everything? 19 of 23" + status rows | `coverage` — JSON string; use `coverage.databases`: `{read, accessDenied, failed, pending, excluded, total}`. "X databases couldn't be read" = `accessDenied + failed` |
| "Where Regulated Data Lives" heatmap | `byContainer` — JSON string: ranked array `[{container, instance, classes:{PII:14,...}, total}]` |
| "Needs Your Attention" | `attention[]`: `{severity: Error\|Warning\|Info, title, detail, connectionId, source, action: View\|Fix\|Test}` — already sorted worst-first |
| "Scanning 1 of 7 connections" (header pill) | `scanningCount` + `connectionCount` |
| Engine split (if shown) | `byEngine` — JSON string `{"SqlServer":{instances,sensitiveObjects},...}` |

`byClass`/`byEngine`/`byContainer`/`topObjects`/`coverage` are JSON **strings**
(the stored aggregates, verbatim) — parse once. `computedAt` tells the user how
fresh they are; null + zero counters = nothing scanned yet, render the empty
state, not a spinner.

## 2. Connections tab

`GET database/connections` → `Data: DatabaseConnectionListItem[]`, pre-sorted
(scanning first, then by sensitive tables).

| Column | Fields |
|---|---|
| Connection (icon, name, host) | `engine` (icon key), `engineDisplayName`, `name`, `hostDescriptor`, `source` |
| Sign-in | `signInStatus`: `Connected` (+`lastTestedAt`) / `ConnectionFailed` (+`signInError`) / `NotTested` / `ViaAgent` (render "via on-site agent") |
| Last scan | `scanStatus`: `NeverScanned`, `Queued`, `Scanning` (+`tablesRead`, `scanStartedAt` for "186 tables read · 7 min"), `Cancelling`, `NotResponding` (+`retryAttempt`/`retryLimit` for "retrying (try 2 of 3)"), `Completed`, `CompletedWithGaps` (+`databasesNotRead` for "1 not read"), `Failed` (+`scanError`), `Cancelled`, `Stopped` |
| Sensitive tables | `sensitiveTables` "of" `totalTablesRead` |
| In scope | `scopeSummary` ("4 databases" / "2 schemas" / "Everything it can see") + `databasesNotRead` + `excludedCount` |
| Button | `primaryAction`: `Scan` / `Cancel` / `Fix` / `Test` / `None` (agent rows: no button — scans start on the server, Phase 3 adds the lever) |

Poll this list (or the drawer) every ~10 s while `scanningCount > 0`; `Scanning`
rows move `tablesRead` on every crawler heartbeat (~30 s).

## 3. Connection drawer

`GET database/connections/detail?connectionId={id}&source={Cloud|Agent}`

- `header` — same shape as a list row (the three stat cards: Sign-in, Last
  scan, Coverage = `scopeSummary` + `databasesNotRead` + `excludedCount`).
- `containers[]` — the **Databases tab**: `{containerId, name, status:
  Pending|Scanned|AccessDenied|Failed, isExcluded, tablesRead, sensitiveCount,
  dominantClass ("mostly PHI"), lastError (the provider's words for the
  access-denied banner), exclusionNote, excludedBy, excludedAt, lastScanAt}`.
  The "1 database wasn't read … treat as unscanned, not clean" banner =
  any row with `!isExcluded && status in (AccessDenied, Failed)`.
- `connection` — the **Connection tab**: `{engineDisplayName, server,
  databasesScope, login, password (always the fixed "stored encrypted"
  sentence — the value never exists in any response), firstAcceptedAt,
  lastTestedAt, lastScanAt}`.
- **Scan history tab**: `POST database/scans/history`
  `{connectionId, source, page, pageSize}` → rows `{scanId, status, trigger,
  startedAt, completedAt, tablesRead, sensitiveFound, failedReads, error}`.

Exclude / re-include a database:
`POST database/containers/exclude` `{containerId, exclude: true|false, note}` —
the author and date are stamped server-side from the signed-in user. Re-include
clears note+author (they explained the exclusion, not the database).

## 4. Scan buttons

- `POST database/scans/start` `{connectionId}` (Cloud connections only) →
  `{jobId, alreadyActive}`. Double-click safe: an active job is returned, not
  duplicated — if `alreadyActive`, just flip the row to its live state.
- `POST database/scans/cancel` `{connectionId}` → 200. Cancelling is a request
  the scanner honours at its next safe boundary; show `Cancelling` until the
  history row lands on `Cancelled`.

## 5. Sensitive tables tab

`POST database/objects` `{instanceId?, container?, classification?, search?,
sensitiveOnly: true, page, pageSize}` — all filters optional; omit
`instanceId` for the cross-estate view. Class tabs = the same call with
`classification: "PII"` etc.; tab counts come from the overview's `byClass`.

Row fields: `objectName`, `schemaName`, `objectType` (Table/View/Collection —
drives the "Table · read 2 hrs ago" line with `lastScanAt`), `container` +
`instanceName` (the Database column's two lines), `classifications[]` (the
badges), **`recognised`** ("National ID, date of birth, diagnosis" — derived
from the masked evidence's sub-category names; it can never contain a real
value), `confidence` (0–100, the bar), `severity`/`risk`/`dpdpFlag` for detail.

## 6. Connect wizard

| Step | API |
|---|---|
| 1 — engine cards + setup guide text | `GET database/providers` → `{key, displayName, tenantLabel, clientIdLabel, clientSecretLabel, extraFields[], setupSummary}`. Render a card per entry (5 today — MySQL/MariaDB is in the list even though the mock showed four). The field labels ARE the step-2 form labels. |
| 2 — details + "Save and test" | **Existing shared connector routes** — create via `cloud/connectors` with `kind: "Database"`, provider key from step 1, name→DisplayName, server→ExternalTenantId, login→ClientId, password→ClientSecret, optional databases list→extras `databases`. (Connector CRUD deliberately stays on the shared surface; one credential store, one path.) |
| 3 — test, "we found 5 databases" | `cloud/connectors/test` — the scope list returned is exactly the found-databases list ("`database patients_prod`", …); a name the login cannot see comes back flagged "(NOT VISIBLE to this login)". |
| 4 — "Start first scan" | `POST database/scans/start` `{connectionId}` — or Scan later, which is just closing the drawer. |

Replace / remove connection (drawer footer): the shared `cloud/connectors`
update/delete routes. Removing stops scans; the inventory rows found so far
stay (retire the instance separately via `database/instances/set-active`).

## 7. States worth getting right

- **Unscanned ≠ clean.** Anywhere `databasesNotRead > 0`, the design's own
  wording applies: "they count as unscanned, not clean".
- **Agent rows have no Scan button** (`primaryAction: "None"`). The row still
  shows everything else; scans arrive when the on-site agent runs.
- **`NotResponding` is derived**, not stored: a running scan silent >10 min.
  It resolves by itself (retry or `Stopped`); the console only watches.
- **No secret ever appears in any response** — passwords are a fixed sentence,
  connection strings never exist server-side beyond the call that uses them.
