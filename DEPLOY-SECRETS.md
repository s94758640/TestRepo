# Deployment secrets

Every credential each app needs at runtime, and where it comes from.

Written 2026-09-08 by scanning the tracked `appsettings.json` files. Two things
follow from that scan and neither is comfortable:

**Real credentials are committed to this repository.** Six Azure storage account
keys, the Postgres passwords, an S3 secret key, Twilio and WhatsApp tokens, and a
Microsoft Graph client secret are all sitting in tracked `appsettings.json` and
`appsettings.Development.json` files. They are also in git history, so deleting
them from the working tree does not remove them — anyone who has ever cloned this
repo still has them.

**Blanking the files is not the fix.** Rotation is. Blanking stops the exposure
growing; it does nothing about what has already leaked. The rotation list is at
the bottom.

---

## Storing a secret: encrypt or hash?

Before adding a credential to this list, answer one question.

> **Will we ever need to send this secret to someone else?**

| Answer | Use | Why |
|---|---|---|
| **Yes** — we replay it outbound | **Encrypt** — `SecretProtector` / `IStorageSecretProtector` | We need the plaintext back. A cloud connector's client secret and a Splunk HEC token go into an `Authorization` header on every call; hashed, the feature cannot work at all. |
| **No** — we only check what someone presents | **Hash** — `SiemClientSecret` | We never need the value, so we should not be able to read it. A database leak then yields nothing spendable. HMAC-SHA256 with a per-row salt and a pepper held outside the database. |

These two look inconsistent side by side and are not: they are the only correct
answers to two different questions. Applying either one everywhere breaks
something — hash the Splunk token and delivery stops; encrypt our own
`client_secret` and a leaked backup hands out working credentials.

Two rules that follow:

- **A hashed secret is shown exactly once**, at creation, and no endpoint
  returns it afterwards. Lost means rotate, not recover.
- **An encrypted secret is only as strong as its key.** `Storage:CredentialEncryptionKey`
  is currently committed and in git history (see the rotation list below), so
  "encrypted at rest" is not a claim we can make on a security questionnaire
  until it moves to Key Vault.

---

## Before blanking anything

An app whose value is blank and whose environment variable is missing fails at
startup. That is the right failure — loud, immediate, obviously about
configuration — but it is still an outage until someone notices which of 56
settings is absent.

So the order is: set the environment variables, confirm the app starts, *then*
blank the file. Not the other way round.

`appsettings.Development.json` is the exception. It only loads when
`ASPNETCORE_ENVIRONMENT=Development`, so if the containers do not run in
Development mode those files are dead weight in QA and production — and they hold
real QA credentials. Confirm that first; if it holds, they can be blanked with no
deployment risk at all.

---

## Required per app

Names are in double-underscore form, ready to paste as container environment
variables.

### DLPProcessLaterService (5)

    AiApiOptions__Password
    BlobOptions__ConnectionString
    ConnectionStrings__NucleusConnection
    QueueOptions__ConnectionString
    ServiceBusOptions__ConnectionString

### SFM.DLP.API (20)

    AiApiOptions__Password
    AmazonS3Settings__SecretKey
    AzureBlobStorage__ConnectionString
    AzureStorageQueue__ConnectionString
    AzureStorageQueue__LargeFileConnectionString
    ConnectionStrings__DefaultConnection
    ConnectionStrings__NucleusConnection
    EffortlyVideoStorageAccountKey
    EmailPassword
    JwtSecurityKey
    MSIInstallerAccountKey
    MSIInstallerBlobConnectionString
    MSIUninstallerAccountKey
    MSIUninstallerBlobConnectionString
    Siem__SecretPepper
    Siem__TokenSigningKey
    Storage__CredentialEncryptionKey
    Twilio__AuthToken
    VideoStorageAccountKey
    WhatsApp__AccessToken

The two `Siem__` values are new and have never been committed, so they are not
on the rotation list — generate them once per environment and keep them there.

- **`Siem__TokenSigningKey`** signs the access tokens a customer's SIEM
  presents. **It must not equal `JwtSecurityKey`.** Sharing it would make a
  read-only SIEM token acceptable to every `[Authorize]` controller in the app,
  and several of those authorise on a role claim alone — a feed credential
  handed to an external system would become a partial admin credential. At
  least 32 bytes, for HS256.
- **`Siem__SecretPepper`** is mixed into every stored credential hash and is the
  reason a Nucleus backup on its own verifies nothing. Changing it invalidates
  every issued credential at once, so treat it as permanent per environment.

Both blank is the safe state, not an outage: no credential can be issued and no
token verified, and the rest of the API is unaffected.

Generate either with — note this is the Windows PowerShell 5.1 form, which is
what the dev machines here run. The shorter
`[RandomNumberGenerator]::GetBytes(32)` static overload is .NET 5+ only and
fails on 5.1 with "does not contain a method named 'GetBytes'":

    $b = New-Object byte[] 32
    [System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
    [Convert]::ToBase64String($b)

### SFM.DLP.ExeApi (2)

    AzureStorage__ConnectionString
    ConnectionStrings__DefaultConnection

ExeApi is a Function App, so these are Application settings, not file values. Its
`appsettings.json` is never loaded — `Program.cs` uses a bare `new HostBuilder()`
with no `AddJsonFile`, and the file is not copied to the build output. It held a
real `qaaidlpapi` account key that nothing read; blanked, with a note in the file
explaining why editing it changes nothing. The key still needs rotating: unused at
runtime is not the same as unexposed, and it is in git history like the rest.

Locally the same values come from `local.settings.json` under `Values`, which the
Functions host loads as environment variables.

### SFM.Identity.API (9)

    ConnectionStrings__ClientA
    ConnectionStrings__ClientB
    ConnectionStrings__DefaultConnection
    ConnectionStrings__NucleusConnection
    EmailPassword
    JwtSecurityKey
    Razorpay__KeyId
    Razorpay__KeySecret
    Razorpay__WebhookSecret

The payment gateway is **Razorpay**. `Razorpay__KeyId` / `Razorpay__KeySecret`
are the API key pair from Dashboard → Account & Settings → API Keys: test
(`rzp_test_…`) in QA, live (`rzp_live_…`) in production. The Key ID is not
secret on its own, but it is set alongside the secret so the pair never drifts.
`Razorpay__WebhookSecret` is the secret you type when creating the webhook
(Dashboard → Settings → Webhooks); it signs every delivery, so it differs per
environment and is rotated together with the webhook, not on its own. None of
them is in `appsettings.json`; the keys are present there but blank so the
shape is discoverable.

Set `Razorpay__ExpectLiveMode=true` in production. Startup logs a warning when
the key's mode (live/test) and this setting disagree.

Two non-secret settings gate the rollout and are worth knowing about:
`Licensing__EnforceEndpointMapping` (ships `false`; turn on only after the
licence backfill has been verified) and `Licensing__UnassignRemovesEndpointMappings`
(`true` — revoking a licence also drops that user's endpoint mappings).

### SFM.Nucleus.API (26)

    AppleWallet__CertificatePassword
    BrandEmail__defensely__Demo__Password
    BrandEmail__defensely__Sales__Password
    BrandEmail__defensely__Support__Password
    BrandEmail__effortly__Demo__Password
    BrandEmail__effortly__Sales__Password
    BrandEmail__effortly__Support__Password
    BrandEmail__innovly__Sales__Password
    CardBlobStorage__ConnectionString
    CardBlobStorage__StorageAccountKey
    ConnectionStrings__ClientA
    ConnectionStrings__ClientB
    ConnectionStrings__DefaultConnection
    ConnectionStrings__MasterCommonConnection
    ConnectionStrings__MasterDlpConnection
    ConnectionStrings__TenantAdminConnection
    ConnectionStrings__TenantTemplate
    ConnectionStrings__WebsiteConnection
    DigitalCard__EmailPassword
    EmailPassword
    JwtSecurityKey
    KbBlobStorage__ConnectionString
    KbBlobStorage__StorageAccountKey
    MicrosoftGraph__ClientSecret
    WhatsApp__AccessToken
    WhatsApp__WebhookVerifyToken

---

## Signed requests

Separate from the above, and already agreed. Three secrets, each set in exactly
two places — the caller that signs and the host that verifies. Generate three
distinct values; sharing one defeats the point, because then a leak from any
caller forces every caller to rotate at once.

| secret | verifying host | signing caller |
|---|---|---|
| 1 | `SignedRequests__Clients__ai-service` on Nucleus.API | AI service, sends `X-Api-Client: ai-service` |
| 2 | `SignedRequests__Clients__identity-api` on Nucleus.API | Identity.API: `NucleusSigningSecret` + `NucleusSigningClient=identity-api` |
| 3 | `SignedRequests__Clients__dlp-worker` on DLP.API | worker: `DlpApiOptions__SigningSecret` (client name defaults to `dlp-worker` in code) |

Identity.API's two settings have no `__` prefix: `AppSettings` binds to the
configuration root, not a section.

Set the verifying host before the signing caller. The reverse order produces
`unknown client` warnings — harmless while `Mode=Audit`, but noise someone then
has to explain.

Both hosts are `Mode=Audit`, which logs the outcome and lets every request through
regardless. Nothing is enforced until that becomes `Enforce`, and nothing should
become `Enforce` until the log shows `valid=true` for that caller. Successful
checks log at Debug, failures at Warning, so at the default Information level a
working caller and a middleware that never ran look identical. Set
`Logging__LogLevel__SFM.Common.Utilities.Security=Debug` while verifying.

One thing to expect: APIM sits in front of Nucleus with a `/qanucleus` prefix. The
signature covers the path, so if APIM forwards that prefix instead of stripping
it, the signature will not match and the log will say `invalid signature` — which
reads like a wrong secret but is not.

---

## Also needed

## Storage target credentials (DLP.API)

    Storage__CredentialEncryptionKey    base64, exactly 32 bytes (AES-256)

Encrypts the secret columns of `dlp."StorageTarget"` — the per-purpose storage
backends behind `POST /storage/targets/save`. Callers send the real connection
string, account key or S3 secret key; the API encrypts with this before the row
is written, and `StorageTargetResolver` decrypts with it on the way back out.

Falls back to `AgentCredentials__EncryptionKey` if unset, so a host that already
has one key does not need a second. Set it explicitly if you would rather the
two rotate independently — sharing it means rotating the agent key also orphans
every stored storage credential.

Without either, `POST /storage/targets/save` returns **400** rather than writing
a credential in the clear. The read path is more forgiving on purpose: a value
that will not decrypt is used as-is and logged at Warning naming the row and
column, so a target hand-seeded during setup still works and a rotated key
surfaces as a storage authentication failure rather than a request that dies
before it reaches storage. Check the logs for `stored as plaintext` after
seeding by hand.

Generate with `SecretProtector.GenerateKey()` or `openssl rand -base64 32`.

Rotating it has the same shape as the agent key: every stored storage secret was
encrypted with the old one. Re-save each target through the endpoint after
rotating — there are only as many rows as the tenant has purposes, so this is a
short job, unlike re-enrolling devices.

## Agent credentials (ExeApi)

Two settings, both Application settings on the Function App. Without them the two
agent endpoints return 503 and refuse to operate — deliberately, because the
alternative is storing a secret unprotected "for now".

    AgentCredentials__EncryptionKey    base64, exactly 32 bytes (AES-256)
    AgentJwt__SigningKey               any long random string

Generate the encryption key with `SecretProtector.GenerateKey()` so it is produced
by the code that consumes it, or `openssl rand -base64 32`.

`AgentJwt__SigningKey` is deliberately **not** `JwtSecurityKey`. That value is
shared by DLP.API, Identity.API and Nucleus.API, and all three validate issuer and
audience — so a distinct audience alone would keep agent tokens out of them. A
separate key means a mistake in audience validation, in any of the three, still
cannot turn an agent token into console access. It also means rotating one does not
sign out the users of the other.

Rotating `AgentCredentials__EncryptionKey` is not a like-for-like swap: every stored
secret was encrypted with the old key and cannot be read with the new one. Every
device would have to re-enrol. Treat it as a recovery action, not maintenance.

The table this uses is created by `scripts/create_endpointcredential_table.sql`,
which runs once against the Identity database. Until it exists the two endpoints
fail; nothing else does, because nothing else reads that table.

## Also needed

    SFM.DLP.ExeApi   EnableSwagger=true    on the QA function app only

Swagger and OpenAPI are refused unless this reads exactly `true`. Off by default so
a new environment starts closed; without it, QA loses its API documentation.
Production should not have this set.

It goes in the Function App's Application settings — not in `appsettings.json`,
which ExeApi does not read. For local work, add it to `local.settings.json` under
`Values`.

Verify after deploying: `GET /swagger.json` should return 404, and the startup log
should read `Swagger/OpenAPI endpoints are disabled.` A 401 instead of a 404 would
mean the request never reached the check.

---

## Rotate

These are in git history. Setting environment variables does not retire them —
only rotating does.

| what | where it appears |
|---|---|
| `qaaidlpapi` storage key | DLP.API, ExeApi, DLPProcessLaterService |
| `qadlpinstaller` storage key | DLP.API |
| `dlpuninstaller` storage key | DLP.API |
| `qaaidlpapihw` storage key | DLP.API |
| `effortlyassetsprojects` storage key | Nucleus.API |
| Postgres password | every app, several databases |
| `AmazonS3Settings__SecretKey` | DLP.API |
| `Twilio__AuthToken`, `WhatsApp__AccessToken` | DLP.API, Nucleus.API |
| `MicrosoftGraph__ClientSecret` | Nucleus.API |
| `JwtSecurityKey` | DLP.API, Identity.API, Nucleus.API — shared value, so rotating means all three together |

An Azure storage account key grants read, write and delete across every container
in that account, permanently, with no scoping. That is the same property that
makes Phase 7 of the agent security plan the highest-value item on it: the agent
receives one of these keys today, on every laptop.

Rotating `JwtSecurityKey` invalidates every issued token, so every logged-in user
is signed out. Worth doing at a quiet hour rather than skipping.
