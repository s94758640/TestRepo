# Connect your SIEM to Defensely

**Audience:** the customer's SOC or platform engineer · **You will need:** an admin account in Defensely and permission to add a data input to your SIEM
**Status:** pull access (your SIEM collects from us) is implemented. Push (we send to you) is not built yet.
**Time:** about 5 minutes.

---

## 1. How it works

Your SIEM calls us. You give it a client id and secret; it exchanges those for a
token and then polls for new events.

```
Your SIEM  ──POST /oauth/token──────────►  Defensely     (once an hour)
Your SIEM  ──GET  /siem/events──────────►  Defensely     (every minute)
```

Nothing needs to be reachable from the internet on your side. This works for a
Splunk or QRadar installed inside your own network, which is why it is the
option we built first.

**Base URL:** `https://qa-email.azure-api.net/qadlp`
Your Defensely admin sees the exact URLs for your environment on the screen
where the credential is created.

---

## 2. Get a credential

In Defensely: **Settings → Integrations → SIEM → Create API access.**

Name it after whatever will use it — `Splunk — Production` — and save. You will
be shown:

| | |
|---|---|
| **Client ID** | `siem_7Kp2mQx9vB4nL1wR8tZ3cF` — not secret, safe to log |
| **Client Secret** | shown **once** |
| **Token URL** | `…/oauth/token` |
| **Events URL** | `…/siem/events` |

> ⚠ **Copy the secret before leaving that page.** There is no endpoint that can
> return it afterwards — not for you, not for Defensely support. If it is lost,
> rotate the credential and update your SIEM with the new one.

---

## 3. Authenticate

Standard OAuth 2.0 client credentials (RFC 6749 §4.4). Credentials go in the
form body or in an HTTP Basic header — both work, use whichever your connector
produces.

```http
POST /oauth/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=siem_7Kp2…&client_secret=…
```

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs…",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "siem:events.read"
}
```

Cache the token for its full `expires_in` and refresh on expiry or on a `401`.
Requesting a new token for every poll will hit the rate limit.

**Errors** are RFC 6749 shaped — `{"error":"invalid_client"}` with `401`, or
`400` for everything else.

| Error | What to check |
|---|---|
| `invalid_client` (401) | The client id or secret is wrong, or the credential was revoked. |
| `invalid_request` (400) | A required form field is missing. |
| `unsupported_grant_type` (400) | `grant_type` must be exactly `client_credentials`. |
| `invalid_scope` (400) | You asked for a scope this credential does not hold. Omit `scope` to get what it has. |

---

## 4. Collect events

```http
GET /siem/events?limit=100&format=json HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIs…
```

```json
{
  "events": [ { "event_id": "558213", "…": "…" } ],
  "next_cursor": "558213",
  "has_more": false,
  "schema_version": "1.0"
}
```

**Save `next_cursor` and send it back on the next call.** That is the whole
paging contract — there are no page numbers.

```
GET /siem/events?cursor=558213&limit=100
```

| Parameter | Default | Notes |
|---|---|---|
| `cursor` | — | `next_cursor` from your previous response. Omit on the first call. |
| `from` | — | ISO-8601 UTC. Required only on a first call if you want history. Max 7 days per request. |
| `limit` | 100 | Clamped to 1–1000. `0` is **not** "all". |
| `format` | `json` | `json`, `ecs`, `cef`, or `leef`. |

When `has_more` is `true`, call again immediately with the new cursor instead
of waiting for your next poll.

### Two things that will otherwise look like bugs

**The feed is 30 seconds behind real time, deliberately.** Event ids are
allocated in order but their database transactions commit out of order. Serving
the newest rows immediately means a poller that reads up to id 500 can
permanently miss id 497 — with nothing to indicate anything went wrong. The lag
gives those transactions time to land. **Do not** work around it by also
filtering on time.

**Delivery is at-least-once.** The same event can arrive twice — after a
network timeout, or if your collector replays a cursor. Deduplicate on
`event_id`, which is stable and unique within your tenant.

---

## 5. Rate limits

Per credential, not per IP — so several collectors behind one corporate NAT do
not share a budget.

| | Limit |
|---|---|
| `POST /oauth/token` | 10 / minute |
| `GET /siem/events` | 60 / minute |
| `GET /siem/events` | 2 concurrent requests |

Exceeding one returns `429` with `Retry-After: 60`. The concurrency cap is the
one worth designing around: run a single sequential poller, not a fan-out.

---

## 6. Paste-ready configuration

### Splunk — REST API Modular Input

```ini
[rest://defensely_dlp]
endpoint = https://qa-email.azure-api.net/qadlp/siem/events
http_method = GET
auth_type = oauth2
oauth2_token_url = https://qa-email.azure-api.net/qadlp/oauth/token
oauth2_client_id = siem_7Kp2mQx9vB4nL1wR8tZ3cF
oauth2_client_secret = <your secret>
oauth2_grant_type = client_credentials
response_type = json
json_response_handler = defensely_cursor
index = main
sourcetype = defensely:dlp
polling_interval = 60
```

Splunk needs a small response handler to carry `next_cursor` forward and to
split the `events` array into individual events.

### Elastic — Filebeat `httpjson`

```yaml
filebeat.inputs:
  - type: httpjson
    interval: 1m
    auth.oauth2:
      client.id: siem_7Kp2mQx9vB4nL1wR8tZ3cF
      client.secret: "<your secret>"
      token_url: https://qa-email.azure-api.net/qadlp/oauth/token
    request.url: https://qa-email.azure-api.net/qadlp/siem/events
    request.method: GET
    request.transforms:
      - set:
          target: url.params.format
          value: ecs
      - set:
          target: url.params.limit
          value: "500"
      - set:
          target: url.params.cursor
          value: '[[.cursor.next]]'
          default: ""
    response.split:
      target: body.events
    cursor:
      next:
        value: '[[.last_response.body.next_cursor]]'
```

Use `format=ecs` for Elastic — the field names then match the schema your
existing dashboards and detection rules already use.

### Microsoft Sentinel — Codeless Connector

```json
{
  "auth": {
    "type": "OAuth2",
    "grantType": "client_credentials",
    "tokenEndpoint": "https://qa-email.azure-api.net/qadlp/oauth/token",
    "ClientId": "siem_7Kp2mQx9vB4nL1wR8tZ3cF",
    "ClientSecret": "<your secret>"
  },
  "request": {
    "apiEndpoint": "https://qa-email.azure-api.net/qadlp/siem/events",
    "httpMethod": "GET",
    "queryParameters": { "limit": 500 },
    "rateLimitQPS": 1,
    "queryWindowInMin": 1
  },
  "paging": {
    "pagingType": "NextPageToken",
    "nextPageParaName": "cursor",
    "nextPageTokenJsonPath": "$.next_cursor"
  },
  "response": { "eventsJsonPaths": [ "$.events" ] }
}
```

### Anything else — curl

```bash
TOKEN=$(curl -s -X POST https://qa-email.azure-api.net/qadlp/oauth/token \
  -d grant_type=client_credentials \
  -d client_id=siem_7Kp2mQx9vB4nL1wR8tZ3cF \
  -d client_secret="$SECRET" | jq -r .access_token)

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://qa-email.azure-api.net/qadlp/siem/events?limit=100" | jq .
```

---

## 7. What an event contains

```json
{
  "schema_version": "1.0",
  "event_id": "558213",
  "event_class": "file.transfer.blocked",
  "@timestamp": "2026-10-15T11:47:02.000Z",
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

### Event classes

`file.transfer.blocked` · `file.transfer.warned` · `file.transfer.allowed` ·
`file.transfer.uncovered` (no policy covered it — a coverage gap worth alerting
on) · `file.transfer` (outcome not classified).

Match a whole family with the `file.transfer.*` prefix. These strings are
published and will not be renamed; new ones may be added.

### Severity

| Word | Number |
|---|---|
| `Critical` | 9 |
| `High` | 7 |
| `Medium` | 5 |
| `Low` | 3 |
| `Unknown` | 0 |

Spread rather than consecutive so `>= 7` keeps meaning "High and above" if a
level is added later. `Unknown` means the value was missing or unrecognised —
it is deliberately not folded into `Low`, because reporting an unknown severity
as low asserts a judgement we did not make.

### What is deliberately **not** in the payload

| Omitted | Why |
|---|---|
| The matched sensitive text | The actual card numbers or identifiers. This is the data the product exists to protect; copying it into a third system over the internet defeats that. |
| Full file path | A directory tree leaks project names, client names, and usually the person's name a second time. You get the file name. |
| Full URL | Query strings routinely carry session tokens and personal data. You get the host. |
| Classification payload, AI verdict text | Both quote the document contents. |
| Evidence video link | A link to a screen recording does not belong in a log line. |

This is an explicit allow-list in our code with a test asserting those fields
stay out, not a filter someone has to remember to apply.

---

## 8. Field mapping

| Defensely | Canonical JSON | ECS | CEF |
|---|---|---|---|
| EnrichedId | `event_id` | `event.id` | `externalId` |
| ReceivedTime | `@timestamp` | `@timestamp` | `rt` *(epoch ms)* |
| Username | `user.name` | `user.name` | `suser` |
| EmployeeCode | `user.id` | `user.id` | `suid` |
| MachineName | `host.name` | `host.hostname` | `shost` |
| SerialNumber | `host.id` | `host.id` | `deviceExternalId` |
| File name | `file.name` | `file.name` | `fname` |
| FileType | `file.extension` | `file.extension` | `fileType` |
| FileSize | `file.size` | `file.size` | `fsize` |
| AppDeviceName | `destination.app` | `dlp.destination_app` | `destinationServiceName` |
| URL host | `destination.domain` | `destination.domain` | `destinationDnsDomain` |
| TransferTypeName | `dlp.transfer_type` | `dlp.transfer_type` | `cs3` |
| ContentType | `dlp.content_type` | `rule.category` | `cs2` |
| PolicyName | `dlp.policy` | `rule.name` | `cs1` |
| PolicyAction | `dlp.action` | `event.action` | `act` |
| RiskScore | `dlp.risk_score` | `event.risk_score` | `cn1` |
| Severity word | `event.severity` | `dlp.severity` | `cs4` |
| Severity number | `event.severity_num` | `event.severity` | header field 7 |

Two notes for whoever writes the parser:

- **Severity appears twice.** ECS types `event.severity` as a number, so the
  numeric form goes there and the word lives at `dlp.severity`.
- **CEF uses `destinationDnsDomain`, not `dst`.** In CEF `dst` is an IP address
  field; a hostname there is either rejected or indexed as an address.

### CEF output

`format=cef` returns `text/plain`, one event per line, with paging in headers
(`X-Next-Cursor`, `X-Has-More`) rather than a JSON envelope.

```
CEF:0|Defensely|DLP|1.0|file.transfer.blocked|Sensitive data transfer blocked|9|rt=1760528822000
externalId=558213 suser=selva.m@acme.com shost=LAP-4471 fname=customers.xlsx fileType=xlsx
fsize=2516582 destinationServiceName=Dropbox destinationDnsDomain=dropbox.com act=Block cat=PII
cs1Label=PolicyName cs1=PII Block cs2Label=ContentType cs2=PII cs3Label=TransferType cs3=Upload
cs4Label=Severity cs4=Critical cs5Label=TenantId cs5=a311119f-… cs6Label=OverrideApplied cs6=false
cn1Label=RiskScore cn1=87
```

---

## 9. Rotating and revoking

**Rotate** when a secret may have been exposed, or on your own schedule. The
old secret keeps working for **24 hours**, so you can update your SIEM without
an outage. End the overlap early from the Defensely UI once you have switched.

**Revoke** is immediate and permanent. Both the current and any
rotating-out secret stop working, and tokens already issued stop being accepted
within **60 seconds**. Create a new credential to reconnect — a revoked one
cannot be rotated back to life.

---

## 10. Troubleshooting

| Symptom | Cause |
|---|---|
| `401 invalid_client` on every request | Wrong secret, or the credential was revoked. Check the status in Defensely; rotate if the secret was lost. |
| `401` after working for an hour | The token expired and is not being refreshed. Refresh on `expires_in` or on a `401`. |
| `429` | Polling faster than once a minute, requesting a token per poll, or running concurrent pollers. |
| Events stop but no error | Your cursor is past the newest event. Normal during quiet periods — `events` is empty and `next_cursor` is `null`. Keep the cursor you have; do not reset it to zero or you will replay the window. |
| Newest events missing for ~30s | Expected. See §4. |
| Duplicate events | Expected at least once. Deduplicate on `event_id`. |
| `400 unsupported_format` | `format` must be `json`, `ecs` or `cef`. LEEF is not implemented. |
| Nothing arrives at all, no errors | Confirm the credential's tenant is the one generating activity, and that `from` is not set to a window with no events. |

---

## 11. Push delivery

Everything above describes **pull** — your SIEM collects from us. We can also
**push**, where we send to your collector directly:

| Your SIEM | How |
|---|---|
| Splunk | HTTP Event Collector |
| Microsoft Sentinel | Logs Ingestion API (DCE + DCR) |
| IBM QRadar | Syslog over TLS, LEEF format, port 6514 |
| Anything else | HTTPS webhook, JSON body |

Push is faster (~14s vs ~60s) but requires your collector to be reachable from
the internet. If it sits inside your network, use pull — that is what it is for.

Your Defensely admin configures it under **Settings → Integrations → SIEM →
Connect a SIEM**.

## 12. Not yet available

- **Backfill** — "send me the last 30 days" on a newly connected push
  destination. Use pull with a `from` date in the meantime.
- **Scopes beyond `siem:events.read`** — read-only is the whole surface today.
