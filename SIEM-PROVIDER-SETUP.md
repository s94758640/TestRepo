# SIEM provider setup — end to end

**For:** sales engineering, support, and the customer's own SIEM administrator.
**Covers:** every provider we support, what the customer does on their side, and
the exact values to bring back to Defensely.

> Vendor consoles change. Where a menu path here no longer matches, the value
> being collected is still the right one — find it by name rather than by path.

---

## What we support

| | Direction | Transport | Format | Status |
|---|---|---|---|---|
| **Splunk** | Push | HTTP Event Collector | JSON | ✅ |
| **Microsoft Sentinel** | Push | Logs Ingestion API | JSON | ✅ |
| **IBM QRadar** | Push | Syslog over TLS | LEEF | ✅ |
| **Generic HTTPS webhook** | Push | POST, JSON body | JSON | ✅ |
| **Any SIEM at all** | **Pull** | They call us | JSON · ECS · CEF · LEEF | ✅ |

**Four push destinations. Pull works with everything**, including Elastic,
Sumo Logic, Google SecOps and anything that can make an authenticated HTTPS
request on a schedule.

---

## First question: push or pull?

Ask the customer one thing:

> *"Can you open your SIEM's web interface from home, without a VPN?"*

| Answer | Use | Why |
|---|---|---|
| **Yes** | Either — **push** is nicer | Their collector has a public address we can reach |
| **No** | **Pull** | Their SIEM is inside their network. We cannot reach in; it can reach out |

Banks and government customers are almost always the second case. **Pull is not
the fallback — for them it is the only option**, and it is the one that needs no
firewall change on their side.

If unsure, start with pull. It works in both situations.

---

## Pull — works with any SIEM

Nothing vendor-specific. The customer's SIEM authenticates with OAuth 2.0
client credentials and polls for events.

**In Defensely:** Settings → Integrations → SIEM → **Create API access**

Give them four values:

| | |
|---|---|
| Client ID | `siem_7Kp2mQx9vB4nL1wR8tZ3cF` |
| Client secret | **Shown once.** Lost means rotate |
| Token URL | `…/oauth/token` |
| Events URL | `…/siem/events` |

Paste-ready configurations for Splunk, Elastic and Sentinel are in
**`SIEM-API-Integration.md`** — hand that document to their engineer.

**Tell them two things up front**, or they will raise a ticket:

- The feed is **30 seconds behind real time**, deliberately. It is a
  correctness guarantee, not lag.
- Delivery is **at-least-once**. Deduplicate on `event_id`.

---

## Push 1 — Splunk

### What they do in Splunk

1. **Settings → Data Inputs → HTTP Event Collector**
2. **Global Settings** → set *All Tokens* to **Enabled**. Note the HTTP Port
   Number (default **8088**) and whether SSL is enabled — **it must be**
3. **New Token** → name it `Defensely DLP` → Next
4. Choose the destination **index** (or create one, e.g. `dlp`)
5. **Review → Submit**, then copy the **Token Value**

### What to collect

| Field | Example | Where |
|---|---|---|
| Collector URL | `https://acme.splunkcloud.com:8088/services/collector/event` | Their Splunk host + the HEC port |
| HEC Token | `8f3b2c1a-5d9e-4a7b-91c3-7f2e8d4a6b10` | Step 5 |
| Index | `dlp` | Optional — blank uses the token's default |
| Sourcetype | `defensely:dlp` | Optional — this is our default |

> **Splunk Cloud uses a different host and port.** It is
> `https://http-inputs-<stack>.splunkcloud.com:443/services/collector/event` —
> port **443**, not 8088, and the host has an `http-inputs-` prefix. Both ports
> are permitted; the wrong *host* is the usual mistake.

### Common failures

| Error | Fix |
|---|---|
| The HEC token was rejected | Token wrong, or *All Tokens* still Disabled in Global Settings |
| No collector at that path | URL must end `/services/collector/event` |
| Only https:// addresses are accepted | SSL is off in HEC Global Settings — enable it |
| Accepted the request but rejected the data | The index does not exist, or the token has no permission on it |

---

## Push 2 — Microsoft Sentinel

The longest setup of the four — **five Azure objects**. Budget 20 minutes.

### What they do in Azure

1. **Create a Data Collection Endpoint (DCE)** — Azure Monitor → Data
   Collection Endpoints → Create. Same region as their Log Analytics workspace
2. **Create a custom table** in the workspace — Tables → Create → New custom
   log (DCR-based). Name it so it ends `_CL`, e.g. `Defensely_CL`
3. **Create a Data Collection Rule (DCR)** linked to that DCE and table. The
   stream name will be `Custom-Defensely_CL`
4. **Register an app** — Entra ID → App registrations → New. Then
   Certificates & secrets → **New client secret** and copy the *Value*
   immediately (Azure also only shows it once)
5. **Grant the app access** — on the **DCR**, Access control (IAM) → Add role
   assignment → **Monitoring Metrics Publisher** → select the app

> Step 5 is the one that gets missed. The role goes on the **DCR**, not on the
> workspace and not on the subscription.

### What to collect

| Field | Example | Where |
|---|---|---|
| DCE endpoint URL | `https://defensely-dce-ab12.eastus-1.ingest.monitor.azure.com` | DCE → Logs ingestion URI |
| Entra Tenant ID | `72f988bf-…` | App registration → Overview |
| Application (client) ID | `a1b2c3d4-…` | App registration → Overview |
| Client secret | | Step 4 — **the Value, not the Secret ID** |
| DCR Immutable ID | `dcr-7f2e8d4a6b104c9e…` | DCR → **JSON View** → `immutableId` |
| Stream name | `Custom-Defensely_CL` | DCR → the declared stream |

### Common failures

| Error | Fix |
|---|---|
| The client secret is wrong | The **Secret ID** was copied instead of the **Value** — the usual one |
| The client secret has expired | Azure secrets expire. Create a new one |
| No app registration with that client ID | Wrong tenant, or Object ID copied instead of Application ID |
| Lacks permission on the Data Collection Rule | Step 5 missing, or the role was added at the wrong scope |
| The DCR immutable ID or stream name was not found | Immutable ID is only in **JSON View** — not the overview blade |
| Azure rejected the data | The custom table's schema does not match the fields we send. Compare against `GET siem/field-map` |

---

## Push 3 — IBM QRadar

Syslog over TLS. The only provider with no token — **TLS authenticates the
server to us**, not us to it.

### What they do in QRadar

1. **Admin → Log Sources → New Log Source**
2. Log Source Type: **Universal LEEF**
3. Protocol Configuration: **TLS Syslog**
4. Listen port: **6514** (the only syslog port we permit)
5. Set Log Source Identifier to `defensely-dlp`
6. **Deploy Changes**

### What to collect

| Field | Example | Notes |
|---|---|---|
| Host | `qradar.acme.com:6514` | Hostname, **not** an IP address |
| CA certificate (PEM) | `-----BEGIN CERTIFICATE-----…` | **Only** if they use a private CA. The CA certificate, not the server's |

> **A hostname is required.** An IP literal is refused — a genuine collector has
> a DNS name and a certificate for it, and accepting IPs is how an internal
> target gets past a name-based check.

### Common failures

| Error | Fix |
|---|---|
| The TLS certificate could not be verified | Private CA — paste its PEM into the connector |
| Use the SIEM's host name rather than an IP address | Give it a DNS name |
| Port N is not allowed | Must be 6514 |
| Resolves to an internal address | QRadar is not reachable from the internet — **use pull instead** |

### Important: no delivery confirmation

Syslog over TCP confirms the socket accepted the bytes, **not** that QRadar
parsed or stored them. A success here is a weaker claim than a 200 from Splunk.
If events are not appearing, check QRadar's own log source status — our side
will show success either way.

---

## Push 4 — Generic HTTPS webhook

Any endpoint that accepts an HTTPS POST with a JSON body. Use this for a SIEM
we have no dedicated transport for, or for the customer's own pipeline.

| Field | Notes |
|---|---|
| Webhook URL | HTTPS only. Ports 443, 8088, 9243 or 6514 |
| Token | Optional. Sent as `Authorization: Bearer <token>` |
| Custom header name | Optional. Sends the token in that header instead |

**Body:** a JSON array of events. With `format=cef` or `leef` it is `text/plain`,
one event per line.

### Best way to prove the pipeline

Before involving any real SIEM, point a webhook connector at a free capture
service such as **webhook.site** and press Test. The event appears in the
browser immediately.

If that works, everything after it is credentials and payload shape. If it
fails, the problem is on our side — which is a much more useful thing to learn
first.

---

## Testing any provider

In Defensely: **Settings → Integrations → SIEM → the destination → Test**.

- Sends **one synthetic event**, clearly labelled
  (`connection.test@defensely.ai`). Never a real incident
- **Works even when delivery is switched off**, so a destination can be proven
  before anything streams
- A destination **cannot be switched on until a test has passed**

### Rules that apply to every provider

| Rule | Why |
|---|---|
| **HTTPS only** | A token in a header over http is a token on the wire |
| **Public hostname, not an IP** | A real collector has a DNS name and a certificate |
| **Ports 443, 8088, 9243, 6514 only** | Narrow by default |
| **Redirects are not followed** | A 3xx from a collector is an error, not a hop. Configure the final URL |
| **Internal addresses refused** | Including cloud metadata endpoints |

If a customer's collector fails the last one, that is the signal to switch them
to **pull** — not to ask them to open a firewall.

---

## What we send, and what we never send

Every provider receives the same event, rendered in their format. Full field
mapping: `GET siem/field-map`, or the Field Reference screen.

**Never sent, to any provider:**

| | |
|---|---|
| The matched sensitive values | The data the product exists to protect |
| Full file path | Leaks project and client names |
| Full URL | Query strings carry session tokens |
| Classification payload, AI verdict text | Both quote the document |
| Evidence video links | Not a log field |

A destination may optionally receive a **summary** of what matched —
`{"PII":2,"PHI":1}`, categories and counts only. **No matched value leaves in
any form**, and the database cannot store a setting that would allow it.

---

## Cost — raise it before they ask

Splunk and Sentinel **bill by the gigabyte ingested**. Our events add to the
customer's bill.

Point them at the **filter** on the connector: they can send only Critical and
High, or only blocked transfers, instead of everything. For a large tenant that
is a material difference, and a SIEM bill that jumps after connecting Defensely
is a bad first week.

---

## Related documents

| Document | For |
|---|---|
| **`SIEM-API-Integration.md`** | The customer's engineer — pull setup, paste-ready configs |
| **`SIEM-OVERVIEW.md`** | Architecture, security, operations |
| **`SIEM-UI-Integration.md`** | UX and frontend |
