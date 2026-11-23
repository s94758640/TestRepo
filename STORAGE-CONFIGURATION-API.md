# Storage configuration API

Backend for the **Settings › Connectors › Storage** screen: where a tenant's
evidence, logs and reports are kept, how it is changed, and what it is costing
them in space.

Host: `SFM.DLP.API`. All routes are under `/storage`. Everything is per tenant —
the DLP connection is resolved from `ClientGUID`, so there is no client id in any
body and no way to address another tenant's storage.

---

## 1. How it works

> "Choose where Defensely keeps evidence, logs and reports. One destination is
> active at a time."

That destination is one row in `dlp."StorageTarget"` with `Purpose = 'All'`.
`StorageTargetResolver` answers every storage request in this order:

| # | Source | Set by |
|---|---|---|
| 1 | `StorageTarget` row for the purpose (`Video`, `ShadowFile`) | the older per-purpose API, `POST /storage/targets/save` |
| 2 | **`StorageTarget` row for `All`** | **this screen** |
| 3 | the tenant's Nucleus `ClientSetting` storage rows | tenant provisioning |
| 4 | the deployment's `Storage` section in appsettings | the deployment |

So:

* **Defensely Storage** is not a row. Choosing it **retires** the `All` row and
  the tenant falls through to 3/4 — which is exactly where every tenant is today.
  An empty table means today's behaviour, unchanged.
* A tenant who already had `Video` pinned to its own account keeps that split.
  Those rows are returned in `purposeOverrides` so the screen can say so rather
  than claiming a single destination that is not the whole truth.
* Switching **never moves data**. Retired rows are kept, so files written to a
  previous destination stay addressable and readable. Moving them is the
  separate, resumable copy job in §6.

### One container holds everything

A customer grants `Storage Blob Data Contributor` on **one** container. The
product writes to several (`videos`, `dlp-shadow-files`, `dlp-log-files`). When a
destination names a container, every other container becomes a **top-level
folder inside it** (`SingleContainerObjectStore`): `videos/video/<endpoint>/…`.
Nothing above `IObjectStore` changes, the customer's grant stays one line, and
the usage breakdown gets a folder per category for free.

A folder prefix, if set, is applied on top of that (`PrefixedObjectStore`), so
`defensely/videos/video/…`. Object names recorded in the database never include
the prefix — it belongs to the destination, not to the object.

---

## 2. Endpoints

| Method | Route | Screen element |
|---|---|---|
| `GET` | `/storage/configuration` | the whole page in one call |
| `GET` | `/storage/providers` | the "Change storage" dialog's cards and forms |
| `POST` | `/storage/test-connection` | **Test connection** |
| `POST` | `/storage/switch` | **Save and switch** |
| `POST` | `/storage/disconnect` | back to Defensely Storage |
| `POST` | `/storage/destinations/{id}/forget` | erase one connection's credentials, keep its history |
| `DELETE` | `/storage/destinations/{id}` | remove the connection entirely — `?confirm=true` |
| `GET` | `/storage/usage` | "What's stored" (cached figures) |
| `POST` | `/storage/usage/refresh` | re-measure against the live account |
| `POST` | `/storage/copy` | "copy existing files across" |
| `GET` | `/storage/copy/current` | the progress bar |
| `POST` | `/storage/copy/run` | advance the copy while the admin watches |
| `POST` | `/storage/copy/{id}/cancel` | stop it |
| `GET` | `/storage/copy/history` | past copies |
| `POST` | `/storage/run-scheduled` | **timer only** — anonymous, `ClientGUID` header |

All responses use the house `ApiResponse<T>` envelope — `success`, `message`,
`data`, `errors` — exactly as every other DLP endpoint does. The payloads below
are the `data` member.

### Who may call them

`Storage` (module code `STORAGE`) in the permission catalog, seeded by
`Migrations/Common/V0008`, and granted to `SuperAdmin` by
`Migrations/Common/V0009`:

| Permission | Routes |
|---|---|
| `VIEW` | `GET /storage/configuration`, `/providers`, `/usage`, `/copy/current`, `/copy/history` |
| `EDIT` | `POST /storage/test-connection`, `/switch`, `/disconnect`, `/usage/refresh`, `/copy`, `/copy/run`, `/copy/{id}/cancel` |

`EDIT` implies `VIEW`. A caller without the permission gets **403**.
`usage/refresh` is `EDIT` rather than `VIEW` deliberately: a sweep lists every
object the tenant stores and costs real requests at their provider.

While the seed rolls out, an `Admin` role is accepted alongside the permission —
the same rollout shape `DiscoveryController` uses — so no admin is locked out of
a tenant database the migration has not reached. Drop the role half once V0008
has run everywhere, including the master Common database.

**The role fallback does not help the web app.** The API lets an Admin through
without the permission; the console decides what to *render* from the token's
`rolePermissions` claim, so a role missing the ids has no Storage page to let
anyone in. That is what V0009 is for: `seed_superadmin_dormant.sql` grants every
permission that exists *at the moment it runs*, so `SuperAdmin` — the one role
meant to see everything — was the one role without `STORAGE`.

The claim is rebuilt from the database on every login **and** on every refresh
(`AuthenticationsService.BuildLoginClaimsAsync`), so a fresh sign-in picks the
new ids up with no code change and no cache to clear. Tokens minted before the
migration keep the old list until they refresh.

Everything except `run-scheduled` requires the usual bearer token.

Pre-existing routes on the same controller prefix are untouched:
`POST /storage/upload-url`, `GET /storage/shadow-keys`,
`POST /storage/shadow-keys/rotate`, `POST /storage/targets`,
`POST /storage/targets/save`.

---

## 3. `GET /storage/configuration`

Everything the page renders, in one call — there is no state in which three
quarters of this screen is useful.

```jsonc
{
  "success": true,
  "message": null,
  "errors": null,
  "data": {
    "current": {
      "storageTargetId": null,               // null = Defensely-managed
      "providerKey": "defensely",
      "providerName": "Defensely Storage",
      "displayName": "Defensely Storage",
      "subtitle": "Azure Blob · Central India",
      "provider": "AzureBlob",
      "isManaged": true,
      "isCurrent": true,
      "purpose": "All",
      "target": null,                        // account name / bucket / root path
      "targetUrl": null,                     // the full resolved address; null on managed
      "containerName": null,
      "folderPrefix": null,
      "region": null,
      "authMode": null,
      "isChosen": false,                     // has anyone ever picked? see below
      "downloadLinkExpiryMinutes": 60,       // same name the form and the switch body use
      "values": {},                          // saved non-secret values, keyed by field.name
      "status": "Healthy",                   // Healthy | Pending | Failing
      "lastTestedUtc": null,
      "lastError": null,
      "hasCredentials": true,
      "canRemove": false,                    // see "Asking first: canRemove" in §5
      "removeRequiresConfirm": false,
      "removeReason": "Defensely Storage is the default destination. It has no connection to remove.",
      "objectsRecorded": null,               // null here = managed; elsewhere = count unavailable
      "encryption": "AES-256 at rest · Defensely-managed keys",
      "retention": "Follows your plan and add ons",
      "defaultFor": "All data unless you connect your own storage",
      "source": "Configuration",             // StorageTarget | ClientSetting | Configuration
      "activatedUtc": null,
      "retiredUtc": null
    },
    "usage": {
      "usedBytes": 442381631488,
      "objectCount": 1651915,                // every object, all categories
      "quotaBytes": 1099511627776,           // managed storage only; null on your own cloud
      "freeBytes": 657129996288,
      "usedPercent": 40.23,
      "measuredUtc": "2026-11-13T04:10:22Z",
      "isMeasured": true,
      "lastError": null,
      "destinationBytes": 442381631488,        // in the current destination
      "destinationObjectCount": 1651915,
      "elsewhereBytes": 0,                     // still in Defensely Storage — see §6
      "elsewhereObjectCount": 0,
      "elsewhereMeasured": false,              // false = not counted, NOT "empty"
      "estimatedDurationSeconds": 183200,      // "about 2 days" — see §6
      "categories": [
        { "key": "ActivityLogs", "label": "Activity Logs",
          "description": "Upload, copy and print events from endpoints",
          "sizeBytes": 79456894976, "objectCount": 1284311, "percent": 17.96,
          "measuredUtc": "2026-11-13T04:10:22Z", "lastError": null,
          "measuredAt": "AzureBlob/dlp-log-files" },   // where the figure came from
        { "key": "ShadowFile", "label": "Shadow File Copy",
          "description": "Copies of files captured when a policy blocks or flags an upload",
          "sizeBytes": 210453397504, "objectCount": 338142, "percent": 47.57,
          "measuredUtc": "2026-11-13T04:10:22Z", "lastError": null },
        { "key": "Video", "label": "Forensic Video Evidence",
          "description": "Screen recordings captured around flagged activity",
          "sizeBytes": 96636764160, "objectCount": 20418, "percent": 21.84,
          "measuredUtc": "2026-11-13T04:10:22Z", "lastError": null },
        { "key": "Other", "label": "Others",
          "description": "Discovery scan results and report exports",
          "sizeBytes": 55834574848, "objectCount": 9044, "percent": 12.62,
          "measuredUtc": "2026-11-13T04:10:22Z", "lastError": null }
      ]
    },
    "copy": null,                            // or a job — see §6
    "purposeOverrides": [],                  // per-purpose rows that outrank the destination
    "previous": [],                          // destinations this tenant has left
    "copySources": [],                       // where a copy can read from — see §6
    "switchLocked": false,                   // true while a copy is Queued or Running
    "switchLockReason": null                 // the tooltip, and the 409's words
  }
}
```

`purposeOverrides[]` and `previous[]` carry **the same object as `current`** —
every field above, with the differences that matter:

| Field | In `purposeOverrides` | In `previous` |
|---|---|---|
| `purpose` | `Video` or `ShadowFile` — what this row serves | `All` |
| `isCurrent` | `true` (it is in force for that purpose) | `false` |
| `retiredUtc` | null | when the tenant left it |
| `storageTargetId` | always set | always set |

So the override note reads straight off `purpose`, `displayName`, `provider` and
`targetUrl` — nothing extra to fetch.

### `isChosen` — never picked, or picked Defensely

`current.isChosen` is **false** until someone chooses a destination for this
tenant, and true forever after. It is not "is the tenant on managed storage":

| State | `isChosen` | `current` |
|---|---|---|
| Never opened this screen | `false` | Defensely Storage |
| Chose Defensely deliberately | `true` | Defensely Storage — identical otherwise |
| Chose their own bucket | `true` | their destination |

Confirming Defensely is `POST /storage/switch` with
`{ "providerKey": "defensely" }`. Nothing about where data goes changes — it
already goes there — and the call is idempotent, so a second confirmation is
harmless.

Behind it is a retired `StorageTarget` row carrying nothing but the choice. It
is born inactive, the resolver is only ever handed **active** rows, and it is
excluded from `previous[]`, so it is invisible to everything except this flag.
Only `Purpose = 'All'` rows count, because that is the only purpose this screen
writes — a tenant with a legacy per-purpose override has not chosen anything
here and still reads `false`.

### Editing a saved destination

`current.values` is the destination's saved non-secret values keyed by the
**`field.name`** from `GET /storage/providers` — `azureAccountName`,
`s3BucketName`, `s3Region`, `s3Endpoint`, `s3RoleArn`, `s3ForcePathStyle`,
`gcsBucketName`, `gcsProjectId`, `containerName`, `folderPrefix`,
`displayName`, `downloadLinkExpiryMinutes`.

Those are the same names the switch body takes, so a generated form
round-trips with no mapping table: read `values[field.name]`, post
`field.name`. A key that is **absent** means nothing was saved — render an
empty box; a key is never present with an empty value.

**Secrets are never in `values`.** No account key, SAS token, connection
string, S3 secret key or service-account JSON, whatever the card asks for.
Leave those inputs blank and send `storageTargetId` to reuse what is stored.
The S3 access key *id* and the role ARN are identifiers, not credentials, so
those do come back.

### UI element → field

| Screen | Field |
|---|---|
| "Defensely Storage" + `Default` chip | `current.displayName`, `current.isManaged` |
| "Azure Blob · Central India" | `current.subtitle` |
| green `Healthy` dot | `current.status` (+ `lastError` in a tooltip when `Failing`) |
| "412 GB used of 1 TB", "588 GB free" | `usage.usedBytes`, `usage.quotaBytes`, `usage.freeBytes` |
| "18,420 files · 412 GB" on the switch confirmation | `usage.objectCount`, `usage.usedBytes` |
| "To: https://…/dlp-evidence/defensely/" | `targetUrl`, from the test result or the destination |
| progress bar | `usage.usedPercent` |
| Encryption / Retention / Default for | `current.encryption`, `.retention`, `.defaultFor` |
| "WHAT'S STORED" total | `usage.usedBytes` |
| stacked bar + four rows | `usage.categories[]` — `label`, `description`, `sizeBytes`, `percent` |

`quotaBytes` is **null** when the tenant is on their own cloud: that bucket's
limits belong to their provider, and printing a number there would be fiction.
Show "—" or hide the allowance line.

`measuredUtc` is the age of the figures. Show it. A number with no age invites an
admin to read a six-hour-old figure as live.

---

## 4. `GET /storage/providers`

The dialog's cards and the forms behind them, so a corrected permission list or a
new region ships without a UI release.

`?includeSelfHosted=true` adds the on-premises "folder on your server" provider.
Off by default: in SaaS the API host cannot see the customer's disk.

```jsonc
[
  { "key": "defensely", "displayName": "Defensely Storage",
    "summary": "Azure Blob, managed by Defensely. Default.",
    "hosting": "Hosted by Defensely · no setup", "badge": "D",
    "isCurrent": true, "isImplemented": true,
    "authModes": [], "fields": [], "switchNotes": [ "..." ] },

  { "key": "azure-blob", "displayName": "Azure Blob Storage",
    "summary": "Your own Azure storage account",
    "hosting": "Your cloud · Microsoft Azure", "badge": "AZ",
    "authModes": [
      { "key": "account-key", "label": "Account key", "isDefault": true,
        "fields": [
          { "name": "azureAccountName", "label": "Storage account name",
            "type": "text", "placeholder": "mystorageaccount", "required": true },
          { "name": "containerName", "label": "Container name",
            "type": "text", "placeholder": "dlp-evidence", "required": true },
          { "name": "azureAccountKey", "label": "Account key", "type": "secret",
            "placeholder": "Paste value", "isSecret": true,
            "helpText": "Found under Security + networking › Access keys." }
        ] },
      { "key": "sas-token",         "label": "SAS token",         "fields": [ "..." ] },
      { "key": "connection-string", "label": "Connection string", "fields": [ "..." ] }
    ],
    "fields": [                                  // shown under STORAGE OPTIONS, all modes
      { "name": "folderPrefix", "label": "Folder prefix", "type": "text",
        "placeholder": "defensely/", "required": false,
        "helpText": "Optional. Files are written under this path." },
      { "name": "downloadLinkExpiryMinutes", "label": "Download link expiry",
        "type": "select", "required": false,
        "helpText": "How long signed links to evidence files stay valid.",
        "options": [ { "value": "15", "label": "15 minutes" },
                     { "value": "60", "label": "1 hour" },
                     { "value": "360", "label": "6 hours" },
                     { "value": "1440", "label": "24 hours" },
                     { "value": "10080", "label": "7 days" } ] }
    ],
    "permissionsSummary": "Assign this role on the container, or create a SAS token with read, write, delete and list.",
    "requiredPermissions": [ "Storage Blob Data Contributor", "sp=rwdl" ],
    "setupGuideUrl": null,
    "switchNotes": [
      "New data goes to the new storage right away.",
      "Existing files stay where they are and remain readable.",
      "You can copy existing files across now or later from this page.",
      "Credentials are encrypted and never shown again after saving."
    ] }
]
```

`setupGuideUrl` is **null unless `Storage:SetupGuideBaseUrl` names an absolute
`https://` base** for the documentation site, in which case every card returns a
full URL. It is never a bare path: a relative one resolves against the console's
own origin and opens a page that does not exist, and a dead "Setup guide" link
is worse than none — the admin stops looking for the real instructions. Keep the
link hidden while it is null.

`field.name` is exactly the property to send back in §5's body, and exactly the
key to read out of `current.values` when reopening a saved destination. `type` is:

| `type` | Renders as | Posts |
|---|---|---|
| `text`, `number` | an input | the value |
| `secret` | a password box | the value, plaintext, once |
| `select` | a dropdown of `options[]` | `option.value` |
| `file` | an upload (the GCS JSON key) | the file's **contents** |
| `checkbox` | a checkbox, default from `value` | a boolean |
| `readonly` | a value with a copy button when `copyable` | **nothing** — it is generated, not entered |

`readonly` exists for one field today: the S3 **External ID**. It is derived per
tenant, shown before the destination is saved because the customer needs it to
write the role's trust policy, and ignored if a client sends it back — the value
used with `AssumeRole` is always the server's own.

Every provider also carries an optional `displayName` field under Storage
options. Leave it blank and the server names the destination after what it is:
`Azure Blob Storage · dlp-evidence`.

`isImplemented: false` greys a card. GCS ships that way until it has been run
against a live bucket: `Storage:Providers:Gcs:Enabled` must be the literal
`true` to turn it on, and no environment sets it. Setting it needs no release on
either side.

Cards: `defensely`, `azure-blob`, `s3` (Amazon S3), `gcs`, `s3-compatible`, and
`filesystem` behind the flag. `s3` and `s3-compatible` are two cards over one
object store — the backend keeps them apart with `providerKey` so the dialog can
reopen the right form.

---

## 5. Test and switch

Both take the **same body**, so what is tested is byte for byte what is saved.

```jsonc
POST /storage/test-connection
POST /storage/switch
{
  "providerKey": "azure-blob",
  "displayName": "Acme evidence",
  "authMode": "account-key",

  "azureAccountName": "acmedlp",
  "containerName": "dlp-evidence",
  "azureAccountKey": "…",            // plaintext; encrypted server-side

  "folderPrefix": "defensely/",
  "downloadLinkExpiryMinutes": 60,

  "storageTargetId": null,           // set when editing a saved destination
  "copyExistingFiles": false,        // switch only
  "testBeforeSwitch": true           // switch only; default true
}
```

Per provider:

| Card | Body |
|---|---|
| `azure-blob` | `azureAccountName` + `containerName` + one of `azureAccountKey`, `azureSasToken`, `azureConnectionString` |
| `s3`, `authMode: "access-keys"` | `s3BucketName`, `s3Region`, `s3AccessKey`, `s3SecretKey` |
| `s3`, `authMode: "iam-role"` | `s3BucketName`, `s3Region`, `s3RoleArn` — no keys, and **do not** send the external id |
| `s3-compatible` | the access-key set plus `s3Endpoint`, and optionally `s3ForcePathStyle` |
| `gcs` | `gcsBucketName`, `gcsProjectId`, `gcsServiceAccountJson` (the key file's contents, verbatim) |
| `defensely` | nothing |

### The IAM role

For a SaaS deployment this is the only role form that works: the API host runs
in Defensely's AWS account, so its own instance profile reaches nothing of the
customer's. The customer creates a role in **their** account and Defensely
assumes it through STS, with credentials refreshed as the session expires —
nothing long-lived is stored.

The external id is what makes that safe. Without it, anyone who learns the role
ARN and can get Defensely to assume a role of their choosing reaches that
customer's bucket through us. So:

1. `GET /storage/providers` returns the tenant's external id on the `iam-role`
   mode as a `readonly` field.
2. The admin pins it in the role's trust policy as a condition on
   `sts:ExternalId`, and allows this deployment's principal.
3. `POST /storage/test-connection` proves it. A role that does not pin the
   value fails at `AssumeRole`, and the `write` check reports it.

The value is stable per tenant and derived, not stored, so it can be shown
before any destination exists. It is not a secret — pinning it is what makes it
work.

**Editing a saved destination.** Send its `storageTargetId` and leave secrets
blank — the stored ones are used. That is what makes "credentials are never shown
again" compatible with changing a folder prefix.

**Back to managed:** `{ "providerKey": "defensely" }`, or
`POST /storage/disconnect?copyExistingFiles=false`.

### Test result

A refused account is **200 with `success: false`**. A wrong key is a result the
dialog renders, not a server error.

```jsonc
{
  "success": false,
  "provider": "AzureBlob", "providerKey": "azure-blob",
  "container": "dlp-evidence",
  "targetUrl": "https://acmedlp.blob.core.windows.net/dlp-evidence/defensely/",
  "probeObjectName": "_defensely-connection-test/20261113T041022-7f3c….txt",
  "durationMs": 812,
  "checks": [
    { "step": "resolve", "label": "Find the endpoint",  "ok": true,  "durationMs": 12 },
    { "step": "write",   "label": "Write a test file",  "ok": true,  "durationMs": 540 },
    { "step": "read",    "label": "Read it back",       "ok": false, "durationMs": 272,
      "error": "The credentials for 'acmedlp' were accepted but do not allow this. Grant read, write, delete and list on the container.",
      "rawError": "This request is not authorized to perform this operation using this permission. ErrorCode: AuthorizationPermissionMismatch" }
  ],
  "message": "Wrote a file but could not read it back.",
  "error": "The credentials for 'acmedlp' were accepted but do not allow this…"
}
```

Steps run in order: `resolve` → `write` → `read` (bytes compared) → `delete` →
`list`. `list` is **not fatal**: usage figures need it, writing evidence does
not. `resolve` is present only where a wrong name really is a name that does not
resolve — an Azure account or a typed S3 endpoint — and absent for GCS, which is
one fixed Google endpoint whatever the bucket, and for the on-premises store.

**`error` is written for the admin; `rawError` is for a support ticket.** Render
`error` as it stands. The provider's own text goes in `rawError`, untouched, and
should not be shown by default: an SDK that retried writes its retry policy out
once per attempt, which is how a mistyped account name first came back as six
copies of *"Retry failed after 6 tries. Retry settings can be adjusted in
ClientOptions.Retry"* — true, and about our HTTP client rather than their
account.

A step is capped at **8 seconds** and the lookup at 3, so a bad destination
fails in seconds rather than spending each SDK's full retry budget. A
destination that does not resolve is reported before anything is written:
`Storage account 'dfnsprobe0000zz' was not found. Check the name for a typo.`

`targetUrl` is the resolved address data will land at —
`https://<account>.blob.core.windows.net/<container>/<prefix>`,
`s3://<bucket>/<prefix>`, `gs://<bucket>/<prefix>`, or the endpoint spelled out
for a non-AWS S3 store. Built server-side from the same values the save will
write, so the line the admin confirms is the line the server acts on. It is
`targetUrl`, not `target`: `target` already means the account or bucket name on
a destination object, and one name per meaning is worth more than matching a
draft.

### Switch result

```jsonc
{
  "switched": true,
  "destination": { /* the StorageDestination above */ },
  "test": { /* the probe, when one ran */ },
  "copy": null,            // a job when copyExistingFiles was true
  "message": "Switched. New data goes to this destination right away; files already written stay readable from where they are."
}
```

`switched: false` with `testBeforeSwitch` left on means the probe failed and
**nothing was changed** — `destination` is still the old one. Show
`test.checks` and keep the dialog open.

**A copy that cannot start never fails the switch.** By the time the copy is
queued the destination row is already saved and new data is already going to
the new place, so throwing here produced a `500` on a call that had in fact
switched — the caller was told it failed while the tenant's storage had moved.
Now `switched` stays `true`, `copy` comes back `null`, and `message` says why:
*"…The copy could not be started (reason) — nothing was moved. Start it again
from the Storage page."* Render the message; the copy can be retried from
`POST /storage/copy` without redoing the switch.

**Usage is discarded on every switch.** The figures described the account that
was just left, so quoting them against the new destination made an empty bucket
report 11.3 GB and offered to copy a size that belonged somewhere else. After a
switch `usage.isMeasured` is `false` until the next sweep or a
`POST /storage/usage/refresh`. Show "not measured yet" rather than zeros.

**Switching away keeps the connection's credentials.** It did not always, and
the change matters: every object records the destination it was written to and
is read back with **that** destination's credential. Erasing on a switch made
every video and shadow copy written beforehand unreadable the moment the switch
completed, while the files sat untouched in the customer's own bucket. A
completed copy used to erase its source for the same reason, and no longer does.

So `hasCredentials` stays `true` across a switch, the non-secret values stay in
`values` for the Edit form, and files written to a destination the tenant has
left keep working. A credential now ends only when an admin ends it, one
destination at a time, having been told what stops being readable:

| | Keeps the row | Keeps the credential | Objects still readable |
|---|---|---|---|
| Switch away | yes | **yes** | **yes** |
| `POST …/forget` | yes | no | no |
| `DELETE …` | no | no | no |

> Credentials already erased by the old behaviour do not come back. Those rows
> read `hasCredentials: false`, and S3 rows are missing `values.s3AccessKey`,
> until the destination is reconnected with its secret.

**To erase one destination's credentials but keep its history:**

```
POST /storage/destinations/{storageTargetId}/forget      (STORAGE EDIT)
```

Clears the account key, SAS token, connection string, S3 access and secret
keys, GCS service-account JSON and on-premises signing key from **that one**
destination, and returns the configuration afterwards. The identity — provider,
bucket, container, prefix — is kept, so `previous[]` and the audit history
still answer "where did this tenant's evidence live, and between which dates".
`hasCredentials` becomes `false`.

The **active** destination is refused with `404`: erasing the key for storage
being written to right now would break the tenant immediately. Switch away
first.

> **Afterwards, files still in that account are unreachable.** The read path
> finds objects written before a switch by walking retired destinations, and it
> cannot do that without a credential. **Copy anything you still want first** —
> and confirm the copy finished, not just that it started.

### Removing a destination entirely

```
DELETE /storage/destinations/{storageTargetId}?confirm=true      (STORAGE EDIT)
```

Removes the connection row **and** its credential. Stronger than `…/forget`,
which keeps the row for the history.

> **It does not delete anything from the customer's bucket.** This product never
> deletes a customer's objects and this endpoint does not start. Every file stays
> exactly where it is, in the account the customer owns, reachable with their own
> tools. What goes is Defensely's route to it.

**Two steps on purpose.** Without `confirm=true` nothing is deleted and the
response reports what it would cost; with it, the row goes.

```jsonc
{
  "storageTargetId": 21,
  "displayName": "Azure Blob Storage · dlp-connecter",
  "deleted": false,
  "objectsRecorded": 412,          // null = the count could not be taken
  "refused": "confirm",            // managed | active | copying | confirm | gone
  "message": "412 objects written to … will stop being readable. Nothing is deleted from your storage …"
}
```

| State | `refused` | Status |
|---|---|---|
| The destination in use | `active` | **409** — switch away first |
| A copy is reading from it | `copying` | **409** — stop it or wait |
| Holds objects, no `confirm` | `confirm` | **409** + `objectsRecorded` |
| `?confirm=true` | — | **200**, `deleted: true` |
| Not this tenant's id | — | **404** |

`active` outranks `copying`: sending someone to stop a copy that still would not
let the delete through wastes their time.

**`objectsRecorded: null` means the count failed, not that there is nothing
there** — most likely because migration `V0041` has not run on that database.
The delete then asks for `confirm=true` anyway rather than treat an unanswerable
question as an all-clear.

The `StorageUploadRecord` rows are deliberately **kept**: what was written and
when outlives the account it went to. Those records then resolve to "not found",
which is the truth once the credential is gone.

### Asking first: `canRemove`

Every destination in `current`, `previous[]` and `purposeOverrides[]` carries the
verdict, so no client has to keep its own copy of the rules above — the flag and
the `409` come from the same function, and a button that is enabled cannot hit a
refusal.

```jsonc
{
  "canRemove": true,
  "removeRequiresConfirm": true,   // send ?confirm=true
  "removeReason": "412 objects written to … will stop being readable. Nothing is deleted from your storage …",
  "objectsRecorded": 412           // null = count unavailable; Defensely Storage is always null
}
```

`canRemove: false` comes with the reason to show: Defensely Storage has no
connection to remove, the current destination must be switched away from, a
destination a copy is reading from must wait. `removeReason` is written to be
displayed as-is.

> **If `removeRequiresConfirm` is `false` on every row**, check `objectsRecorded`.
> `null` means the count could not be taken — run `V0041`. `0` means the column
> exists and nothing has been recorded against that destination yet: only
> uploads issued *after* V0041 ran carry a destination id, so a destination that
> predates the migration genuinely counts zero. Older objects are unaffected
> either way; they simply do not produce a confirmation step.

**Reconnecting to a saved destination reuses its row.** Sending
`storageTargetId` for a destination whose provider, account/bucket, container
and folder prefix are unchanged **reactivates** that row instead of cloning it,
so each connection appears once in `previous[]`. Change where it points — a
different bucket, prefix or container — and it is a new destination with its
own row, because the history has to say which account held the evidence and
when. Renaming it, or changing the link expiry, is not a move.

`400` is only for a malformed body: an unknown provider, a missing required
field, a bad folder prefix, an expiry over 7 days, or a host with no
`Storage:CredentialEncryptionKey` configured. The `error` array carries one
sentence per problem, written for the admin.

---

## 6. Copying existing files

Switching is instant and safe; copying terabytes is neither. So it is a job with
a cursor, and it **never deletes from the source** — a copy that goes wrong costs
duplicate storage and nothing else.

```jsonc
POST /storage/copy
{
  "sourceStorageTargetId": 14,           // null = Defensely Storage; omit = choose for me
  "categories": ["ShadowFile", "Video"]  // omit for everything copyable
}

GET  /storage/copy/current
POST /storage/copy/run                   // advance one batch
POST /storage/copy/{id}/cancel
GET  /storage/copy/history?take=10
```

**A copy always writes to the current destination.** `sourceStorageTargetId`
says where it reads, and has three distinct meanings:

| Sent | Means |
|---|---|
| a `storageTargetId` | that destination |
| `null`, explicitly | Defensely-managed storage, which has no row |
| property omitted | the destination most recently left — what "copy my files across" means right after a switch |

The explicit null matters: a tenant who moved to their own bucket and wants
their old managed data brought over has no id to send. `fromStorageTargetId` is
still accepted as the previous name, but cannot express Defensely Storage.

**Omit `categories` to copy everything that can be copied** — shadow files,
videos and activity logs. The server decides what is copyable; a client should
not have to know that report exports and discovery results are regenerated from
the database rather than moved. Name categories only to narrow it. A list naming
nothing copyable is a `400`, not a job that copies nothing.

**`400`, not a doomed job**, when the source cannot be read — chiefly when its
credentials were erased on disconnect. The message is written for the admin:
*"…has no saved credentials, so its files cannot be read. Reconnect that storage
with its key, then start the copy."* Queuing it anyway is what produced a job
that died on its first batch with the provider's own error.

**One copy at a time.** If a job is already `Queued` or `Running`, the call
returns **that** job with `200` rather than starting a second — two jobs reading
one source and writing one destination would double the egress bill to achieve
nothing.

### Which sources are available: `copySources[]`

`GET /storage/configuration` carries the list, so the dialog does not have to
work it out from `previous[]`:

```jsonc
"copySources": [
  { "storageTargetId": null, "displayName": "Defensely Storage",
    "targetUrl": null, "usedBytes": 11851927369, "objectCount": 7002,
    "measuredUtc": "2026-11-17T04:10:22Z",
    "estimatedDurationSeconds": 1688, "canCopy": true, "reason": null },

  { "storageTargetId": 14, "displayName": "QA",
    "targetUrl": "s3://dlptest-inn/QATesting/",
    "usedBytes": null, "objectCount": null, "measuredUtc": null,
    "estimatedDurationSeconds": null,
    "canCopy": false,
    "reason": "No saved credentials — reconnect this storage with its key to copy from it." }
]
```

One entry per **connection**, not per row, and the current destination is
excluded — copying a place onto itself does nothing. A source that cannot be
read is listed with `canCopy: false` and a reason, rather than hidden, so an
admin can see it exists and why it is unavailable.

> **`usedBytes: null` means "nobody has counted", not "empty".** Render null as
> "size unknown" — showing `0 B` would read as "nothing to copy" when the truth
> is that nothing has measured it.

**Two places are measured: the current destination, and Defensely Storage.** The
sweep walks managed storage for every purpose whenever that is not where the
tenant currently writes, because a tenant who connects their own bucket does not
move what is already in Defensely — the videos, shadow copies and logs written
before the switch stay there. That is what fills the `storageTargetId: null`
row above. Destinations the tenant has *left* are still unmeasured: walking
every account a tenant has ever used would cost a full listing of each on a page
that is usually just being looked at.

The same split is on `usage`, if you want it directly:

| | |
|---|---|
| `usage.destinationBytes` / `destinationObjectCount` | in the current destination — the capacity bar's numerator |
| `usage.elsewhereBytes` / `elsewhereObjectCount` | outside it, i.e. still in Defensely Storage |
| `usage.elsewhereMeasured` | whether anything outside was actually counted |

> **If the Defensely row reads null, nothing has swept yet.** Figures are
> discarded on every switch and the background tick re-measures at most every
> six hours (`Storage:UsageSweep:MaxAgeHours`). Call
> `POST /storage/usage/refresh` and re-read the configuration. If `usage` has no
> `elsewhereMeasured` field at all, the deployed build predates this and the
> sweep is still walking only the destination.

`estimatedDurationSeconds` follows `usedBytes` exactly: **null wherever the size
is null**, because an estimate needs something to estimate from. See
*How long will this take* below for what the number means.

```jsonc
{
  "storageCopyJobId": 14, "status": "Running",      // see "What a finished copy says" below
  "from": "Defensely Storage", "to": "Acme evidence (AzureBlob/dlp-evidence)",
  "fromStorageTargetId": null, "toStorageTargetId": 17,   // null = Defensely Storage
  "categories": [ "ShadowFile", "Video", "ActivityLogs" ],
  "objectsCopied": 41201, "objectsSkipped": 118, "objectsFailed": 3,
  "bytesCopied": 88213452800,
  "estimatedObjects": 1642871, "estimatedBytes": 386547056640,
  "percentComplete": 2.51,
  "estimatedRemainingSeconds": 162540,              // "about 2 days left"
  "currentCategory": "ShadowFile",
  "startedUtc": "…", "heartbeatUtc": "…", "finishedUtc": null,
  "lastError": "While writing to Acme evidence (S3/dlptest-inn): The credentials for 'Acme evidence (S3/dlptest-inn)' were accepted but do not allow this. Grant read, write, delete and list on the container.",
  "lastErrorRaw": "Forbidden",                      // the provider's own words, for a ticket
  "recentErrors": [ "2026-11-18T04:22:01Z While writing to …: …" ]   // DISTINCT reasons
}
```

### What a finished copy says

`status` is one of **`Queued`**, **`Running`**, **`Completed`**,
**`CompletedWithErrors`**, **`Cancelled`**, **`Failed`**.

> **`CompletedWithErrors` is new, and it is the fix for QA job 4.** That job
> copied 197 objects, failed 7,793, and reported `Completed` — because reaching
> the end of the walk was treated as success on its own, whatever the counters
> said. A status that contradicts its own numbers wins, because it is the thing
> that goes in a badge.

| Counters at the end | `status` |
|---|---|
| nothing failed | `Completed` |
| some copied, some failed | `CompletedWithErrors` |
| nothing copied at all, something failed | `Failed` |

No threshold — "mostly worked" is not a judgement the server can make for a
customer, since one shadow copy left behind may be the one that matters.
`objectsFailed` carries how much.

**Render anything you do not recognise as finished-and-not-clean, never as
success.** `CompletedWithErrors` does not hold the switch lock and does not
block a new copy: it is over, and starting another retries what failed (objects
already at the destination are skipped).

This is **not retroactive.** Job 4's recorded status stays `Completed` on QA —
rewriting history is worse than a stale row. Its counters were always right.

### Error text

`lastError` is now an explained sentence that names **the stage and the end that
refused**, mapped by the same code behind the Test connection button, so one
provider code reads the same way in both places:

```
While writing to Acme evidence (S3/dlptest-inn): The credentials for
'Acme evidence (S3/dlptest-inn)' were accepted but do not allow this.
Grant read, write, delete and list on the container.
```

The bare word `Forbidden` is what job 4 recorded 7,793 times, and it could not
say whether Defensely Storage was refusing the **read** or the customer's bucket
was refusing the **write** — opposite fixes, and the error text was the only
evidence. Reads, writes and the destination's existence check are now caught
separately.

`lastErrorRaw` keeps the provider's untouched text for a support ticket. **Not
for the card** — it is long and names our own client options. One sample, not
one per object.

`recentErrors[]` now holds **distinct reasons, newest first — not one entry per
failure.** It was every failure capped at 50, so a bulk failure filled all fifty
slots with the same sentence and the one question the list exists to answer —
*is this one problem or several?* — was the one it could not. `objectsFailed`
carries the volume; `recentErrors` carries the variety.

> **A copy that left objects behind no longer erases the source's credentials.**
> Erasing is justified by the files being safely at the destination; for job 4
> that was untrue of 7,793 of them, and erasing would have made the only
> remaining copy unreachable for good. Fix the cause, run the copy again, and
> the keys go when it finishes clean.

* **One job at a time**, enforced by a partial unique index, not by a check in
  code. Starting a second returns the running one.
* **Driving it.** `POST /storage/copy/run` runs one bounded batch inside the
  request (200 objects or 50 seconds, configurable) — call it on a timer from the
  console while the admin watches. Unattended, the scheduler tick works through
  it (§8).
* **Resumable.** The cursor is a category plus the last object name. A restart, a
  deploy or a cancel loses nothing.
* `percentComplete` is capped at 99 until the walk ends: it is measured against
  the usage estimate taken when the job started, and files keep arriving.
* **Scope.** Shadow copies, videos and activity logs. Report exports and
  discovery results are not copied — they are regenerated from the database on
  demand, so moving them costs transfer and buys nothing.
* Objects over `Storage:Copy:MaxObjectBytes` (1 GiB default) are counted as
  failures with a plain reason rather than copied: only the Azure store streams a
  read, the others buffer, and a 20 GB video would take the API host down.

### How long will this take

Two fields answer it, and both are **estimates that are allowed to be wrong**.
They exist so an admin can tell "this finishes while I wait" from "this runs all
weekend" *before* pressing the button.

| Field | Where | Means |
|---|---|---|
| `usage.estimatedDurationSeconds` | `GET /storage/configuration`, `GET /storage/usage` | copying what is stored now — the switch confirmation's "Estimated time: about 1 hour" |
| `copySources[].estimatedDurationSeconds` | `GET /storage/configuration` | copying from that one source — the copy dialog's line |
| `estimatedRemainingSeconds` | any copy job | how much longer a running job has — "About 40 minutes left" under the bar |

The first two share a name on purpose: same quantity, same arithmetic. (It was
`usage.estimatedCopySeconds` in the 18 Nov build — renamed, and the old name is
gone rather than kept as an alias, because two names for one number is an
invitation to read the wrong one.)

**`estimatedRemainingSeconds` is sent on `Queued` and `Running` jobs, and is
null on finished ones.** A `Queued` job has not started and still answers, from
the configured rate — the card appears the moment the job is created. That is
why it was absent on QA job 4: the job had finished, which is the one state
where there is no time remaining to report.

**Round it and say "about".** Show `about 2 hours`, not `1h 54m 12s`, and never
count it down with a timer: it is recomputed from what is left each time you
poll, so it moves — sometimes upwards, when the estimate meets reality.

The figure is **size plus round trips, added**, not size alone:

```
seconds ≈ bytes / Storage:Copy:EstimatedBytesPerSecond      (default 12 MB/s)
        + objects × Storage:Copy:EstimatedSecondsPerObject  (default 0.1 s)
```

> **Expect large numbers, and show them.** A copy moves one object at a time
> between two clouds, and each one costs an existence check plus an upload. Two
> million small shadow files is about 2 GB — half an hour of bandwidth — but
> **55 hours** of round trips. The per-object term is what dominates every real
> tenant's estimate, and a dialog that promises "about 30 minutes" there is how
> a customer finds out two days later.

`estimatedRemainingSeconds` is computed from what is **left**, and switches to
the job's **own measured throughput** once it has moved 64 MB over at least 30
seconds — a tenant's actual link is the only honest input available. Before
that, and for a job that has not started, it uses the configured rate.

It is **null** for a job that has ended, and for one whose size was never
estimated. That second case is why `percentComplete` alone cannot answer this: a
percentage has no idea whether the remaining 40% is one video or four million
small files.

### Switching is locked while a copy runs

`POST /storage/switch` and `POST /storage/disconnect` answer **409 Conflict**
while a copy is `Queued` or `Running`:

```jsonc
{ "success": false, "message": "Storage cannot be changed right now.",
  "errors": [ "A copy is running. Stop it or wait for it to finish before changing storage." ] }
```

A conflict rather than a 400: nothing about the request is wrong, and the same
request works once the copy stops. **Nothing is written** — the guard runs before
the switch.

`GET /storage/configuration` says so in advance, so the buttons can be disabled
without re-deriving the rule from the job's status:

```jsonc
"switchLocked": true,
"switchLockReason": "A copy is running. Stop it or wait for it to finish before changing storage."
```

Disable **Change storage**, **Edit connection** and **Revert** on `switchLocked`,
with `switchLockReason` as the tooltip. Keep **Stop copy** enabled: stopping is
safe, and a later copy resumes from the cursor rather than walking the account
again.

> **Confirmed: cancelling releases the lock at once.**
> `POST /storage/copy/{id}/cancel` sets the status to `Cancelled`, and only
> `Queued` and `Running` hold the lock. There is no sweep in between, so the
> admin can switch immediately after pressing Stop. The worker notices within one
> batch and stops moving files.

`POST /storage/destinations/{id}/forget` also answers **409** while a copy is
reading from *that* destination — erasing the key mid-walk kills the copy with
the provider's own "access key does not exist" as the only clue. Any other
destination's credentials can still be erased. Match a job to a source with
`fromStorageTargetId`, not with the `from` label, which is only a display name.

> The server enforces all of this. The disabled buttons are for the admin's
> benefit; a second tab, a stale page or a direct call reaches the endpoint with
> the buttons still enabled in its own browser.

---

## 7. Security

* Secrets arrive in plaintext **once**, on `test-connection` and `switch`, and
  are encrypted with `SecretProtector` (AES-256-GCM) under
  `Storage:CredentialEncryptionKey` (falling back to
  `AgentCredentials:EncryptionKey`) before they reach the database. Without a key
  configured, the save is **refused** rather than writing a credential in the
  clear.
* **No endpoint returns a credential** — not masked, not partial. A saved
  destination reports `hasCredentials: true` and nothing else.
* Request bodies on this controller are **never** written to the action log. Only
  a provider / container / options summary is.
* `GcsServiceAccountJson` is a private key. Treat a leak of
  `dlp."StorageTarget"` as a leak of the customer's bucket.
* Shadow copies are sealed with the tenant's own KEK (`/storage/shadow-keys`)
  **before** they are written, whatever the destination. Connecting your own
  storage does not change that, and Defensely cannot read a shadow file out of a
  customer's bucket without the tenant key.

### Azure SAS-token mode, honestly

With a SAS and no account key, Defensely **cannot sign anything new** — a SAS
cannot mint another SAS. Download and upload URLs are the customer's token
appended to the blob URL, so they inherit **its** permissions and **its** expiry,
not `downloadLinkExpiryMinutes`, and the per-file narrowing an account key gives
is lost. The dialog should say that where it asks for `sp=rwdl`. Account key or
connection string is the better default.

---

## 8a. Production rollout

Run `scripts/prod_storage_rollout.sql`. It covers the permission catalog for
the master common database and, more importantly, **which roles may see and
change storage** — the one thing the migrations cannot decide.

**The role name is the trap.** `Common/V0008` grants to a role named exactly
`Admin`. On QA the administrator role is `DLP Admin`, so that grant matched
nothing and the permission had to be ticked by hand. The API still accepted the
calls, because an Admin *role* is accepted while the seed rolls out — but the
console builds its menu from the token's `rolePermissions`, so the Storage page
simply never appeared. It looks exactly like a failed deployment. Part 2 STEP 1
of the script lists the real role names; use those.

**Before the first tenant connects anything:**

| Setting | Why it matters in production |
|---|---|
| `Storage:CredentialEncryptionKey` | **Mandatory.** Without it `POST /storage/switch` is refused rather than writing a credential in the clear. It must be byte-identical to the key DLPProcessLaterService uses, or cloud-connector secrets saved by the API cannot be decrypted by the worker. Never rotate it without re-saving every destination. |
| `Storage:Providers:Gcs:Enabled` | Leave **unset**. GCS has never run against a live bucket; absent means the card is greyed. |
| `Storage:SetupGuideBaseUrl` | Unset means every `setupGuideUrl` is null and the console hides the link. Set it only when the docs site is live. |
| `Storage:AzureBlob` AccountName + Key | Must be the **same account's** pair. `AzureCredentialPairing` now stops a name from one layer being matched with a key from another, but a mismatched pair in configuration itself is still a mismatched pair. |
| `Storage:Managed:Region` / `EncryptionText` / `RetentionText` | What the "Stored in" card says about Defensely-managed storage. Wrong text here is a support ticket, not an outage. |

**Then:**

1. **APIM import.** Every `/storage/*` route is new and returns 404 at the
   gateway until `azure-pipelines-dlp-apim.yml` has been run.
2. **Check the timer.** `StorageMaintenanceScheduler` in LogTrigger calls
   `storage/run-scheduled` hourly; without it, copies and usage figures only
   move while someone has the page open.
3. **Tell support what has not changed.** The agent still uploads video
   directly to the configured video account rather than through
   `/storage/upload-url`, so **video will not follow a tenant's chosen
   destination** until the agent is changed (see §9). Shadow copies already go
   through the API and do follow it. A customer who connects their own bucket
   and expects recordings to appear there will be disappointed, and that is a
   product expectation to set, not a bug to chase.
4. **Nothing is configured on day one.** An empty `dlp."StorageTarget"` is the
   correct state: every tenant keeps the storage it already had, and
   `current.isChosen` is `false` until someone picks.

## 8. Deployment

1. **Migrations**, all applied automatically per tenant by
   `MigrationRunnerService`:
   - `Dlp/V0032__tenant_storage_destination.sql` — destination columns on
     `dlp."StorageTarget"`, plus `dlp."StorageUsageSnapshot"` and
     `dlp."StorageCopyJob"`.
   - `Dlp/V0033__storage_destination_s3_role.sql` — `S3RoleArn`,
     `S3ExternalId`, `S3ForcePathStyle`.
   - `Common/V0008__storage_permission_module.sql` — the `STORAGE` module with
     `VIEW` and `EDIT`, granted to Admin. This one runs against the tenant's
     **user** database, which is why it is in the Common set.
   - `Dlp/V0040__storage_copy_last_error_raw.sql` — `LastErrorRaw` on
     `dlp."StorageCopyJob"`, and widens `Status` from `VARCHAR(20)` to `(40)`
     because `CompletedWithErrors` fit the old one by a single character.
     **Tick `runMigrations` on the QA pipeline run that carries this**, or the
     column is missing and the copy endpoints 500.
   - `Common/V0009__storage_permission_superadmin.sql` — the same two
     permissions for `SuperAdmin`. A separate file because V0008 has already
     been applied on QA and the runner will not re-run a recorded version.
     Note it does **not** filter on `Role.IsActive`: the SuperAdmin role is
     seeded inactive on purpose, so that filter would make it a no-op.
2. **APIM.** New routes 404 at the gateway until
   `azure-pipelines-dlp-apim.yml` is run by hand. Every route in §2 is new.
   **Still outstanding.**
3. **The copy worker — `StorageCopyWorker` in `DLPProcessLaterService`.** This is
   what actually moves the bytes. It polls every tenant, runs batches
   back-to-back until each tenant's job ends, and needs no browser open and no
   timer.

   It is in the worker rather than the API for two reasons. A batch streams
   objects between two clouds for the best part of a minute, which a
   request-serving container app should not be doing. And before it existed, a
   copy advanced **only** while an admin had the Storage page open or the hourly
   tick fired — unattended, that is one 50-second batch per hour, about four
   minutes of copying a day. That is why a QA job appeared never to finish.

   It needs, in the worker's configuration:
   - `ConnectionStrings:NucleusConnection` — already required for the cloud
     crawl; without it neither worker is registered.
   - **`Storage:CredentialEncryptionKey`** (or `AgentCredentials:EncryptionKey`)
     — *the same value the DLP API uses*. Without it the worker hands the
     provider ciphertext and every copy involving a customer's own storage fails
     to authenticate. It says so loudly on stderr at startup.

   `Storage:Copy:WorkerEnabled=false` turns it off. The API keeps
   `POST /storage/copy/run` either way, so the console can still drive a batch
   while an admin watches; running both is safe, since a batch skips objects the
   other already copied and the cursor only moves forward.
4. **Scheduler.** `StorageMaintenanceScheduler` in `SFM.DLP.LogTrigger` calls
   `POST /storage/run-scheduled` hourly, per tenant, with the `ClientGUID`
   header and no token — the same shape as `DiscoveryScanScheduler`. It runs one
   copy batch and re-measures usage only when the snapshot is older than
   `Storage:UsageSweep:MaxAgeHours`, so most ticks are a no-op. With the worker
   deployed this is a floor, not the engine.
5. **Configuration** (all optional):

   | Key | Default | What it does |
   |---|---|---|
   | `Storage:Managed:QuotaBytes` | 1 TiB | the allowance shown for managed storage |
   | `Storage:Managed:QuotaBytes` | — | the allowance for Defensely-managed storage. **Unset means no allowance and no bar** — `quotaBytes`, `freeBytes` and `usedPercent` all come back null. There is deliberately no default: a hardcoded 1 TB told every new tenant it had "1 TB free" when nobody had agreed any such figure. |
   | `Storage:Managed:Region` | — | "Azure Blob · **Central India**" |
   | `Storage:Managed:EncryptionText` | AES-256 at rest · Defensely-managed keys | the Encryption line |
   | `Storage:Managed:RetentionText` | Follows your plan and add ons | the Retention line |
   | `Storage:DownloadUrlExpiryMinutes` | 60 | default link lifetime (the API field is `downloadLinkExpiryMinutes`) |
   | `Storage:SetupGuideBaseUrl` | — | absolute `https://` base for `setupGuideUrl`; unset means every card returns null |
   | `Storage:UsageSweep:BudgetSeconds` | 120 | time bound on a measurement |
   | `Storage:UsageSweep:MaxObjects` | 5,000,000 | object bound on a measurement |
   | `Storage:UsageSweep:MaxAgeHours` | 6 | how stale figures may get before the tick re-measures |
   | `Storage:Copy:BatchObjects` | 200 | objects per copy batch |
   | `Storage:Copy:BatchSeconds` | 50 | seconds per copy batch |
   | `Storage:Copy:MaxObjectBytes` | 1 GiB | per-object copy limit |
   | `Storage:Copy:WorkerEnabled` | **true** | the `DLPProcessLaterService` copy worker. False leaves copies advancing only on the hourly tick |
   | `Storage:Copy:EstimatedBytesPerSecond` | 12,000,000 | bandwidth assumed for `estimatedCopySeconds` and friends |
   | `Storage:Copy:EstimatedSecondsPerObject` | 0.1 | per-object round-trip cost assumed for the same. **This is the term that dominates** — see §6 |
   | `Storage:Providers:Gcs:Enabled` | **false** | the GCS card is greyed (`isImplemented: false`) unless this is the literal `true`; absent, blank or unparseable all mean off |

---

## 9. What this governs today, and what it does not

**Governed by the destination** (everything resolved through
`StorageTargetResolver`): shadow-copy and video upload URLs
(`POST /storage/upload-url`), shadow-file downloads, and the usage and copy
machinery here.

**Not yet governed:** activity-log payloads, report exports, MSI installers and
feedback attachments still resolve storage through `IBlobServices` /
`ITenantObjectStore`, which read the tenant's `ClientSetting` rows and the
deployment's configuration. Migrating those call sites is a separate change, and
it needs a read path that falls back across destinations so files written before
the switch stay readable.

Because of that, the **Activity Logs** figure is measured in *both* places — the
destination and the managed account — and summed. That stays correct once the
pipeline moves, and after a copy: a file copied forward genuinely occupies two
accounts until someone deletes the original.

### The selection governs

**Switching retires every per-purpose row.** What the admin picks is what every
purpose uses — a screen that says "Amazon S3" while video is still written to an
Azure account pinned years ago is a screen that lies. `POST /storage/switch`
retires `Video`, `ShadowFile` and any other purpose row, and the result message
names what moved: *"Switched. Forensic Video Evidence and Shadow File Copy now
go here too…"*.

`purposeOverrides[]` is therefore normally empty after a switch. It stays in the
contract because a row can still be created directly through the older
`POST /storage/targets/save`, and because a tenant who has never used this
screen may already have one.

Resolution order, for writes:

1. the active row for that purpose — `Video`, `ShadowFile`;
2. the active `All` row — the destination set on the Storage screen;
3. the tenant's ClientSetting storage rows;
4. the deployment's configuration.

### Reads do not resolve forwards

Writing takes the newest destination. **Reading has to find the file.** Evidence
written last year is still in the account it was written to; the copy job is
optional and not instant. So `ResolveForExistingObjectAsync` walks candidates
newest first — current destination, then the container recorded at write time,
then destinations this tenant has retired, then managed storage — and the first
that actually holds the object answers. One existence check per candidate, and
the common case hits the first.

That is what makes retiring the overrides safe: playback and shadow-file
download follow the file, not the configuration.

`source` on the upload-url response now says **which** row answered:

| `source` | Means |
|---|---|
| `PurposeOverride` | a row pinning this purpose to its own account — outranks the destination |
| `Destination` | the `All` row, i.e. what the Storage screen shows |
| `ClientSetting` | the tenant's Nucleus storage rows |
| `Configuration` | the deployment's own settings — Defensely-managed storage |

Both of the first two reported `StorageTarget` until this was split, which made
"I connected S3 but the API says AzureBlob" impossible to answer from the
response alone.

**To move a purpose onto the destination, retire its row** — `purposeOverrides`
then empties and that purpose resolves to the destination. For `Video`, read the
next section first: the upload path would follow the destination and the
playback path would not.

### The two download paths, specifically

| Endpoint | Resolves through | Follows the destination? |
|---|---|---|
| `GET /incidents/{metaId}/shadow-file` | read-path walk | **Yes** |
| `POST /GetVideoUrlsById/{enrichedId}` | read-path walk, then legacy | **Yes** |

**A log row does not always carry the object name.** The upload API answers
with `objectName` — `video/endpoint/232/{guid}/Video_39_…mp4` — but the agent
records only `Video_39_…mp4` against the log. A bare name cannot be found at a
destination that stored it under a device-scoped path, so playback used to fall
through to the legacy path, which mints a URL **without checking the object
exists**: a link that looks right and returns `404 BlobNotFound`. Observed on QA
with `GetVideoUrlsById/205235` while both files sat in S3.

`GetVideoUrlsById` therefore has a second step. When a name is not found as
given, it asks `dlp."StorageUploadRecord"` what that file was stored as — the
row was written when the URL was issued, and it is the one place that still
holds the mapping the log row lost — then resolves that object name instead.
**The agent should still record the returned `objectName`**; this makes the
rows already written work, it is not a reason to keep sending the file name.

`GetVideoUrlsById` resolves each name against where it actually is, and mints
the link from the store that holds it. Only names **no** destination holds fall
through to the two legacy paths — the `IsAmazonBlob` ClientSetting branch into
the deployment-wide `AmazonS3Settings` bucket, or `IBlobServices` with the
legacy `VideoStorageAccountName`/`Key` pair. Those mint a URL without checking
the object exists, so they run last and only on what was not found; a deployment
that has never used the Storage screen behaves exactly as it did.

`shadow-file` previously took the container from the upload record but the store
from the *current* target, so the two disagreed the moment a tenant switched. It
now uses the same walk, with the recorded container as the hint.

**Not verified against a live account:**

- **Google Cloud Storage** — implemented against `Google.Cloud.Storage.V1`,
  compiles and unit-tests, never run against a real bucket. The card is greyed
  by default (`Storage:Providers:Gcs:Enabled`); turn it on once
  `POST /storage/test-connection` passes against a real one.
- **The S3 IAM role** — `AssumeRole` with an external id, through
  `AWSSDK.SecurityToken`. The derivation and the form are unit-tested; the
  assume itself needs a real role with a matching trust policy, and the
  deployment's own principal has to be the one that role trusts. Confirm which
  principal that is before writing the setup guide.
