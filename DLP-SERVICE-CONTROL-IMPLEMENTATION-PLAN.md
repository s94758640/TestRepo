# DLP Service Control — backend implementation plan

Against **DLP Service Control: Backend Requirements v1.0, 20 Nov 2026** (EXE team).
Branch `Development/v2-0-6/ServiceControl`, cut from `PROD/ENV-01/21112026-V2-0-5`.

> **The fuller write-up is `DLP-Service-Control-Plan.html`** — it covers the UI as well and is
> the version shared with the EXE team. This file is the working plan.

Admin clicks Stop / Start / Restart in the console; `DLPService` on that endpoint changes
state within 10 seconds, with no polling from the endpoint. The agent (`iseccmd`) holds one
outbound WebSocket open and we push the command down it.

**The spec is accepted as written.** The protocol, the state machine, the business rules and
the thirteen acceptance tests are implementable here as specified, and nothing below asks the
EXE team to change the agent. What follows is what *our* environment adds to it, and the three
things that need an answer before the socket can be built.

---

## 1. Status

| Phase | State |
|---|---|
| **1 — data layer** | **Done.** `V0054__service_command.sql`, `ServiceCommand` entity, EF mapping, wired into `OnModelCreatingPartial` |
| **2 — REST for the console** | **Done.** Three routes + the agent's HTTP fallback, admin-gated, tenant-scoped |
| **3 — WebSocket** | **Done.** Upgrade, auth, registry, replay, ping/pong, ack, result, 90 s idle drop, newest-wins |
| **4 — HTTP fallback** | **Done.** `ReportServiceCommandResult`, ownership-checked |
| **5 — cross-replica delivery** | **Done.** Socket-holder poll — see §2.2, *not* Service Bus |
| 6 — infrastructure | **Outstanding.** APIM WebSocket API per environment (§4), ingress check |
| 7 — acceptance tests | Needs a QA endpoint running the new agent (§7) |

**Verified running:** container validation passes, the dispatcher starts, the socket endpoint
rejects a non-WebSocket GET with 400, and both console routes answer 401 without a token.

**No design question is outstanding.** What remains is the APIM task and the acceptance pass.

---

## 2. Blocking — we are not a single instance

> §9: *"The serial → socket map lives in one instance's memory… A single instance needs
> neither."*

`prod02-dlp-api` is an **Azure Container App** and `pipeline-templates/deploy-container-app.yml`
sets no replica bounds — it only does `az containerapp update --image`. More than one replica
is the normal case, which breaks delivery in a way a single-user test will not show:

```
React  ──POST──▶  replica A   (writes the command, holds no socket)
iseccmd ══ws══▶   replica B   (holds the socket, never hears about it)
```

The command persists as `Pending` and is never pushed. Intermittent, load-dependent.

**We have no Redis** — no `StackExchange.Redis` package, nothing in `appsettings`.

| Option | Cost | Verdict |
|---|---|---|
| **Azure Service Bus fan-out** — every replica subscribes, the socket holder sends | No new dependency: `DLPProcessLaterService` already brokers over Service Bus | **Recommended** |
| Redis pub/sub (the spec's suggestion) | New infrastructure to provision, pay for and run | Works, buys nothing Service Bus doesn't |
| Pin `minReplicas = maxReplicas = 1` | Free today; throws away scaling and HA **for the whole API** | Stop-gap only |

### SignalR was considered and rejected — raw WebSocket it is

SignalR would have given us a backplane for free, and that is genuinely tempting. It is still
the wrong choice here, because **it is not the protocol the agent speaks**:

| | This spec (§4) | SignalR |
|---|---|---|
| Connect | WebSocket upgrade, then straight to messages | HTTP negotiate, then a handshake frame |
| Framing | one JSON object per frame | every message terminated by `` |
| `type` | a **string** naming the message — `"ping"`, `"ServiceCommand"` | an **integer** message-kind, inside an envelope |

An agent sending `{"type":"ping"}` as a bare frame is not a SignalR client. Adopting SignalR
would mean asking the EXE team to rewrite and re-test the endpoint half that §summary says is
**already finished and tested** — to buy a backplane we can get from Service Bus without
touching the agent at all.

**Decided: raw WebSocket exactly as §3 and §4 specify.** Nothing about the agent changes, and
§2 stops being a blocker.

### 2.2 Cross-replica delivery — a poll, not Service Bus

The table above recommended Service Bus. **That was over-engineered and is not what shipped.**
The socket holder polls for its own serials instead:

```sql
every ~2 s:  SELECT * FROM dlp."ServiceCommand"
             WHERE "State" = 'Pending' AND "Serial" = ANY(@serialsThisReplicaHolds)
```

Why it wins over a bus for this feature: 2 s worst-case pickup against a **10 s budget** (the
agent's own sample executes in 2.3 s); **no infrastructure added**; self-healing, since a
missed push simply retries next tick with no delivery guarantees or dead-lettering to reason
about; and it is **the same query replay-on-connect already runs**, so wrong-replica and
offline-reconnect share one code path instead of two.

Service Bus buys sub-second delivery. The budget does not need it. It stays the upgrade path
if measurement ever says otherwise — implemented in `ServiceCommandDispatcher`.

### 2.1 Managed transport — SignalR, Web PubSub, or self-hosted

Worth settling properly, because it is expensive to change after the agent ships against it.
**Neither SignalR nor Web PubSub is referenced anywhere in the solution today; Service Bus
already is — including in `SFM.DLP.API` itself**, not just the worker.

The two Azure services are often spoken of together and are not interchangeable here:

| | Client protocol | Fits this agent? |
|---|---|---|
| **Azure SignalR Service** | SignalR only — negotiate, handshake, `` separators, integer `type` in an envelope | **No.** The agent sends `{"type":"ping"}` as a bare frame |
| **Azure Web PubSub** | Raw WebSocket, arbitrary text frames | **Protocol yes** — but its own hostname and a client access token, so the agent's connect URL and flow change |
| **Self-hosted in the Container App** | Raw WebSocket | **Yes, unchanged** |

So the real comparison is Web PubSub versus self-hosted. SignalR is out on protocol grounds
alone, whatever we have provisioned.

**Recommendation: self-hosted for v1.**

- **No agent change.** §3 fixes the URL as `{BaseUrl}` + `/ws/endpoint-commands` — the same
  base the agent already uses. Web PubSub lives on a different host and expects a token, so
  adopting it means changing the half the EXE team says is finished and tested. We could hide
  it behind APIM's WebSocket passthrough and inject the token there, but that is more moving
  parts than the thing it replaces.
- **No new dependency.** `Azure.Messaging.ServiceBus` is already a package reference in
  `SFM.DLP.API`. The fan-out is a topic and a subscription, not a new service to provision,
  budget and learn.
- **The scale does not call for it.** §9 of the spec: one idle socket per endpoint, about two
  small frames a minute. That is a modest connection count, not a broadcast workload.

**Where Web PubSub would genuinely win — and the trigger to revisit.** Connection metrics,
drop counts and scale-out come free with a managed service; self-hosted means we instrument
it. Two things would change the answer:

1. **Connection count outgrowing a Container App replica.** Measure before the third feature
   moves onto the socket (§9.4).
2. **Deploy churn becoming painful.** Every revision drops every socket at once, and all
   agents reconnect together. The spec handles it — 2–60 s jittered backoff plus replay on
   connect — but a managed service holds the connections outside the app and smooths it.

**Observability without the managed service.** We are not blind: `dlp."ServiceCommand"` already
records the whole lifecycle per command — `CreatedAt`, `DeliveredAt`, `AckedAt`, `CompletedAt`,
`ElapsedMs` — so "how long from click to result", "how many never acked", "which endpoints
never deliver" are all SQL questions, not transport questions. That is feature-level tracking
and it is better than connection counts for answering whether the feature works. Add connection
open/close/drop logging to Log Analytics and the gap to managed is small at this scale.

---

## 3. Socket auth — settled, `ClientGUID` only

§3 and §8 name three headers — `ClientGUID`, `CompanyGUID`, `X-Desktop-Client` — and say to
validate them *"exactly as the existing desktop endpoints do"*. Two of the three do not exist
here: neither `CompanyGUID` nor `X-Desktop-Client` is referenced anywhere in the codebase.

**Decided: we validate what we actually have.** The upgrade handler checks `ClientGUID`
resolves to an active tenant — the same check `TenantBindingMiddleware` already performs for
every desktop call — and that the serial is registered under that client. The other two
headers are accepted and ignored; the agent may keep sending them.

This satisfies §8's real requirement (presence alone is not enough) without inventing a
validation rule for a header our API has never had a meaning for. Not raised with the EXE
team — nothing about it changes the agent.

---

## 4. APIM will not carry the socket

Our rule — every new route must be registered in APIM or QA gets a gateway 404 — applies, but
the usual mechanism **cannot** do it. `pipeline-templates/import-apim-apis.yml` runs:

```
az apim api import --specification-format OpenApiJson …
```

A WebSocket endpoint is not an OpenAPI operation, so it will never appear in that document.
The agent connects through `https://qa-email.azure-api.net/qadlp`, so the socket goes
**through APIM**, which needs a separate WebSocket API:

```
az apim api create --api-type websocket --service-url wss://<fqdn>/ws/endpoint-commands …
```

That is a new pipeline step. Verify it is possible on our APIM tier **before promising QA a
date** — this is the same shape as the `V0037` migration that left every web drill-down
returning 500.

Also confirm the Container App's ingress allows `Upgrade: websocket` and its idle timeout
exceeds the spec's 90 s.

---

## 5. Data model — done

`dlp."ServiceCommand"`, per tenant. §6.1's DDL was SQL Server (`BIT`, `NVARCHAR`,
`DATETIMEOFFSET`); ours is PostgreSQL with a singular table name, guarded inside the `DO $$`
block, idempotent.

**One addition to the spec:** rule 1 ("at most one open command per endpoint") is enforced by
a partial unique index, not only by the service layer —

```sql
CREATE UNIQUE INDEX "UX_ServiceCommand_OneOpenPerSerial"
    ON dlp."ServiceCommand" ("Serial")
    WHERE "State" IN ('Pending','Delivered','Acked');
```

Two admins clicking in the same instant is exactly what the service cannot serialise on its
own, and the failure is otherwise silent: two open rows, both replayed on connect, the agent
running whichever lands last. Now the second insert fails loudly and the caller retries after
superseding.

**`V0054` must run against every tenant database**, not just the template — the tenant a
`ClientGUID` resolves to comes from `sfm."Client"."DlpName"` in Nucleus. Plan that rollout as
part of this feature; `V0037` is the cautionary tale.

### 5.1 Serial as the endpoint identity — checked against QA

Measured on `QA_SFM_DLP`, 68 endpoint rows:

```
distinct serials                      49
serials spanning more than one machine  0
nulls / empties / untrimmed             0      all uppercase
length                                  7 – 57
```

**Serial is a safe identity.** Every value maps to exactly one machine and one `EndpointId`,
so §3's "one socket per serial" and the `UX_ServiceCommand_OneOpenPerSerial` index are both
sound. Compare **case-insensitively and trimmed**, as §3 says.

**But it is frequently not a BIOS serial.** The column holds two shapes:

| Example | What it is |
|---|---|
| `PW0F6X3B`, `F0WJNQ2`, `PF5EDZCA` | Dell service tags — real BIOS serials |
| `5CD2026Q39`, `5CD6360NR6` | HP serials — real BIOS serials |
| `1415-6109-6884-2945-9939-7533-26` | a 32-char dashed fallback, **not** a BIOS serial |
| `0000-0000-6440-8197-7895-1768-42` | the same fallback, partly zeroed |

The dashed values are what gets stored when the BIOS reports nothing usable. **If `iseccmd`
sends a BIOS serial where our row holds the fallback, the upgrade is rejected and the machine
looks permanently offline to the console** — with nothing on screen to explain it. That is the
question to put to the EXE team (§7.3), and one machine from each group is enough to answer it.

**Implementation note:** a serial matches *several* endpoint rows (19 of the 68 are repeat
registrations). The ownership check is "does any row with this serial belong to this client" —
an `EXISTS` / `AnyAsync`, never `Single()`, which would throw on exactly the rows most likely
to be in use.

---

## 6. Remaining phases

**Phase 2 — REST for the console.** Controller → Application service → repository, per our
layering. Admin policy on all three; a tenant admin reaches only their own tenant's endpoints;
serial checked against `Endpoint.SerialNumber`.

| Route | Notes |
|---|---|
| `POST /api/endpoints/{serial}/service-command` | 202 + `Location`; supersedes open commands |
| `GET /api/service-commands/{commandId}` | polled every 1 s until final |
| `GET /api/endpoints/{serial}/push-status` | recommended; cheap |

**Phase 3 — WebSocket.** `UseWebSockets` with a 30 s keep-alive, socket handler (upgrade,
auth, replay on connect, `ping`→`pong`, ack, result, 90 s idle drop, newest-socket-wins), and
a serial→socket registry. The agent sends `ClientGUID`, so `TenantBindingMiddleware` binds the
tenant on the upgrade exactly as it does for existing desktop calls.

**Phase 4 — HTTP fallback.** `POST {BaseUrl}/ReportServiceCommandResult`, PascalCase body,
beside `GetEndPointCommand` on `ExeController`. A result for another serial's `commandId` is a
404.

**Phase 6 — acceptance.** Tests 1–9 and 11 can run against a scripted socket client; 10, 12
and 13 are plain API tests. Reserve a real endpoint for a final pass.

---

## 7. Questions for the EXE team

**Nothing is blocking.** The transport question is answered in §2 and the auth question in §3,
both on our side. What remains is logistics for the acceptance pass, not design:

1. Which `iseccmd` build carries this, and is it deployed to QA endpoints? *(needed for
   acceptance tests 1–9; everything else can be built and unit-tested without it)*
2. QA and prod are separately hosted with different base URLs — confirmed. The socket path is
   appended to each environment's own base, so the APIM WebSocket API (§4) is one task per
   environment.
3. **The sharpened serial question — see §5.1.** Not "is it the BIOS serial" (we checked, and
   it often is not) but: *does `iseccmd` derive the serial the same way the registering agent
   did, including the fallback used when the BIOS reports nothing?*

The six reference implementation files named in §11 were not received with the document. We do
not need them — the spec is complete enough to build from — but if they exist they are worth
having as a cross-check against our socket handler.

## 8. Decided on our side — no answer needed

Socket auth: `ClientGUID` + serial ownership, extra headers ignored (§3) · fan-out mechanism ·
PostgreSQL schema and the per-tenant rollout · APIM WebSocket API and Container Apps ingress ·
admin auth and tenant scoping · the DB-enforced one-open-command rule · never originating a
`ping`, only answering · `commandId` emitted as 32-char lowercase hex, matching the spec's
sample.

## 9. After this: moving the polled endpoints onto the socket

Service control is the first user of this transport, and **it will not be the only one — but
nor will it be many.** The agent surface is 101 routes and a good number are polls, yet the
decision taken is that **four or five move onto the socket at most, ever**. Everything else
keeps polling.

That bound is the design input, not a caveat on it:

- **No message bus, no envelope, no routing layer.** Four or five `type` values on a flat
  protocol is the right size. Build for that and stop.
- **Web PubSub is off the table permanently**, not deferred — §2.1's triggers were sized for a
  socket that kept accumulating features. At this bound they will not fire.
- **§9.5's concentration risk mostly dissolves.** The worry was one connection quietly becoming
  load-bearing for approvals, policy and scans at once. Bounded at five, each keeping a
  lengthened poll underneath, it stays a latency concern rather than an availability one.

### 9.1 Two decisions to take now, both free

**Keep the registry generic.** The serial → socket map and the Service Bus fan-out should
speak "send this JSON frame to this serial", not "send this ServiceCommand". Service control
becomes one caller of a transport rather than the transport itself. Costs nothing today;
retrofitting it later means touching the socket handler again.

**The `type` discriminator is already the extension point.** §4's protocol is a string-named
`type` per frame, and both sides ignore unknown types. So a new message type is additive and
backward compatible: an old agent ignores a frame it does not know, a new agent works against
an old backend. **Nothing in the protocol needs changing to add the next feature** — which is
why this section is a roadmap and not a redesign.

### 9.2 The rule that makes migration safe: push the notification, not the payload

The temptation is to push data down the socket. Don't. Push a **change notification** and let
the agent fetch over the HTTP endpoint it already uses:

```
server → agent   {"type":"PolicyChanged","endpointUserId":4821}
agent  → server  GET V2/PolicySnapshot/4821        (the existing route, unchanged)
```

Why this shape:

- **The frame cap is 64 KB** (§3). A policy snapshot or an app catalogue will exceed it. A
  notification never will.
- **The HTTP route stays the source of truth**, so cold start, reinstall and socket-down all
  keep working with no second code path.
- **Migration is reversible.** If push misbehaves, lengthen the poll back and nothing else
  changes.
- **No ordering, retry, chunking or partial-delivery problem** on the socket. The agent's
  fetch is already idempotent.

**Never remove a poll — lengthen it.** Every migrated endpoint keeps its poll as a slow safety
net (say 15 minutes instead of 30 seconds). Push becomes the fast path, polling the floor. An
endpoint that only works over the socket makes every socket outage a correctness bug rather
than a latency one.

### 9.3 Candidates, best first

| Polled today | Why it is a candidate | Gain |
|---|---|---|
| `V1/CheckOverrideStatus`, `CheckPromptOverrideStatus`, `CheckMetaStatus` | **A user is blocked at their screen** waiting for an admin decision while the agent polls | The strongest case in the list — seconds instead of a poll interval, on a user-visible wait |
| `V2/PolicyCheck/{id}` + `V2/PolicySnapshot/{id}` | `PolicyCheck` already is "has it changed?" — exactly the question a push answers | A published policy reaches endpoints at once; the cheap poll disappears |
| `V1/GetEndPointCommand` | The existing pull command channel, which this feature deliberately did not fold into | Would unify the two channels — **do this last**, once the socket has proven itself |
| `V1/GetDiscoveryScanJob` | Agent waits for a queued scan | Scans start on click |
| `V1/GetEndpointExeSettings`, `GetAgentProxyExclusions` | Config refresh on a timer | Settings apply immediately |
| `V1/GetAlertMessagesByEndpointUser` | Messages to display to the user | Shown when sent |

Not candidates: everything the agent **writes** — `InsertLandLog`, `UpdateEndpointStatus`,
`Submit*`, `Upsert*` — and the `Download*` routes. Those are already request/response with no
waiting, and a socket adds nothing.

### 9.4 Sequencing

1. **Ship service control.** Prove the socket in production: connection stability, reconnect,
   the Service Bus fan-out, APIM behaviour under real load.
2. **Then one migration** — override status, the highest-value and lowest-risk, since its
   payload is a single status value and its poll already exists as the fallback.
3. **Measure before the third.** Sockets per replica, memory, reconnect storms after a deploy.
   The capacity note in §9 of the spec assumes one idle socket per endpoint; that assumption
   should be checked against real numbers before several features depend on it.
4. **`GetEndPointCommand` last**, if at all. It is the one with an existing contract and real
   consequences, and it is the least painful poll of the set.

### 9.5 What would make this a bad idea

Worth writing down so it is a decision and not a drift:

- **One socket carrying everything is a single point of failure.** Today a flaky socket costs
  service control. After four migrations it costs policy delivery, approvals and scans at
  once. The lengthened polls (§9.2) are what keep that from being an outage.
- **Reconnect storms.** A backend deploy drops every socket simultaneously; the agent's 2–60 s
  jittered backoff handles that for one feature, but the fetch-on-notify pattern means every
  endpoint may also hit an HTTP route at once on reconnect. Stagger the replay.
- **It is not free on the agent either.** Every migration is EXE-team work too. The roadmap
  above is only worth it where the poll interval is a user-visible wait — which is why the
  override endpoints lead and config refresh trails.
