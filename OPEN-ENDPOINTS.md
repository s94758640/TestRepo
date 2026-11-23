# Endpoints without authentication

Counted from the code on 2026-09-08, not from the approval document. The approval
document records 80 unauthenticated endpoints in total. That number is wrong in
both directions and has caused repeated confusion, so this file records what is
actually there and why each group is open.

**126 of 338 endpoints require no authentication.** ExeApi is excluded from both
figures — its 41 endpoints do require a credential, just a weak one.

    SFM.DLP.API        217 total    63 open
    SFM.Identity.API    66 total    14 open
    SFM.Nucleus.API     55 total    49 open

Counted by scanning controller attributes: a method is open if it carries
`[AllowAnonymous]`, or if it has no `[Authorize]` and its controller does not
derive from `BaseController` (which carries `[Authorize]`). That is a regex over
source, not compiled routing metadata, so treat the totals as accurate to within
one or two rather than exact. The classification below is what matters.

Nucleus.API deserves its own line: it has **no `[Authorize]` anywhere and no
fallback policy**. `app.UseAuthorization()` runs, but nothing requires a user. Five
of its endpoints are closed, and those are the three added by this work plus two
others — the rest of the surface is open by default rather than by decision.

---

## Waiting on the EXE agent — about 59

    DLP  TrayApp          14
    DLP  Exe              13
    DLP  Feedback          2
    DLP  LogFileProcess   11
    DLP  Endpoint          7
    DLP  Log               7
    DLP  Policy            4
    DLP  ClearCache        1
    DLP  Storage           2
    Identity  Endpoint     2

The agent calls these and has no token to present. Closing them means taking the
fleet offline, so they wait for the credential scheme in `AGENT-SECURITY-PLAN.md`
— phases 1 to 6, blocked on two confirmations from the EXE team.

Most already carry a TEMPORARY comment naming the ExeApi replacement. `Policy` and
most of `Exe`, `LogFileProcess` and `Endpoint` do not yet.

Two in this group are worse than the rest and are worth naming:

`GetEndpointExeSettings` returns the Azure storage account key and the UBX S3
secret. That key grants read, write and delete across every container,
permanently, and it reaches every laptop. No authentication work touches this — the
agent is the legitimate caller. Phase 7 removes the key from the response rather
than protecting the endpoint, which is why it is the highest-value item on the
plan and independent of the EXE team.

`Identity/InsertEndpointDetails` is self-enrolment: post a serial number, receive
working credentials. Phase 2 closes it with a single-use token in the installer.

`Storage/upload-url` was opened on 2026-09-11 and is the newest member of this
group. It mints a write-capable URL into the tenant's storage account, which
sounds worse than it is: the URL names one object, permits `PUT` only, expires in
15 minutes, and cannot read or delete. The caller chooses neither the container
nor the path — both are derived server-side from the purpose and the endpoint id
— so there is no field in which to express "write over that other device's
evidence", and `endpointUserId` is validated against the tenant before anything
is signed.

`GET storage/shadow-keys` (opened 2026-10-19) deliberately shares upload-url's
posture: the shadow-copy proxy calls it at start-up and daily and has no token to
present, so the `ClientGUID` header is the whole gate. What it hands out is the
tenant's shadow-copy KEKs — wrapping keys for new shadow files, not credentials
into anyone's account — and it is the one route in the API behind a rate limit
(20 requests per 10-minute window per caller, no queue), because a legitimate
proxy needs the list a few times a day and anything past that is a scripted pull
of key material. Every fetch writes an action-log row naming tenant, caller and
key count; key bytes are never logged. Closes with the same credential scheme as
upload-url. The admin-side routes next to it (`POST storage/shadow-keys/rotate`,
`GET incidents/{metaId}/shadow-file`) are NOT open — they require the console
JWT.

`POST feedback/submit` and `GET feedback/lookups` are the newest members of this
group. The tray app is where employees raise a false positive, and it ships no
token, so requiring one here would simply refuse every employee. Both are marked
`[AllowAnonymous]` explicitly rather than inheriting openness.

What an anonymous caller can do is create rows in `dlp."Feedback"` and add
attachments — up to five files of 10 MB, extension-checked and magic-byte
checked, stored under a server-derived name they cannot influence. Nuisance and
storage cost, not disclosure: the route reads nothing back except the row it just
wrote, and the user id it records is asserted rather than proven, so a forged
submission can misattribute itself but cannot read anyone else's feedback. The
admin-side routes beside it — the grid, the detail, triage and export — are NOT
open and require the console JWT.

This one is better placed than the rest of the group, because its replacement
already exists: `POST V1/SubmitFeedback` on the Functions host runs the same
service behind `BasicAuthMiddleware`, which identifies tenant and device. The
route closes when the tray moves over, not when the credential scheme lands.

What an anonymous caller *can* do, holding a valid `ClientGUID`, is add objects
indefinitely: storage cost and noise in the evidence containers, not disclosure
and not tampering. That is the same trade the rest of this group makes, and it
closes the same way — the agent credential scheme in `AGENT-SECURITY-PLAN.md`.
Note it is strictly better than the endpoint it replaces for the agent:
`GetEndpointExeSettings` hands the agent a permanent account key with read, write
and delete across every container, which is exactly what Phase 7 exists to stop.

---

## Not to be touched — 27

    Nucleus  DigitalCard  11
    Nucleus  Meeting      10
    Nucleus  Contact       4
    Nucleus  WhatsAppWebhook  2

Excluded by instruction. WhatsAppWebhook is also a genuine external webhook, which
cannot present a credential the way an internal caller can; it should be verified
by signature rather than left to the general auth scheme, but that is out of scope
here.

---

## Public by decision — 15

    Nucleus  KnowledgeBase    12
    DLP      Master            2
    Nucleus  GetClientGuidByEmail  1

`GetAllMaster` and `GetAllCountryCodes` are read during trial signup, before any
account exists. `GetClientGuidByEmail` is the pre-login tenant lookup — it does
also confirm whether an email is known to the system, which is user enumeration,
but it cannot be closed without breaking login.

KnowledgeBase is a decision rather than a necessity. The website has no login, so
all twelve stay open and volume is capped at APIM. Eight of them are `Admin/*`
routes — `POST`/`PUT` article and category, `DELETE Admin/Article/{id}`, `DELETE
Admin/Attachment/{id}`. Rate limiting caps how often, not who: anyone can still
delete any article, just not thousands per minute. Accepted on the basis that the
content is recoverable. If that stops being true, these eight are the first thing
to revisit.

One correction to that, found by exporting the gateway configuration on
2026-09-08: **the rate limiting this relies on was not configured.** Every APIM
API — `qadlp`, `qanucleus`, `qaidentity`, `qauser`, `web` — had an inbound policy
of `base` and, on two of them, CORS. No `rate-limit`, no `rate-limit-by-key`, no
`quota` anywhere, at any scope. So the control the KnowledgeBase decision depends
on did not exist while the decision was being made. The team is adding it by APIM
policy.

Worth carrying forward: an APIM policy protects the hosted deployment only.
On-premises installations have no gateway, so anything relying on APIM for a
security control is unprotected there. `AGENT-SECURITY-PLAN.md` already makes this
point about `/AgentToken`, which needs limiting per access key rather than per IP —
a thousand agents behind one corporate NAT share an address, so an IP limit either
blocks legitimate traffic or is set high enough to stop nothing.

The same export answered two other questions. No policy reads `X-Api-Key`, which
is what made it safe to delete from the worker. And every backend is a bare host
with no path rewriting, so the `/qanucleus` and `/qadlp` suffixes are stripped
before the request reaches the service — meaning the signed path matches what the
service sees, and signing will validate when `Mode` becomes `Enforce`.

---

## Login and account recovery — 8

    Identity  Authentication  8

Login, forgot password, reset password, change password. Each was checked
individually. `ChangePassword` looks like it should be closed but requires and
validates `OldPassword`, so it is legitimately anonymous.

`UpdateAllUserPassword` was in this group and is now `[Authorize]`. Anonymous, it
reset every user's password to a value compiled into the source. It was the most
serious finding in this codebase.

---

## Internal service-to-service — 7

    Identity  User      4    sync-job endpoints
    DLP       UserSync  2
    Nucleus   Client    1    GetAllClient

These have no user token because no user is involved. `[Authorize]` is the wrong
tool; signing is the right one, and the mechanism exists and is proven.

`GetAllClient` is already in the Nucleus allow-list. The other six are not — the
worker signs the three log-file endpoints, but `Identity/User`'s four sync-job
endpoints have no signing on either side yet.

None of this rejects anything today. Both hosts run `SignedRequests.Mode=Audit`,
which validates, logs the outcome, and passes the request through regardless.
Signing is deployed and watching, not enforcing. See `DEPLOY-SECRETS.md`.

---

## Closed by this work

| endpoint | was | why it could close |
|---|---|---|
| `Identity/UpdateAllUserPassword` | anonymous | reset every password to a hardcoded value |
| `Nucleus/GetClientUserMap` | anonymous | no caller anywhere |
| `Nucleus/GetClientUserMapsByClientId/{clientId}` | anonymous | no caller, and took the tenant id from the URL unchecked |
| `DLP/EnableClearCacheForAllClients` | anonymous | ops trigger, no caller, reached every tenant in one call |
| 55 + 8 more | anonymous | console and non-wizard trial routes, closed in earlier commits |

`Nucleus/UpsertClientUserMap` was deliberately left open. Identity.API calls it
server-to-server during signup with no user token, so `[Authorize]` would break
trial account creation silently — the call site ignores the result, so the user
would be created, the email-to-tenant mapping would not be written, and that user
would simply fail to log in with nothing pointing at Nucleus. It closes by signing
instead.

---

## What would still be true after everything here ships

About 58 endpoints stay open until the agent has a credential, and one of them
hands out a storage account key that grants delete across every container. That is
the remaining exposure, and neither authentication work nor rate limiting touches
it — only Phase 7 does.
