# AI Bake-off — read-only views and column dictionary

Four flattened views on the QA tenant database, plus a dedicated read-only
login. Everything the evaluation needs is on the row: no joins, no application
code, no second data pull.

| | |
|---|---|
| **Schema** | `ai_bakeoff` |
| **Login** | `ai_bakeoff_ro` (password sent separately) |
| **Access** | `SELECT` on the four views below. Nothing else in the database. |
| **Session** | `default_transaction_read_only = on`, `statement_timeout = 120s`, `search_path = ai_bakeoff` |
| **Expires** | The login stops working after **2026-12-31** |
| **Created by** | `scripts/create_ai_bakeoff_views.sql`, then `scripts/create_ai_bakeoff_ro_role.sql` |

```sql
-- unqualified names work; search_path is already set
SELECT "Category", COUNT(*) FROM vw_dlp_events GROUP BY 1;
```

---

## What is deliberately not exposed

No raw captured content, in any view:

- the captured file body (`FileContent`) and captured prompt text (`PromptContent`)
- `PromptMessage` — not persisted at all since 2026-09-16
- shadow-copy and video URLs

`SensitiveContent` **is** exposed. It is the **masked** matched-value JSON —
masking runs before the write on every path, so it carries the shape of what
matched (`{"DlpType": {"PII": "..."}}`) without the values themselves. It is the
classification signal, not a second copy of the data.

---

## 1. `vw_dlp_events` — one row per DLP event

Every event the product passes a verdict on, across all three channels, in one
list. `Category` says which channel a row came from, and some columns only apply
to one of them — the **Channel** column below says which.

- `File` — a file transfer: email, USB, cloud upload, print, …
- `Content` — text or an attachment stopped on its way into an AI assistant
- `Web` — a browsing verdict, or a tenant-account verdict (a decision about an
  *account* on a service, e.g. a personal Outlook address on Microsoft 365)

**Row count** = all crunched file transfers + all AI prompt blocks + all active
web verdicts. One row per event, always.

### Identity

| Column | Type | Channel | Meaning |
|---|---|---|---|
| `EventId` | text | all | Unique across the whole view — `Category` + source id, e.g. `File:84213`. Use this as the primary key, not `SourceId`. |
| `Category` | text | all | Which channel the row came from: `File`, `Content` or `Web`. |
| `SourceId` | bigint | all | The id in the underlying table. Unique **within** a category, not across. |
| `EventTime` | timestamptz | all | When the event happened on the endpoint. |
| `EventDateUtc` | date | all | `EventTime` truncated to a **UTC** day — pre-computed so every query buckets days the same way. |

### Person

| Column | Type | Channel | Meaning |
|---|---|---|---|
| `UserId` | int | all | The registered user this event belongs to. Null when the endpoint account was never mapped to a person. |
| `UserName` | text | all | The person's name. For file rows this is the name as it stood when the transfer happened; for prompt rows it falls back to the Windows account if the user is unmapped. |
| `EmailId` | text | all | The person's registered email address. |
| `EmployeeCode` | text | all | Employer-assigned staff id. |
| `Designation` | text | all | Job title ("Senior Designer"). |
| `Department` | text | all | Department name, resolved from the current user record. |
| `IsUserActive` | boolean | all | Whether that user account is active today. Null if unmapped. |

### Machine

| Column | Type | Channel | Meaning |
|---|---|---|---|
| `EndpointUserId` | int | all | The agent instance that reported the event — one per (machine, signed-in user). |
| `EndpointUserName` | text | File, Web | The OS account name on the machine (`boobalan.r`). |
| `MachineName` | text | all | Hostname of the endpoint. |
| `SerialNumber` | text | all | Hardware serial of the endpoint. |
| `OsName` | text | File, Web | Operating system as the agent reported it. |

### What happened

| Column | Type | Channel | Meaning |
|---|---|---|---|
| `ActivityType` | text | all | The kind of action: `File Transfer`, `AI Prompt`, `Web Access` or `Tenant Access`. |
| `ChannelName` | text | all | The route it took — transfer channel (`Email`, `USB`, …) for files, the AI service for prompts, the site category for web. |
| `ApplicationName` | text | all | The application or service involved — app/device for files, AI provider for prompts, browser for web. |
| `ProcessName` | text | File, Web | The executable that performed the action. |
| `Url` | text | File, Web | Destination URL as the agent saw it. |
| `Domain` | text | Web | Normalised host (`web.whatsapp.com`). |
| `FileFullPath` | text | File | Full source path of the transferred file. |
| `FileName` | text | File | Just the file name, split off the path here so nobody has to parse it. |
| `FileType` | text | File | File type/extension as classified by the agent. |
| `FileSizeBytes` | bigint | File | Size of the transferred file, in bytes. |
| `ContentCategory` | text | all | What kind of content it was — content type for files, `Prompt` (typed) vs `Content` (pasted/attached) for prompts, site category for web. |
| `MatchedRules` | text | Content, Web | The rule labels that fired ("Bank Account, PAN"); the rule name for web rows. Labels only, never values. |
| `SensitiveContent` | text | File, Content | **Masked** matched-value JSON. See the note above. |
| `HasSensitiveMatch` | boolean | all | True when `SensitiveContent` holds at least one match — a ready-made flag so you don't have to probe the JSON. |

### The verdict

| Column | Type | Channel | Meaning |
|---|---|---|---|
| `PolicyId` | bigint | Web | Id of the policy that decided. Only web rows store it. |
| `PolicyName` | text | all | Name of the policy that decided. Null when no policy matched. |
| `PolicyAction` | text | all | The outcome: `Allow`, `Warn`, `Block` or `Override Applied`. Normalised — the same outcome has the same spelling on every channel. |
| `IsPolicyCompliant` | boolean | all | True only when `PolicyAction = 'Allow'`. **A warn is not compliant** — the user was told the policy objected. This is the same rule the product's Policy Compliance tile uses. |
| `Severity` | text | File, Web | How serious the policy rates this event: `Critical`, `High`, `Medium`, `Low`, `None`. |
| `ProcessStatus` | text | File | Where the event is in the processing pipeline, and its investigation/override outcome. |
| `IsOverrideApplied` | boolean | all | A block that was lifted — the user asked for an exception and got it. |
| `IsPendingApproval` | boolean | all | An override has been requested and is still waiting on an approver. |
| `OverrideRequestId` | uuid | Content | The override request raised for a blocked prompt. |
| `RuleSource` | text | Web | What kind of rule decided: `RestrictAll`, `CustomRules`, `Exception`, `NoRestriction`. |
| `IsAppliedToPolicy` | boolean | File | Whether a policy was actually resolved and applied to this transfer. |

### Investigation

| Column | Type | Channel | Meaning |
|---|---|---|---|
| `IsInvestigation` | boolean | File | **The investigation flag** — an analyst moved this event into investigation from the Log Summary. Always false on prompt and web rows: there is no mechanism that moves those into investigation, so this is a real false, not an unknown. |
| `InvestigationCommentCount` | int | File | How many active analyst comments are attached. A rough proxy for how much attention the event has had. |
| `ReviewStatus` | text | File | Triage queue state: null (untriaged), `NeedsReview`, `Ignored` or `Resolved`. Separate from the investigation flag — an event can be triaged without ever being investigated. |
| `ReviewedAt` | timestamptz | File | When it was last triaged. |

### Risk scores

Computed by the product's AI scoring pass. File rows only — prompt blocks and
web verdicts are not scored, so these are null there, which is **not** a zero.

| Column | Type | Meaning |
|---|---|---|
| `RiskScore` | numeric | The final composite risk score for the event. |
| `AnomalyScore` | numeric | How unusual the event was against the baseline. |
| `ContextScore` | numeric | Risk from the user and environment context. |
| `BehaviorScore` | numeric | Risk from the user's own activity pattern. |
| `DestinationScore` | numeric | Risk from where the data was going. |
| `SensitivityScore` | numeric | Risk from how sensitive the content was. |
| `SensitivitySource` | text | What determined sensitivity: `DLP`, `Classification`, `Manual` or `ML`. |
| `IsAIScan` | boolean | Whether the AI scoring pass has run on this row. |

### Other

| Column | Type | Channel | Meaning |
|---|---|---|---|
| `IsDevice` | boolean | File | The transfer went to a physical device (USB, printer) rather than over the network. |
| `DurationSeconds` | int | Web | Time on page, as the agent measured it. Null means not measured — a blocked navigation never rendered. |
| `PageTitle` | text | Web | The page's own title. |

---

## 2. `vw_discovery_files` — data discovery, one row per file at rest

What the endpoint agent found sitting **on disk**, as opposed to `vw_dlp_events`,
which is data **in motion**.

> **Scope:** only sensitive and unclassified files are stored row-by-row. Clean
> files are counted on the volume, never listed. So this view is the at-risk
> inventory, not a filesystem listing, and `COUNT(*)` here is not "files on the
> estate".

| Column | Type | Meaning |
|---|---|---|
| `FileId` | bigint | Primary key — one row per known file path on an endpoint. |
| `EndpointId` | smallint | The machine the file is on. |
| `MachineName` | text | Hostname of that machine. |
| `SerialNumber` | text | Hardware serial of that machine. |
| `OsName` | text | Its operating system. |
| `IsMonitoringEnabled` | boolean | Whether DLP monitoring is currently switched on for that machine. |
| `OwnerUserId` | int | The machine's primary user. |
| `OwnerUserName` | text | That person's name. |
| `OwnerEmailId` | text | That person's email address. |
| `OwnerDesignation` | text | That person's job title. |
| `OwnerDepartment` | text | That person's department. |
| `VolumeId` | bigint | The volume the file sits on. |
| `DriveLetter` | text | Drive letter, or the UNC root for a mapped share. |
| `VolumeKind` | text | `Local`, `External` (USB) or `Mapped` (network share). |
| `FileSystem` | text | `NTFS`, `FAT32`, … |
| `VolumeSizeGb` | numeric | Capacity of the volume, as the agent reported it. |
| `VolumeEncryptionStatus` | text | `Encrypted`, `Partial`, `Suspended` or `Unencrypted` — the exposure multiplier on a sensitive file. |
| `VolumeEncryptionMethod` | text | The cipher in use (`XTS-AES 256`). |
| `VolumeLastScanAt` | timestamptz | When that volume was last scanned — how stale this row is. |
| `Directory` | text | Folder holding the file (`D:\Finance\Payroll\2026`). |
| `FileName` | text | The file name. |
| `Extension` | text | Lowercased, no dot (`xlsx`). |
| `SizeBytes` | bigint | File size in bytes. |
| `LastModified` | timestamptz | When the file was last changed on disk. |
| `AgeDays` | int | Days since `LastModified`, computed here — stale sensitive data is the finding, not the fresh kind. |
| `Classifications` | jsonb | The classification verdict as an array (`["PHI","PII"]`). An empty array means scanned but unreadable or unsupported, i.e. unclassified. |
| `ClassificationList` | text | The same list as flat text (`PHI, PII`), for tools that would rather not parse JSON. Null when unclassified. |
| `IsSensitive` | boolean | True when `Classifications` is non-empty. The sensitive/unclassified split. |
| `SensitiveContent` | jsonb | **Masked** record of what tripped detection. Always masked at write time. |
| `Confidence` | numeric | The classifier's confidence in the verdict. |
| `Risk` | varchar | The agent's own risk label (`HIGH`). Kept verbatim. **Null means not recorded — not a benign verdict.** |
| `Severity` | varchar | The agent's own severity label (`CRITICAL`). Same caveat on null. |
| `SeverityReason` | text | Why, in the agent's words ("Health insurance + Email + Phone"). |
| `DpdpFlag` | varchar | DPDP disposition for the file (`Protect`). |
| `FirstSeenAt` | timestamptz | When discovery first saw this file — drives "newly discovered sensitive files". |
| `LastSeenAt` | timestamptz | The most recent scan that still found it. |
| `IsDeleted` | boolean | The file was gone at the last scan. Kept for trend lines, purged after 90 days. Filter these out for a current-state picture. |

---

## 3. `vw_policy_compliance_daily` — compliance rollup

One row per **UTC day × channel × activity type × policy × severity**. Derived
from `vw_dlp_events`, so the rollup and the rows can never disagree.

> The console shows a tenant-local day, so a single day's figure here can differ
> from the console at the day boundary. Over a week they agree.

| Column | Type | Meaning |
|---|---|---|
| `EventDateUtc` | date | The UTC day being summarised. |
| `Category` | text | Channel: `File`, `Content` or `Web`. |
| `ActivityType` | text | The kind of action within that channel. |
| `PolicyName` | text | The policy that decided; `(no policy)` where none matched. |
| `Severity` | text | Severity band; `(none)` where unset. |
| `TotalEvents` | bigint | All events in the group. |
| `AllowedEvents` | bigint | Allowed through. |
| `WarnedEvents` | bigint | Warned — went through, but the user was told. |
| `BlockedEvents` | bigint | Stopped. |
| `OverriddenEvents` | bigint | Blocked, then lifted by an approved override. |
| `CompliancePercent` | numeric | `AllowedEvents ÷ TotalEvents`, as a percentage to 2 decimals. The same definition the console's Policy Compliance tile uses, web requests included. |
| `InvestigationEvents` | bigint | How many of these are flagged for investigation. |
| `SensitiveEvents` | bigint | How many had at least one sensitive match. |
| `DistinctUsers` | bigint | Distinct people in the group. |
| `DistinctMachines` | bigint | Distinct machines in the group. |

> **Careful with averages:** averaging `CompliancePercent` across rows weights a
> policy with 3 events the same as one with 30,000. For a figure that matches the
> console, re-derive it: `SUM(AllowedEvents) / SUM(TotalEvents)`.

---

## 4. `vw_policy_inventory` — the policies behind the verdicts

Active policies from the current policy engine, so "is this block explained by a
published policy?" is answerable without another data pull. Join to the event
views on `PolicyName`.

| Column | Type | Meaning |
|---|---|---|
| `PolicyId` | bigint | Policy id. |
| `PolicyGuid` | uuid | Stable external identifier. |
| `PolicyName` | text | Display name — the value that appears as `PolicyName` on events. |
| `Description` | text | What the policy is for, as its author wrote it. |
| `Status` | text | `Draft`, `Published` or `Disabled`. |
| `IsPublished` | boolean | Shorthand for `Status = 'Published'` — only published policies can decide anything. |
| `Severity` | text | The severity the policy stamps on its events: `Critical`, `High`, `Medium`, `Low`. |
| `Category` | text | The policy's category grouping. |
| `PolicyScope` | text | Which direction it governs (`UploadAndDownload`, …). |
| `OsCode` | text | Which OS it targets: `Windows`, `Mac`, `Linux`, `Any`. |
| `IsDefault` | boolean | The tenant's fallback policy — applies where nothing more specific does. |
| `HasUpload` | boolean | The policy configures upload rules. A policy with this false can never explain a blocked upload. |
| `HasDownload` | boolean | It configures download rules. |
| `HasWebFiltering` | boolean | It configures web filtering rules. |
| `HasTenantConfig` | boolean | It configures tenant/account restrictions. |
| `HasResponseControls` | boolean | It configures response controls (override and approval behaviour). |
| `PublishedRevisionNo` | int | Which revision is live. |
| `DateAdded` | timestamptz | When the policy was created. |
| `DateUpdated` | timestamptz | When it was last changed. |

---

## Notes for whoever queries this

- **Nulls carry meaning, and it is not zero.** A null `RiskScore` means that
  channel is not scored. A null discovery `Risk` means the agent did not record
  one. Neither is a low score.
- **`Category` before anything else.** Most columns are channel-specific; an
  aggregate across all three without filtering will mix populations. The
  **Channel** column in each table above says where a column is populated.
- **Compliance is allow-only.** Warns are not compliant. If the evaluation wants
  "not blocked" instead, that is `PolicyAction <> 'Block'` — a different number,
  say which one a result used.
- **Filter `IsDeleted` on discovery** for a current-state picture.
- **`statement_timeout` is 120s.** Put a time bound on exploratory queries over
  the event view; `EventTime` and `EventDateUtc` are the indexed way in.
