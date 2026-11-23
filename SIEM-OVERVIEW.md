# SIEM Integration — complete reference

**One document covering:** what a SIEM is · why we integrate · what exists today ·
how a customer connects · the API · the data we send · security decisions ·
what is not built yet.

**Branch:** `feature/siem-integration` · **Commit:** `99c0cfb` and later
**Audience:** anyone picking this up — engineering, UX, sales engineering, support.

---

# Part 1 — What this is

## 1.1 What a SIEM is

A **SIEM** is a search engine for a company's logs. The well-known one is
**Splunk**; the others are **Microsoft Sentinel**, **IBM QRadar** and
**Elastic Security**.

A bank runs twenty security systems and each writes its own logs in its own
place. Checking all twenty takes hours. So the bank buys a SIEM, every system
pipes its logs in, and there is one place to search.

```
Firewall     ─┐
Antivirus    ─┤
VPN          ─┼──►   their SIEM   ──►   one search box
Windows      ─┤      (Splunk)           their security team, all day
Defensely ←us─┘
```

Each contributing system is called a **log source**. This feature makes
Defensely one of them.

## 1.2 Why we built it

**Commercial.** Every enterprise security RFP asks *"Does your product
integrate with our SIEM?"* Answering **No** loses the deal regardless of how
good the detection is. This is the main driver.

**Technical — correlation.** Our event alone says:

> *Selva uploaded customers.xlsx to Dropbox at 11:47 PM. Blocked.*

Their SIEM holds the other facts about the same person at the same time:

| Time | Source | Fact |
|---|---|---|
| 11:20 PM | VPN | login from a new country |
| 11:31 PM | EDR | USB device inserted |
| 11:40 PM | SharePoint | downloaded 4 GB |
| **11:47 PM** | **Defensely** | **sensitive upload blocked** |
| last week | HR | resigned |

Individually noise. Together: *an employee exfiltrating data before leaving*.
Only the SIEM can build that story, because only it holds all five sources.

## 1.3 The two directions

Both end with our events in their SIEM. Only the direction of the call changes.

```
PULL   their SIEM ──asks──► us        built, v1
PUSH   us ──sends──► their SIEM       configuration + test built; delivery worker not yet
```

| | Pull | Push |
|---|---|---|
| Who calls | Their SIEM | Us |
| Latency | ~60s (their poll interval) | ~14s |
| Works behind their firewall | **Yes** | No |
| Admin pastes | Nothing — *we* give *them* an id + secret | Their URL + token |
| Our SSRF exposure | None | Real — see §6.3 |

**Pull was built first** because a Splunk or QRadar installed inside a
customer's own building has no address we can reach. Banks and government
customers are almost always in that situation.

---

# Part 2 — What exists today

## 2.1 Built and tested

| Area | Detail |
|---|---|
| **Credentials** | `sfm."SiemApiClient"` in Nucleus. Hashed, rotation overlap, revoke |
| **Token endpoint** | `POST /oauth/token` — OAuth 2.0 client credentials, RFC 6749 |
| **Event feed** | `GET /siem/events` — keyset cursor, 30s lag window |
| **Formats** | JSON (canonical), ECS (Elastic), CEF (ArcSight/Sentinel), LEEF (QRadar) |
| **Admin API** | create · list · verify · rotate · complete rotation · revoke |
| **Audit trail** | `siem."SiemAuditLog"` — append-only, enforced by trigger |
| **Rate limits** | Chained per-`client_id`: 10/min token, 60/min feed, 2 concurrent |
| **SSRF guard** | `OutboundUrlGuard` — DNS-rebinding safe |
| **Tenant binding** | `TenantBindingMiddleware` — token decides the tenant, not a header |
| **Push destinations** | Connector table, provider catalog, create/test/toggle/delete/health |
| **Push transports** | Splunk HEC · Microsoft Sentinel (Logs Ingestion API) · IBM QRadar (syslog over TLS) · generic HTTPS webhook |
| **Push delivery** | Enqueue → claim → send → retry → dead-letter, with `SiemDeliveryWorker` |

**1820 tests pass.** Everything above is unit-tested with fakes.

### Push delivery, in detail

```
SiemDeliveryWorker  every 15s, per tenant
   │
   ├─ EnqueueAsync   new events (past the 30s lag) → one queue row per enabled connector
   │                 unique (connector, event) makes this idempotent across instances
   │
   └─ DrainAsync     claim with FOR UPDATE SKIP LOCKED → render → send
                     success  → Sent, connector Connected
                     retry    → 30s · 2m · 5m · 15m · 30m · 1h · 2h · 3h  (6.9h total)
                     3 fails  → connector Failing (red pill, backlog banner)
                     exhausted or non-retryable → DeadLettered
```

**Off by default** — `Siem:DeliveryEnabled` must be set. Push makes outbound
requests to customer-supplied addresses, so it does not start merely because
the code shipped.

### Filtering

A connector's optional `FilterJson` is evaluated at enqueue time by
`NotificationRuleEvaluator` — the same evaluator the alert rules use, over the
same 20 trigger fields, so an admin builds a SIEM filter with the existing
condition builder rather than learning a second language.

Three behaviours worth knowing:

- **No filter means send everything.** The evaluator returns `false` for an
  empty condition, because for an alert rule "no condition" means "never
  fire" — here it means the opposite, so an absent filter never reaches it.
  Inverting this would silently stop every unfiltered destination.
- **An unreadable filter queues nothing and marks the connector `Failing`**
  with an actionable message. Fail closed, because sending more than the
  customer asked for costs them money and may exceed what they agreed to — but
  loudly, because a silent stop is worse than either.
- **The cursor advances past filtered-out events**, not just queued ones.
  Otherwise a filter matching nothing would re-scan the same window forever
  and never move.

**Multi-instance safe without leader election**, which differs from the
original plan. Two database guarantees replace it: the unique constraint on
`(SiemConnectorId, EnrichedId)` makes enqueue idempotent, and
`FOR UPDATE SKIP LOCKED` means two instances never claim the same row. That is
why it runs in the API rather than `DLPProcessLaterService` — and why it is
*not* the same answer as `LandingLogMonitorService`, which has neither
protection and does send every alert twice when two pods run.

## 2.2 Not built

| | |
|---|---|
| **Backfill** | "Send me the last 30 days" — the only remaining feature gap |

## 2.3 Not verified

**Nothing has run against a real database.** All tests use fakes. The
migrations are SQL no Postgres has executed, and the EF entity↔column mappings
have never round-tripped. Migrations auto-run on deploy, so broken SQL fails
loudly — but a mapping mismatch only appears on the first query.

## 2.4 Open blockers

| | Issue | Impact |
|---|---|---|
| **B1** | `Storage:CredentialEncryptionKey` is committed and in git history | "Encrypted at rest" is not a claim we can make on a questionnaire. Must rotate to Key Vault **before** any customer push credential is stored |
| **B2** | `TenantBinding__Mode` is still `Audit` | The cross-tenant header hole is open. Flip to `Enforce` once the `TenantBinding:` warnings are clean |
| **B3** | No `UseForwardedHeaders` | `ActorIp` records the APIM gateway for every caller; the per-credential IP allowlist cannot work |

B1 and B2 are pre-existing problems that SIEM made exploitable from outside the
company — they are not SIEM bugs.

---

# Part 3 — How a customer connects

## 3.1 Pull (built)

| Step | Who | What |
|---|---|---|
| 1 | Admin | Defensely → Settings → Integrations → SIEM → **Create API access** |
| 2 | Admin | Names it, e.g. `Splunk — Production` |
| 3 | **Us** | Generate `client_id` + secret; secret shown **once** |
| 4 | Admin | Copies the secret, downloads the config snippet |
| 5 | Admin | Pastes it into their SIEM as a REST/HTTP input |
| 6 | Their SIEM | `POST /oauth/token` → bearer token |
| 7 | Their SIEM | `GET /siem/events` every 60s, paging by cursor |

Target: under five minutes. Full guide with paste-ready Splunk, Elastic and
Sentinel configs: **`SIEM-API-Integration.md`**.

## 3.2 Push (config only)

The admin picks a vendor, pastes a URL and token, chooses a format and filter,
clicks **Test connection**, and saves. The connector saves **paused** — an
explicit toggle starts it, so nobody begins streaming by clicking Save.

Today the test works and the configuration persists. **Nothing is delivered**
until the worker exists.

---

# Part 4 — The API

## 4.1 Consumed by the customer's SIEM

These two deliberately **do not** use the house `ApiResponse<T>` envelope.
Splunk, Sentinel and Elastic parse RFC 6749 strictly and look for
`access_token` at the root; wrapped, they fail with "no token in response".

### `POST /oauth/token`

```http
Content-Type: application/x-www-form-urlencoded
grant_type=client_credentials&client_id=siem_7Kp2…&client_secret=…
```
```json
{"access_token":"eyJ…","token_type":"Bearer","expires_in":3600,"scope":"siem:events.read"}
```

Credentials accepted in the body (`client_secret_post`) **or** an HTTP Basic
header (`client_secret_basic`) — Elastic sends the first, Sentinel the second.

Errors: `invalid_client` (401) · `invalid_request` · `invalid_scope` ·
`unsupported_grant_type` (400).

### `GET /siem/events`

```
?from=&cursor=&limit=&format=json|ecs|cef
```
```json
{"events":[…],"next_cursor":"558213","has_more":false,"schema_version":"1.0"}
```

`limit` default 100, clamped 1–1000 — `0` is **not** "all". `from` capped at a
7-day window. CEF returns `text/plain` with paging in `X-Next-Cursor` /
`X-Has-More`.

## 4.2 Consumed by our dashboard

These **do** use the house envelope.

| Route | Purpose |
|---|---|
| `POST siem/clients/list` | Credential grid |
| `POST siem/clients` | Create — response carries the only copy of the secret |
| `POST siem/clients/{id}/verify` | Issues a token and reads one page, as their SIEM will |
| `POST siem/clients/{id}/rotate` | New secret; old works 24h |
| `POST siem/clients/{id}/rotate/complete` | End the overlap early |
| `POST siem/clients/{id}/revoke` | Immediate; both secrets die |
| `GET siem/field-map` | Field reference screen |
| `POST siem/audit/list` | Audit history |
| `GET siem/providers` | Vendor catalog for the connect wizard |
| `POST siem/connectors/list` | Destinations grid |
| `POST siem/connectors` | Create — saves **paused** |
| `POST siem/connectors/{id}/test` | Sends one synthetic event |
| `POST siem/connectors/{id}/toggle?enabled=` | Start or pause sending |
| `POST siem/connectors/{id}/delete` | Remove |
| `POST siem/connectors/{id}/health` | Sent / failed / backlog / dead-lettered |

All require an exact admin role match — see §6.5.

Two deliberate behaviours in the connector flow:

- **A connector saves paused**, and cannot be switched on until it has been
  tested. Enabling an untested destination means queuing events at a URL
  nobody has proven and finding out through a backlog.
- **Test returns `200` with `success: false`** when the collector rejects it —
  the same decision recorded on `cloud/connectors/test`. A wrong token is a
  result the wizard renders, not an exception.

---

# Part 5 — The data we send

## 5.1 One event

```json
{
  "schema_version": "1.0",
  "event_id": "558213",
  "event_class": "file.transfer.blocked",
  "@timestamp": "2026-10-15T11:47:02.000Z",
  "tenant_id": "a311119f-4c2e-4e2a-9f11-7b3d5c8e1a02",
  "vendor": "Defensely", "product": "DLP",
  "user":        { "name": "selva.m@acme.com", "id": "EMP-4471" },
  "host":        { "name": "LAP-4471", "id": "PF2X9KQ3", "os": "Windows 11" },
  "file":        { "name": "customers.xlsx", "extension": "xlsx", "size": 2516582 },
  "destination": { "app": "Dropbox", "domain": "dropbox.com" },
  "dlp": { "transfer_type": "Upload", "content_type": "PII", "policy": "PII Block",
           "action": "Block", "override_applied": false, "risk_score": 87 },
  "event": { "severity": "Critical", "severity_num": 9 },
  "incident_url": "https://qadlp.defensely.ai/incidents/558213"
}
```

## 5.2 Event classes

`file.transfer.blocked` · `.warned` · `.allowed` · `.uncovered` (no policy
covered it — a coverage gap worth alerting on) · `file.transfer` (unclassified).

Prefix-matchable as `file.transfer.*`. These strings are published and will not
be renamed.

## 5.3 Severity

`Critical 9` · `High 7` · `Medium 5` · `Low 3` · `Unknown 0`

Spread, not consecutive, so `>= 7` keeps meaning "High and above" if a level is
inserted later. **`Unknown` is deliberately not folded into `Low`** — reporting
an unknown severity as low asserts a judgement we did not make.

## 5.4 What is never sent

| Omitted | Why |
|---|---|
| **Matched sensitive text** | The actual card numbers. This is the data the product exists to protect |
| Full file path | A directory tree leaks project names, client names, and usually the person's name again |
| Full URL | Query strings routinely carry session tokens |
| Classification payload, AI verdict | Both quote the document |
| Evidence video link | Not a log field |

An **explicit allow-list** in `SiemEventMapper`, never reflection over the
entity — a column added to that 70-field table next year must not auto-publish
to every customer. Tests assert the exclusions hold across all three formats.

### The `masked` option is a summary, not masked content

A push destination may be set to `SensitivePolicy = masked`. That emits
**categories and counts only**:

```json
"dlp": { "sensitive_content": "{\"PII\":2,\"PHI\":1}" }
```

**It is not a masked copy of the matched values.** The obvious implementation —
passing the stored column through `SensitiveValueMasker` at egress — was
written, tested, and rejected: that masker rewrites only the JSON shapes it
recognises and **returns anything else unchanged**. A stored value in an
unexpected shape would have been published raw, and the tenant most likely to
have one is the tenant who switched write-masking off, whose column holds live
card numbers. A masking function that silently no-ops on unknown input is not a
control.

Emitting structure is safe by construction: no value is ever copied, only key
names and match counts. An analyst gets what they correlate on — *"two PII
matches"* — and the values stay where they are. Anything unparseable emits
nothing rather than a fallback that might contain data.

**There is no setting that emits matched content in any form.** The database
column is constrained to `off`/`masked`, so even a bug in the API cannot
persist anything else.

## 5.5 Field mapping

Served live from `GET siem/field-map`, generated from `SiemFieldMap` so it
cannot drift from what the formatters emit.

| Defensely | Canonical | ECS | CEF |
|---|---|---|---|
| EnrichedId | `event_id` | `event.id` | `externalId` |
| ReceivedTime | `@timestamp` | `@timestamp` | `rt` (epoch ms) |
| Username | `user.name` | `user.name` | `suser` |
| MachineName | `host.name` | `host.hostname` | `shost` |
| File name | `file.name` | `file.name` | `fname` |
| FileSize | `file.size` | `file.size` | `fsize` |
| AppDeviceName | `destination.app` | `dlp.destination_app` | `destinationServiceName` |
| URL host | `destination.domain` | `destination.domain` | `destinationDnsDomain` |
| ContentType | `dlp.content_type` | `rule.category` | `cs2` |
| PolicyName | `dlp.policy` | `rule.name` | `cs1` |
| PolicyAction | `dlp.action` | `event.action` | `act` |
| RiskScore | `dlp.risk_score` | `event.risk_score` | `cn1` |
| Severity (word) | `event.severity` | `dlp.severity` | `cs4` |
| Severity (number) | `event.severity_num` | `event.severity` | header field 7 |

Two traps:

- **Severity appears twice.** ECS types `event.severity` as a *number*; putting
  `"Critical"` there makes Elastic reject the document.
- **Not `dst`.** In CEF `dst` is an *IP address* field — a hostname there is
  rejected or mis-indexed.

---

# Part 6 — Security decisions

## 6.1 Encrypt or hash?

> **Will we ever need to send this secret to someone else?**

| Answer | Use | Example |
|---|---|---|
| Yes — we replay it outbound | **Encrypt** (`SecretProtector`) | Splunk HEC token: it goes in an `Authorization` header on every send |
| No — we only check what is presented | **Hash** (`SiemClientSecret`) | Our own `client_secret`: a leak then yields nothing spendable |

These look inconsistent side by side and are not — applying either everywhere
breaks something. Also in `DEPLOY-SECRETS.md`.

HMAC-SHA256, not PBKDF2: these are 32 bytes of CSPRNG, so there is no
dictionary to run, and a slow KDF would add ~100ms to every token request on an
endpoint a fleet of collectors polls on a schedule.

## 6.2 Token isolation

SIEM tokens use a **separate signing key, issuer and audience** and a second
JWT scheme, `SiemBearer`. Signed with the dashboard's `JwtSecurityKey`, a
read-only feed token would be accepted by every `[Authorize]` controller — and
several authorise on a role claim alone, so a feed credential handed to an
external system would become a partial admin credential.

## 6.3 SSRF

Push posts to an address an admin typed in. `OutboundUrlGuard` enforces:

- **https only.** No flag, no on-prem exception
- No credentials in the URL, no IP-literal hosts
- Port allow-list: 443, 8088, 9243, 6514
- Blocked ranges including `169.254.169.254` (cloud metadata), all RFC1918,
  CGNAT, loopback, link-local, ULA, multicast
- **IPv4-mapped IPv6 unwrapped first** — `::ffff:169.254.169.254` is the
  standard bypass
- **Resolve, check, then dial the checked address.** Validating the URL and
  letting `HttpClient` resolve independently does not stop DNS rebinding
- `AllowAutoRedirect = false`, `UseProxy = false`, TLS 1.2+, 64 KB response cap

There must be **no** code path that returns `true` from a certificate
validation callback. A test asserts this.

## 6.4 Tenant binding

`TenantBindingMiddleware` makes the **token** decide the tenant. Previously the
`ClientGUID` header outranked the JWT claim, so any caller could read another
tenant's stream by changing one header.

Machine tokens bind in **both** modes — a SIEM token carries `client_guid`, not
`ClientGUID`, so without this the feed would fall through to the default
connection in Audit mode.

## 6.5 Authorisation

Admin endpoints use **exact** role matching from configuration, not the
`roleName.Contains("Admin")` check used elsewhere — that passes for
`"Non-Admin"` and `"Admin Assistant"`.

## 6.6 Audit

`siem."SiemAuditLog"`, append-only **enforced by trigger**, not `REVOKE`: the
table owner keeps those rights implicitly, so a trigger is the only form that
makes the guarantee real. Secrets, hashes, salts and hints are never written to
it — an audit log is read by more people than the credential list is.

---

# Part 7 — Operations

## 7.1 Configuration

| Setting | Notes |
|---|---|
| `Siem__TokenSigningKey` | **Secret.** ≥32 bytes. **Must not equal `JwtSecurityKey`** |
| `Siem__SecretPepper` | **Secret.** Changing it invalidates every credential |
| `Siem:PublicBaseUrl` | What the customer's SIEM calls — the APIM gateway |
| `Siem:IncidentBaseUrl` | Dashboard origin for deep links |
| `Siem:AdminRoleNames` | Exact matches |
| `Siem:DeliveryEnabled` | **Default false.** Push does not run until set |
| `Siem:DeliveryIntervalSeconds` | Default 15, floor 5 |
| `TenantBinding:Mode` | `Audit` → `Enforce` |

Both secrets blank is the **safe** state: no credential can be issued and no
token verified. That is intended, not an outage.

Generate with — the Windows PowerShell 5.1 form, since that is what the dev
machines run. The shorter static `GetBytes(32)` overload is .NET 5+ only and
fails on 5.1:

```powershell
$b = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
[Convert]::ToBase64String($b)
```

## 7.2 Migrations

| Script | Creates |
|---|---|
| `Nucleus/V0002__siem_api_client.sql` | Pull credentials (global DB) |
| `Dlp/V0017__siem_audit_log.sql` | `siem` schema + audit table |
| `Dlp/V0018__siem_connector.sql` | Push connector, cursor, queue, attempts |

Auto-run on deploy. **Originally numbered V0007/V0008** — the QA merge brought
V0007–V0016 onto the branch, which collided. Caught by
`Every_dlp_migration_has_its_own_version_number`.

## 7.3 Deployment checklist

1. Set both `Siem__` secrets from Key Vault
2. Deploy — migrations run automatically
3. **Register routes in APIM**: `/oauth/token`, `/siem/events`, `/siem/clients`,
   `/siem/field-map`, `/siem/audit/list`. Without this the UI and customers
   cannot reach them
4. Confirm a credential can be created and verified
5. Watch `TenantBinding:` warnings before flipping to `Enforce`

## 7.4 Post-deploy smoke test

Run these in order. Each one proves a different thing, so the **first failure
tells you what is wrong** — which is faster than testing end to end and
guessing.

| # | Proves |
|---|---|
| 1 | Routes exist and APIM is registered |
| 2 | Nucleus `V0002` ran **and** `Siem__SecretPepper` is set |
| 3 | `Siem__TokenSigningKey` is set and valid |
| 4 | Tenant binding works and the DLP-side mapping is correct |
| 5 | Dlp `V0017` ran |

```powershell
$base   = "https://qa-email.azure-api.net/qadlp"
$jwt    = "<a dashboard JWT for an admin role>"
$tenant = "<that admin's ClientGUID>"
$h = @{ Authorization = "Bearer $jwt"; ClientGUID = $tenant }

# 1 — routes live. 404 here means APIM was not updated, not that code is missing.
Invoke-RestMethod "$base/siem/providers" -Headers $h | % data | ft key, isImplemented

# 2 — create a credential. The secret appears once, in this response only.
$new = Invoke-RestMethod "$base/siem/clients" -Method Post -Headers $h `
       -ContentType application/json -Body '{"displayName":"Deploy smoke test"}'
$id     = $new.data.client.clientId
$secret = $new.data.clientSecret
"client_id = $id"

# 3 — exchange it for a token. Form-encoded, exactly as a SIEM would.
$tok = Invoke-RestMethod "$base/oauth/token" -Method Post -Body @{
    grant_type    = 'client_credentials'
    client_id     = $id
    client_secret = $secret
}
"expires_in = $($tok.expires_in)"

# 4 — read the feed with that token. No ClientGUID header: the token carries it.
$feed = Invoke-RestMethod "$base/siem/events?limit=5" `
        -Headers @{ Authorization = "Bearer $($tok.access_token)" }
"events = $($feed.events.Count), next_cursor = $($feed.next_cursor)"

# 5 — the audit trail recorded step 2.
(Invoke-RestMethod "$base/siem/audit/list" -Method Post -Headers $h `
 -ContentType application/json -Body '{}').data.data | ft occurredAtUtc, action, actorUserName

# 6 — clean up. Leaving a live credential from a smoke test is how one ends up in production.
Invoke-RestMethod "$base/siem/clients/$id/revoke" -Method Post -Headers $h | % message
```

### Reading the failures

| Step | Symptom | Cause |
|---|---|---|
| 1 | `404` | Routes not registered in APIM |
| 1 | `403` | The JWT's role is not in `Siem:AdminRoleNames` |
| 2 | `"SIEM access is not configured"` | `Siem__SecretPepper` is unset |
| 2 | `500`, relation does not exist | Nucleus `V0002` did not run |
| 3 | `invalid_request` | `Siem__TokenSigningKey` unset or shorter than 32 bytes |
| 3 | `invalid_client` | The secret was not copied correctly from step 2 |
| 4 | `403` | The token carries no tenant — check the `ClientGUID` claim on the JWT used in step 2 |
| 4 | `500` | EF mapping mismatch against the real schema — **the most likely genuine bug**, since this path has never run against a database |
| 5 | `500` | Dlp `V0017` did not run |

### Zero events is a pass

`events = 0` at step 4 means the tenant is quiet, or everything is inside the
30-second lag window. It does **not** mean the feed is broken. Confirm with a
wide window:

```powershell
Invoke-RestMethod "$base/siem/events?from=$((Get-Date).AddDays(-7).ToString('yyyy-MM-ddTHH:mm:ssZ'))&limit=5" `
  -Headers @{ Authorization = "Bearer $($tok.access_token)" }
```

### Push

Push stays off unless `Siem:DeliveryEnabled` is set. To test it, create a
connector, call `POST siem/connectors/{id}/test`, and only then toggle it on —
the API refuses to enable a destination that has never been tested.

---

# Part 8 — Troubleshooting

| Symptom | Cause |
|---|---|
| `401 invalid_client` always | Wrong secret, or revoked. Check status; rotate if lost |
| `401` after an hour | Token expired and is not being refreshed |
| `429` | Polling faster than 1/min, a token per poll, or concurrent pollers |
| Events stop, no error | Cursor is past the newest event — normal in quiet periods. Keep the cursor; resetting to zero replays the window |
| Newest events missing ~30s | Expected. See §9.1 |
| Duplicates | Expected — at-least-once. Deduplicate on `event_id` |
| `400 unsupported_format` | `json`, `ecs` or `cef` only |
| Nothing arrives at all | Confirm the credential's tenant is the one with activity |

---

# Part 9 — Two things that look like bugs

## 9.1 The feed lags 30 seconds

`EnrichedId` is an identity column: ids are **allocated** in order but
**commit** out of order. Serving the newest rows immediately means a poller
that reads to id 500 and saves that cursor can permanently miss id 497 — with
nothing to indicate anything went wrong.

The honest fix is a monotonic outbox sequence written by the crunch step. This
is the cheap version, and 30 seconds is invisible to SIEM ingestion.

**Do not work around it by also filtering on time.**

## 9.2 Delivery is at-least-once

The same event can arrive twice — after a network timeout, or a replayed
cursor. Deduplicate on `event_id`, which is stable and unique within a tenant.

---

# Part 10 — Where the code lives

| Path | Contents |
|---|---|
| `SFM.DLP.SiemIntegration/` | Event contract, mapper, formatters, transports, provider catalog |
| `SFM.DLP.API/Services/Siem/` | Token, feed query, admin, audit, rate limits, options |
| `SFM.DLP.API/Controllers/Siem*.cs` | OAuth, events, admin, console |
| `SFM.DLP.API/Middleware/` | Tenant binding |
| `SFM.Common.Utilities/Security/` | `SiemClientSecret`, `OutboundUrlGuard` |
| `SFM.Nucleus.*/…/SiemApiClient*` | Pull credential entity and repository |
| `SFM.Platform.Storage.Tests/Siem*.cs` | ~220 tests |

## Related documents

| Document | For |
|---|---|
| **`SIEM-API-Integration.md`** | The customer's SOC engineer — setup, paste-ready configs |
| **`SIEM-UI-Integration.md`** | UX and frontend — screens, fields, states, error copy |
| **`DEPLOY-SECRETS.md`** | DevOps — env vars, encrypt-vs-hash rule |
| **`DATABASE-CHANGES.md`** | Migration conventions |

---

# Part 11 — What to do next

1. **Run against a real database.** Nothing has. This is where problems will be,
   and it is the single highest-value step remaining
2. **B1** — rotate the encryption key to Key Vault. **Required before any
   customer push token is stored**, because that token is encrypted with it
3. **B2** — flip tenant binding to `Enforce` once the Audit logs are clean
4. **B3** — forwarded headers, which unblocks `ActorIp` and the IP allowlist
5. Backfill — the last feature gap

### A note on QRadar

The only transport that is not HTTP. Three consequences worth knowing:

- **Framing is ours.** Messages use RFC 5425 octet counting — each prefixed
  with its byte length — not newline termination. Newline framing breaks the
  moment a value contains one, and silently: the collector reads half an event
  and treats the remainder as the next. The count is of **UTF-8 bytes**, which
  differs from character count for any non-ASCII filename, and a wrong count
  desynchronises every subsequent event rather than just the one.
- **LEEF is tab-delimited**, and LEEF has no escape sequence for its own
  delimiter. A tab in a filename would split one attribute into two and QRadar
  would index the fragments without complaint, so tabs are replaced.
- **There is no delivery confirmation.** Syslog over TCP tells us the socket
  accepted the bytes, not that QRadar parsed or stored them. Success here is a
  weaker claim than a 200 from Splunk, and the health screen should not imply
  otherwise.

A custom CA **pins** to the customer's root (`CustomRootTrust`) rather than
disabling validation. There is no code path that accepts any certificate.

### A note on Sentinel

The transport uses the **Logs Ingestion API** (Data Collection Endpoint + Data
Collection Rule), not the legacy HTTP Data Collection API. The legacy one is
retiring, and its shared workspace key is a credential we would have to store
and replay on every send. The newer path uses the customer's own app
registration, so the credential is theirs and scoped by them.

Entra token responses are cached per `tenantId|clientId` and refreshed two
minutes early — without that, every batch would hit
`login.microsoftonline.com`, which rate-limits per app registration.

### Before the first push customer

Push is switched off by default, and should stay off until B1 is resolved. The
pull path has no such dependency: its credentials are hashed, not encrypted, so
the committed key does not affect them.
