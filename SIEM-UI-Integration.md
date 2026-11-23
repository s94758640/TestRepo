# SIEM Integration — UI / UX Design Guide

**Audience:** UX / design team (sections 1–9) · UI integration team (sections 10–12)
**Screens:** SIEM landing, Connect-a-SIEM wizard (5 steps), Create API access (3 steps), Delivery health, Field mapping reference, Audit log
**Status:** **design phase — nothing is built.** Branch `feature/siem-integration` is empty. Routes below are the proposed contract, not yet live.
**Backend contacts:** DLP API team · Design reference: the approved SIEM Integration plan (security, data contract, flows)

---

## 1. What this feature is, in one paragraph

A **SIEM** is the customer's central security log system — Splunk, Microsoft Sentinel, IBM QRadar,
Elastic. Every security tool they own reports into it, and their security team works inside it all
day. Today Defensely events live only in our own dashboard, so that team never sees them. This
feature lets a customer admin connect their SIEM in under five minutes, after which DLP events
appear in their tool within a minute.

**Why it matters:** every enterprise security RFP asks *"Does it integrate with our SIEM?"*
Answering "No" loses the deal. This is a sales-blocking feature, not a nice-to-have.

---

## 2. The one decision the admin makes first

There are two directions, and the landing screen must make the choice obvious. This is the single
most important UX decision in the feature.

| | **Connect a SIEM** (we push) | **Create API access** (they pull) |
|---|---|---|
| Who starts the connection | We do — our server sends to them | Their SIEM does — it calls us |
| Speed | ~14 seconds | ~60 seconds (their poll interval) |
| Works with on-prem SIEM behind a firewall | ❌ No | ✅ **Yes** |
| Setup effort for the admin | Paste a URL + token | Paste a client id + secret into their SIEM |
| Auth | Their token, stored encrypted by us | **OAuth 2.0** — they get a client id + secret |
| Ships in | v2 | **v1 — design this first** |

**Design both, but Create API access (pull) ships first.** It is the only option that works for
customers running Splunk or QRadar inside their own data centre, and it is what we can deliver
soonest.

---

## 3. Basics (for the integration team)

- **Base URL:** the DLP API gateway, e.g. `https://qa-email.azure-api.net/qadlp`
- **Dashboard calls** use the normal `Authorization: Bearer <JWT>` + `ClientGUID` header, same as
  every other screen, and the standard envelope:
  ```json
  { "success": true, "message": null, "data": { ... }, "errors": null }
  ```
- **Two endpoints deliberately break that envelope** — `POST /oauth/token` and `GET /siem/events`.
  Those are consumed by the customer's SIEM, not by our UI, and must follow RFC 6749 exactly or
  stock connectors fail to parse them. The dashboard never calls them.
- **Admin-only:** every write below requires the `siem.manage` permission. Non-admins get **403**.
  The UI should show the screen read-only rather than hiding it — admins and analysts both need to
  *see* whether a SIEM is connected.

---

## 4. Endpoint map (what each screen element calls)

| Screen element | Method + route |
|---|---|
| Landing — destinations list | `POST siem/connectors/list` |
| Landing — API clients list | `POST siem/clients/list` |
| Wizard step 1 — vendor cards | `GET siem/providers` |
| Wizard step 3 — filter field list | `GET Notification/TriggerFields/List` *(existing)* |
| Wizard step 4 — sample event preview | `POST siem/preview` |
| Wizard step 5 — Test connection | `POST siem/connectors/test` |
| Wizard step 5 — Save | `POST siem/connectors` |
| Start / pause a destination | `POST siem/connectors/toggle/{id}` |
| Delete a destination | `POST siem/connectors/delete/{id}` |
| Delivery health screen | `POST siem/connectors/{id}/health` |
| Retry now | `POST siem/connectors/{id}/retry` |
| Backfill | `POST siem/connectors/{id}/backfill` |
| Create API client | `POST siem/clients` |
| Rotate secret | `POST siem/clients/{id}/rotate` |
| End rotation overlap early | `POST siem/clients/{id}/rotate/complete` |
| Revoke client | `POST siem/clients/{id}/revoke` |
| Verify setup | `POST siem/clients/{id}/verify` |
| Field mapping reference | `GET siem/field-map?format=json\|ecs\|cef` |
| Audit log | `POST siem/audit/list` |

Everything lives under **Settings → Integrations → SIEM**, alongside the existing Cloud Discovery
integrations. **Reuse that visual language — this is a sibling feature, not a new design system.**

---

## 5. Screen 1 — SIEM landing

### 5a. Empty state

- **Headline:** *Send your DLP events to your SIEM*
- **Sub:** *Stream Defensely events into Splunk, Microsoft Sentinel, QRadar or Elastic so your
  security team sees them alongside everything else they monitor.*
- **Two primary cards, side by side** — the §2 choice:

| Card | Title | Sub | Badges |
|---|---|---|---|
| Left | **Connect a SIEM** | We send events to your SIEM as they happen. | `Real-time` · `Cloud SIEM` |
| Right | **Create API access** | Your SIEM pulls events from us on a schedule. | `Works on-prem` · `OAuth 2.0` |

- Helper link below: *"Which one should I choose?"* → opens a short panel with the §2 table.

### 5b. Populated state — two sections on one page

**Destinations** (push) — one row per connector:
vendor logo · display name · endpoint host · status pill · last delivery · 7-day sparkline ·
`⋯` menu (Test · Edit · Pause · Delete)

**API clients** (pull) — one row per client:
display name · `client_id` (with copy button) · scopes · last used · status pill ·
`⋯` menu (Rotate secret · Revoke · View audit)

### Status pills — design all five

| Pill | Colour | Meaning |
|---|---|---|
| `● Connected` | green | Delivering normally |
| `● Pending` | grey | Saved but never successfully tested |
| `● Failing` | red | 3+ consecutive delivery failures — events are queuing |
| `● Paused` | amber | Admin stopped it; nothing is being sent |
| `● Revoked` | dark grey | Credential killed; cannot be restored |

---

## 6. Screen 2 — Connect a SIEM (push wizard, 5 steps)

### Step 1 — Choose a destination

Card grid. **Available:** Splunk HEC · Microsoft Sentinel · Elastic · Generic Webhook.
**Greyed with a `Coming soon` ribbon:** IBM QRadar · Syslog.

> The vendor list comes from `GET siem/providers`, including the field definitions for step 2.
> **Design the card so a new vendor can be added with no UI release** — the same approach the Cloud
> Discovery wizard already uses.

### Step 2 — Credentials

Fields change per vendor. All four variants need designing:

| Splunk HEC | Microsoft Sentinel | Elastic | Generic Webhook |
|---|---|---|---|
| Display name | Display name | Display name | Display name |
| Collector URL | DCE endpoint URL | Elasticsearch URL | Webhook URL |
| HEC Token 🔒 | Entra Tenant ID | API Key 🔒 | Auth type: Bearer / Basic / Header |
| Index *(optional)* | Client ID | Index or data stream | Token 🔒 |
| Sourcetype *(optional)* | Client Secret 🔒 | | Custom header name *(optional)* |
| | DCR Immutable ID | | |
| | Stream name | | |

Placeholder examples to show in the fields:
- Collector URL — `https://acme.splunkcloud.com:8088/services/collector/event`
- Sourcetype — default `defensely:dlp`
- DCR Immutable ID — `dcr-7f2e8d4a6b104c9e8a1b3d5c7e9f0a2b`

**🔒 = secret field.** Masked input. **Never echo a stored secret back on edit** — show
`Token ending ••••a3f9` with a **Replace** button instead of the value.

Every field needs a `?` tooltip saying **where in their SIEM to find that value**, e.g.
*"Splunk → Settings → Data Inputs → HTTP Event Collector → your token."*
**This copy is the single biggest driver of support tickets — please draft it with the backend team.**

Collapsed by default at the bottom: **Custom CA certificate (PEM)** textarea, for on-prem SIEMs
using a private certificate authority.

### Step 3 — What to send

Radio group:
- `All DLP events` *(default)*
- `Only Critical and High severity`
- `Custom filter` → opens the **existing alert-rule condition builder**, reusing the same 20 fields
  (PolicyAction, Severity, ContentType, Username, FileName, Url, MachineName, PolicyName,
  TransferTypeName, FileType, IsLargeFile, ReceivedTime …). Do not design a second filter UI.

Below it, a separate bordered block — **security-critical, give it visual weight**:

> **Matched sensitive content**
> ◉ **Do not include** — only metadata leaves Defensely *(default, pre-selected)*
> ○ Include a summary of what matched — e.g. `PII: 2, PHI: 1`
>
> ⚠ *Only the category and the number of matches. The matched values themselves never leave Defensely.*

There is deliberately **no third option**, and the second one is a *summary*,
not masked content — no matched value is ever copied, in any form. The database
column physically cannot store any other setting.

For the designer: option 2 is safer than it may read, so it does not need a
heavy warning treatment. The emphasis belongs on option 1 being the default.

### Step 4 — Format

Radio: `JSON (recommended)` · `ECS` · `CEF`.
Beside it, a **live preview pane** showing a real sample event in the chosen format
(`POST siem/preview`). Monospace, scrollable, with a copy button.

### Step 5 — Test & save

**Four states of the Test button — all need designing:**

| State | Appearance |
|---|---|
| Idle | `Test connection` |
| Testing | Spinner · *"Sending a test event to splunk.acme.com…"* |
| Success | Green · *"Test event delivered. Check index `main` in Splunk."* + collapsible **View the event we sent** |
| Failure | Red · the real reason **and the fix** (see §9) |

**Save is disabled until a test succeeds.**

A saved connector starts as **`Paused`**. An explicit **Start sending** toggle turns it on — nobody
should accidentally begin streaming their DLP data to an external system.

---

## 7. Screen 3 — Create API access (pull / OAuth)

### Step 1 — Name it

- *"What will use this?"* → e.g. `Splunk — Production`
- Scopes: `siem:events.read` — pre-checked and disabled in v1
- Optional: IP allowlist (CIDR list), collapsed, with help text that most customers should leave it
  empty because SIEM egress ranges change

### Step 2 — Credentials created ⚠ the most important screen in the feature

| Field | Behaviour |
|---|---|
| `Client ID` | Shown, copy button, always retrievable later |
| `Client Secret` | **Shown once. Never again.** Copy button. |
| `Token URL` | Shown, copy button |
| `Events URL` | Shown, copy button |

The secret sits under an unmissable banner:

> 🔐 **Copy this secret now. We will never show it again.**
> If you lose it, you will have to rotate the credential.

Also on this screen: **Download setup guide** — a ready-made config the admin pastes into their
SIEM (a Splunk REST-input stanza, a Sentinel CCP block, an Elastic `httpjson` config). This is what
turns a 30-minute setup into a 3-minute one.

**Leaving the page requires ticking "I have saved the secret."**

### Step 3 — Verify

A **Verify setup** button that performs a real token request plus one event read, and reports back
in the same four states as §6 step 5.

### Rotate flow (from the `⋯` menu)

Modal copy:
> *Your old secret keeps working for 24 hours, so you can update your SIEM without downtime.*

→ new secret shown once → a persistent banner on the row:
`Old secret expires in 23h 41m` with an **End overlap now** action.

### Revoke flow

Confirm dialog naming the client:
> *Your SIEM will stop receiving events within 60 seconds. This cannot be undone.*

---

## 8. Screens 4–6

### Screen 4 — Delivery health (per destination)

- Four stat tiles: `Events sent (24h)` · `Failed` · `Backlog` · `Last success`
- Time-series chart: sent vs failed, 7 days
- **Failure banner** when `Status = Failing` — the real last error, when it started, how many events
  are waiting, and a **Retry now** button:
  > *Delivery failing since 14:00. 1,284 events waiting.*
  > *Last error: connection timed out to splunk.acme.com:8088*
- Recent deliveries table: time · batch size · status · duration · error
- **Backfill** action → date range picker (max 90 days) → progress bar:
  *"Backfilling… 12,400 / 184,210 events (7%) · ~24 min remaining"*

> Important for design: when a customer's SIEM is down, **we keep queuing — nothing is dropped.**
> The UI must communicate "waiting, not lost." The backlog number is reassurance, not an error.

### Screen 5 — Field mapping reference (read-only)

A static table the customer's SOC engineer reads so they know what to expect. Format switcher
(JSON / ECS / CEF) and a copy-as-CSV button. Low design effort, high support value.

| Defensely | Canonical | ECS | CEF |
|---|---|---|---|
| EnrichedId | `event_id` | `event.id` | `externalId` |
| ReceivedTime | `@timestamp` | `@timestamp` | `rt` *(epoch ms)* |
| Username | `user.name` | `user.name` | `suser` |
| EmployeeCode | `user.id` | `user.id` | `suid` |
| MachineName | `host.name` | `host.hostname` | `shost` |
| SerialNumber | `host.id` | `host.id` | `deviceExternalId` |
| FileName *(name only)* | `file.name` | `file.name` | `fname` |
| FileType | `file.extension` | `file.extension` | `fileType` |
| FileSize | `file.size` | `file.size` | `fsize` |
| AppDeviceName | `destination.app` | `dlp.destination_app` | `destinationServiceName` |
| Url *(host only)* | `destination.domain` | `destination.domain` | `destinationDnsDomain` |
| TransferTypeName | `dlp.transfer_type` | `dlp.transfer_type` | `cs3` |
| ContentType | `dlp.content_type` | `rule.category` | `cs2` |
| PolicyName | `dlp.policy` | `rule.name` | `cs1` |
| PolicyAction | `dlp.action` | `event.action` | `act` |
| RiskScore | `dlp.risk_score` | `event.risk_score` | `cn1` |
| Severity *(word)* | `event.severity` | `dlp.severity` | `cs4` |
| Severity *(number)* | `event.severity_num` | `event.severity` | header slot 7 |

Two details the customer's SOC engineer will ask about:

- **Severity is carried twice.** ECS types `event.severity` as a number, so the
  numeric form goes there and the word lives at `dlp.severity`. Putting
  `"Critical"` in `event.severity` makes Elastic reject the document outright.
  Numbers are `Critical → 9` · `High → 7` · `Medium → 5` · `Low → 3` ·
  `Unknown → 0`, spread so a rule can filter `>= 7` and still work if a level is
  added later.
- **Not `dst`.** In CEF, `dst` is an IP address field — a collector will either
  reject a hostname there or index it as an address. The correct keys for a
  named destination are `destinationServiceName` and `destinationDnsDomain`.

LEEF (for IBM QRadar) is v2 and not listed here yet.

### Screen 6 — Audit log

Table: time · action · who · from where · outcome. Filter by action and date. Export.
**Read-only — no edit, no delete**, and the UI should make that visually obvious. This is the
evidence trail for "who sent our DLP data where."

---

## 9. Error copy — please draft all of these

The design rule throughout: **the message names the setup step to revisit, not just the HTTP code.**

| Situation | Message |
|---|---|
| Not HTTPS | *Only https:// addresses are accepted. Check the collector URL.* |
| Private/internal address | *That address is on an internal network and cannot be reached from Defensely. Use your SIEM's public hostname.* |
| Bad URL | *This doesn't look like a valid address. Example: `https://acme.splunkcloud.com:8088/services/collector/event`* |
| 401 | *The token was rejected. Check it in Splunk → Settings → Data Inputs → HTTP Event Collector.* |
| 403 | *Your SIEM accepted the token but refused the request. Check the token's index permissions.* |
| 404 | *No collector found at that path. Splunk URLs usually end in `/services/collector/event`.* |
| Timeout | *No response from splunk.acme.com within 15 seconds. Check the host is reachable and the port is open.* |
| TLS failure | *The certificate could not be verified. If your SIEM uses a private certificate authority, add its CA certificate in step 2.* |
| Rate limited | *Your SIEM is asking us to slow down. We'll retry automatically.* |
| Permission denied | *You need the SIEM administrator permission to change this. Contact your Defensely admin.* |

---

## 10. Cross-cutting design requirements

1. **Every secret field** uses the same masked-input + **Replace** pattern. A stored secret is never
   rendered, on any screen, to any role.
2. **Every list and detail screen** needs four states: empty · loading skeleton · error ·
   **permission denied** (visible but read-only for non-admins).
3. **Desktop-first.** These are admin screens. Tablet-responsive is enough; phone is not a target.
4. **Reuse the existing Integrations / Cloud Discovery patterns** — grid, wizard chrome, status
   pills, `⋯` menus.

---

## 11. Reference — what one event looks like

This is what lands in the customer's SIEM. Useful for the preview pane and the setup guide.

```json
{
  "schema_version": "1.0",
  "event_id": "558213",
  "event_class": "file.transfer.blocked",
  "@timestamp": "2026-10-15T11:47:02Z",
  "tenant_id": "a311119f-4c2e-4e2a-9f11-7b3d5c8e1a02",
  "vendor": "Defensely",
  "product": "DLP",
  "user":        { "name": "selva.m@acme.com", "id": "EMP-4471" },
  "host":        { "name": "LAP-4471", "id": "PF2X9KQ3", "os": "Windows 11" },
  "file":        { "name": "customers.xlsx", "extension": "xlsx", "size": 2516582 },
  "destination": { "app": "Dropbox", "domain": "dropbox.com" },
  "dlp": {
    "transfer_type": "Upload",
    "content_type": "PII",
    "policy": "PII Block",
    "action": "Block",
    "override_applied": false,
    "risk_score": 87
  },
  "event": { "severity": "Critical", "severity_num": 9 },
  "incident_url": "https://qadlp.defensely.ai/incidents/558213"
}
```

Note what is **not** in it: no matched card numbers, no full file path, no URL query string, no AI
verdict text. That exclusion is deliberate and permanent.

The same event in CEF:

```
CEF:0|Defensely|DLP|1.0|file.transfer.blocked|Sensitive file upload blocked|9|rt=1760528822000
suser=selva.m@acme.com shost=LAP-4471 fname=customers.xlsx fsize=2516582 act=Block dst=Dropbox
cs1Label=PolicyName cs1=PII Block cs2Label=ContentType cs2=PII cn1Label=RiskScore cn1=87
externalId=558213
```

---

## 12. Reference — the end-to-end story (for the storyboard)

```
11:47:02  An employee drags customers.xlsx into Dropbox
11:47:02  Defensely agent scans it → 500 card numbers → PII
11:47:03  Policy "PII Block" → BLOCKED, popup shown to the employee
11:47:09  Event enriched and stored
11:48:00  The customer's Splunk polls us and collects it
11:48:01  Event indexed in Splunk
11:48:05  Splunk correlates: same user, VPN login from a new country at 11:20
          → SOC analyst gets a P1 alert
```

**63 seconds** from the drag to the alert. That correlation on the last line is the entire value of
the feature — Defensely supplies one puzzle piece, and the SIEM already holds the others.
