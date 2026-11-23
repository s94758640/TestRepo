# Connecting a provider to Defensely

What has to be created on each provider's side before Defensely can scan it, and
what it costs. Written for whoever will do the sign-ups, with the management
decision stated first because two of them need one.

Two kinds of provider, one wizard. Pick the provider and the fields change.

| | Providers | What a scan answers |
|---|---|---|
| **Part 1 · Cloud storage** | Box · Dropbox · Google Drive · Adobe | Which files hold sensitive data, and who can reach them |
| **Part 2 · Source repositories** | Azure DevOps Repos · GitHub · GitLab · Bitbucket | Whether a live credential is committed in our source |

SharePoint and OneDrive are set up through Entra and are documented separately.

**Every credential in this document is read-only.** None of them can change,
share or delete anything — worth saying first, because it is the first thing any
admin asks.

---

# Part 1 · Cloud storage

Box, Dropbox, Google Drive and Adobe. A scan here answers **which files hold
sensitive data, and who can reach them**.

---

## The short answer on cost

**Only Box can be tested indefinitely for nothing.** The other two give a trial
with a clock, and then they are paid products.

| | account needed | free? | clock | what happens after |
|---|---|---|---|---|
| **Box** | Box Developer Account | **Yes, permanently** | none | nothing — it stays free |
| **Dropbox** | Dropbox **Business** (Standard or Advanced) | trial only | **30 days** | paid per user, minimum seat count |
| **Google Drive** | Google **Workspace** (Business Starter or above) | trial only | **14 days** | paid per user |
| **Adobe Creative Cloud** | existing Adobe org — but see below | **not yet known** | none | nothing to buy; it is not established that this can be built at all |

Consumer tiers do not work for any of this, and it is worth being precise about
why: a personal Google account or a free Dropbox exposes only **per-user OAuth**,
where each individual grants access to their own files. Defensely scans on
behalf of an organisation — one admin credential covering everybody. That
capability exists only in the business tiers. It is not a setting we can switch
on; the API simply is not there.

### Recommendation: build first, then start the trials

The trial clocks should cover **testing**, not development. If a Dropbox trial is
started today, most of its 30 days will be spent while the connector is still
being written, and the clock will run out around the time it becomes useful.

So:

1. **Box now** — free and permanent, so there is no reason to wait. It also
   exercises the same code path the other two use.
2. **Dropbox and Google when their connectors are ready to test.** A day or two
   of notice is enough; both sign-ups are same-day.

### What management is being asked to approve

- **Box** — nothing. No cost, no commitment.
- **Dropbox Business** — a 30-day trial, which normally requires a payment card.
  Someone must own cancelling it or approving the spend before day 30.
- **Google Workspace** — a 14-day trial **and a domain we control**. We already
  own `innovly.co` and `secureforme.io`, so no purchase is needed, but a DNS TXT
  record has to be added to prove ownership. That needs whoever holds DNS.
- **Adobe** — the **Developer role** on the Adobe org we already have. Free, and
  granted by an Adobe System Administrator. Nothing to purchase; it only makes
  the API catalogue visible so the question can be answered at all.

A question worth settling early: whether these stay as permanent internal test
tenants, or are only stood up to verify each connector and then dropped. The
second is cheaper and is enough to ship; the first is what lets us reproduce a
customer problem six months later without asking them for access.

---

## Box

Free, permanent, and the only one with no clock. Start here.

### Create it

1. Go to **developer.box.com** and sign up for a **Box Developer Account**. This
   creates a self-contained test enterprise with its own Admin Console — which
   is the part that matters, because it is what makes admin-level API access
   possible at all.
2. **Dev Console → Create Platform App → Custom App**.
3. Authentication method: **Server Authentication (Client Credentials Grant)**.

   Box also offers JWT with a keypair. Both work; CCG is chosen because the
   credential is just an ID and a secret, with no PEM private key or passphrase
   to paste wrong.
4. On the app's **Configuration** tab:
   - **App Access Level:** *App + Enterprise Access*
   - **Application Scopes:** tick
     - *Read all files and folders stored in Box*
     - *Manage users* — this is what enumerates whose accounts exist
   - **Advanced Features:** tick **Generate user access tokens**

     Without this the app can authenticate but cannot read any individual's
     content, and the scan finds nothing while reporting success.
5. **Authorization** tab → **Review and Submit**.
6. **Admin Console → Platform → Platform Apps** → authorise the app. On a
   developer account you are the admin, so you are approving your own request.

   Box's own documentation still says "Admin Console → Apps", and some tenants
   show it as *Integrations → Custom Apps Manager*. On a current developer
   enterprise it is **Platform**, and the page lists each app with its
   Authorization state, Enablement, and Client ID — which is also the quickest
   way to confirm the right app is the one authorised.

   **How to tell it worked without guessing:** authorising an app makes Box
   create a service account for it, named `AutomationUser_<id>_<suffix>`, which
   then appears in **Content → Content Manager** alongside the real users. If
   it is there, the app is authorised.

   That account's own drive is always empty, and that is correct — the app never
   stores anything of its own. It reads each real user's content by acting as
   them, which is what *Generate user access tokens* enables. An empty service
   account is expected; a scan that finds nothing is the symptom of that setting
   being off.

### What to send us

From **Configuration → OAuth 2.0 Credentials**, and **Admin Console → Account &
Billing** for the third:

| field | where |
|---|---|
| Client ID | app's Configuration tab |
| Client Secret | same (shown once — copy it then) |
| Enterprise ID | Admin Console → Account & Billing |

Nothing else. Box needs no extra fields in the connect wizard.

> **If a change is made to scopes later, the app must be re-authorised in the
> Admin Console.** Box does not apply scope changes to an already-approved app
> until it is re-submitted, and the symptom is a permission error on a call that
> worked yesterday.

---

## Dropbox Business

30-day trial. Do not start it until the connector is ready to test.

### Create it

1. **dropbox.com/business** → start a free trial of **Standard** or
   **Advanced**. A payment card is normally required. Note the date — the trial
   ends 30 days later whether or not anyone is watching.
2. **dropbox.com/developers/apps → Create app**:
   - **Scoped access**
   - **Full Dropbox** — not App folder. App folder restricts the app to its own
     directory, which is the opposite of what a scanner needs.
3. **Permissions** tab, tick:

   | scope | why |
   |---|---|
   | `team_info.read` | the team's own identity |
   | `members.read` | enumerate team members |
   | `team_data.member` | act on a member's behalf |
   | `files.metadata.read` | list files |
   | `files.content.read` | read them for classification |
   | `sharing.read` | shared links — the Public Links screen |

   Submit the permission change before generating any token. A token issued
   before a scope was added does not carry it.
4. **Get a refresh token.** Dropbox has no app-only flow for team data, so a
   team admin authorises once and the resulting refresh token becomes the
   connector's credential. In a browser, signed in as a team admin:

   ```
   https://www.dropbox.com/oauth2/authorize?client_id=APP_KEY&response_type=code&token_access_type=offline
   ```

   `token_access_type=offline` is what makes Dropbox return a refresh token
   rather than a short-lived one. Without it you get a token that expires in
   four hours and a connector that stops working the same day.

   Approve, copy the code, then exchange it (the code is single-use and expires
   in minutes):

   ```
   curl https://api.dropboxapi.com/oauth2/token \
     -d code=PASTED_CODE \
     -d grant_type=authorization_code \
     -d client_id=APP_KEY \
     -d client_secret=APP_SECRET
   ```

   Pass the key and secret as **form fields**, not as `-u APP_KEY:APP_SECRET`.
   Basic auth works in principle, but a stray space copied from the console
   lands inside the base64 and Dropbox answers `invalid_client` — which reads
   as a wrong secret when the secret was fine.

   On Windows use `curl.exe`. In PowerShell, `curl` is an alias for
   `Invoke-WebRequest`, which does not understand `-d`.

   The response contains `refresh_token`. That is the value we need.

   **Done once, not per scan.** A refresh token does not expire. It is stored
   encrypted on the connector and exchanged for a short-lived access token on
   every call automatically — which is exactly what `token_access_type=offline`
   buys. Without it Dropbox returns only a four-hour token and somebody repeats
   this by hand every afternoon.

   Redo it only if a team admin revokes the app, the App secret is regenerated,
   or the scopes change: Dropbox needs fresh consent for a new scope, and an
   existing token keeps working while silently lacking it. Submitting all six
   scopes before this step makes that impossible.

5. **Approve the app for the team — if your console has it.** Dropbox's
   documentation points at *Admin Console → Apps*; that page does not exist on
   every team, and searching the admin console for "app" may turn up only
   sharing settings. It applies when someone other than a team admin created the
   app, or where app-approval enforcement is on. If you are the team admin doing
   the grant, that grant IS the approval — and the token succeeding proves it.

### What to send us

| field | where |
|---|---|
| App key | app's Settings tab |
| App secret | same |
| Refresh token | from step 4 |
| Team ID | optional — we can read it from the API |

---

## Google Drive (Workspace)

14-day trial, and the only one that needs a domain.

### Create it

1. **First, check what kind of Google account you have.** This decides whether
   anything below is possible, and it is not obvious from the address.

   Sign in and open **admin.google.com**:

   | what happens | what it means |
   |---|---|
   | An admin console opens | Workspace, and you are an administrator. Skip to step 2 |
   | "You're not an administrator" | Workspace, but on somebody else's account. Get a super-admin, or have steps 5–6 done for you |
   | It bounces you out entirely | A personal Google account. Nothing below will work |

   The last case is the one to catch now. A personal account has no admin
   console, so there is no domain-wide delegation to grant — and delegation is
   the entire mechanism by which a service account reads anybody's Drive. An
   ordinary Gmail can only grant access to *its own* files, one user at a time,
   which is not what a scanner does.

   If a Workspace tenant is needed: **workspace.google.com** → start a trial,
   supplying a domain you control and proving it with a **DNS TXT record**. We
   own `innovly.co` and `secureforme.io`; a spare subdomain keeps it clear of
   production mail.

   > Do not use a domain that carries live email. Workspace sign-up will want to
   > take over MX records for it.
2. **console.cloud.google.com** → create a project.
3. **APIs & Services → Library** → enable:
   - **Google Drive API**
   - **Admin SDK API**
4. **IAM & Admin → Service Accounts → Create service account.**
   - No roles needed. Project roles govern Google Cloud resources; access to
     Workspace data comes entirely from step 6.
5. On the service account: **Keys → Add key → Create new key → JSON**. Download
   it. This file is a credential — treat it like a password, and it is only
   downloadable once.
6. **Enable domain-wide delegation** on the service account, and copy its
   numeric **Client ID** (the "Unique ID", not the email).

   Then in the **Workspace Admin console** → **Security → Access and data
   control → API controls → Domain-wide delegation → Add new**:

   - **Client ID:** the numeric ID from above
   - **OAuth scopes**, comma-separated, exactly:

     ```
     https://www.googleapis.com/auth/drive.readonly,
     https://www.googleapis.com/auth/admin.directory.user.readonly
     ```

   This step is the one that actually grants access, and it is done in the
   Workspace admin console, not in Google Cloud. The two consoles look similar
   and doing it in the wrong one is the most common way this fails.

### What to send us

From the downloaded JSON key, plus two values you choose:

| field | where |
|---|---|
| `client_email` | in the JSON |
| `client_id` | in the JSON (the numeric one) |
| `private_key` | in the JSON — the whole `-----BEGIN PRIVATE KEY-----` block |
| Admin user to impersonate | a Workspace super-admin address |
| Customer ID or primary domain | Admin console → Account settings |

> **Why an admin to impersonate.** A service account owns no Drive of its own.
> Every read happens *as somebody*, and listing the domain's users happens as an
> admin. If that address is not a super-admin, user enumeration returns nothing
> and the scan completes successfully having found no accounts at all.

### After connecting — what to check

Google fails more opaquely than the others: nearly every mistake comes back as
`invalid_grant` with no detail, so the order matters.

| symptom | almost always |
|---|---|
| Test fails, `invalid_grant` | Delegation not authorised in the **Workspace** admin console (people do it in Google Cloud by mistake), the two scopes not granted there, or the impersonated user not a real super-admin |
| Private key rejected | The JSON key's `\n` escapes pasted literally instead of as real line breaks |
| Test succeeds, **zero users** | The impersonated address is not a super-admin |
| Files found, Google Docs empty | Expected only for Drawings and Forms — Docs, Sheets and Slides are exported as text and should carry content |

The third row is the dangerous one, because it looks like success. A scan that
enumerated nobody reports the organisation as holding no sensitive data. If the
user count is zero, treat it as a failure whatever the status says.

---

## Adobe Creative Cloud

Not like the other three, and worth stating before anybody starts a sign-up.

The providers above all share one thing: a documented, admin-level API where **a
single credential can enumerate every user's files and read their contents**.
That is the entire basis of a connector. Adobe is not known to publish that for
Creative Cloud storage.

What Adobe does expose at enterprise level:

- **User Management API** — users, groups, entitlements. Real and well
  documented, but it lists *people*, not files.
- **Admin Console storage reports** — per-user asset inventory exported from the
  console. A report rather than an API, and it carries file names without
  contents.
- **Asset transfer on offboarding** — an admin can move a departing user's folder
  to someone else, through the console UI. One user at a time, no API.
- **Photoshop / Lightroom / InDesign APIs** — these act on assets handed to them,
  authorised per user. They do not enumerate an organisation.

None of that gives the four operations a scan needs. So what follows is a
**check**, not a setup guide: this should be settled with evidence from our own
Adobe org rather than anybody's assumption, and it costs nothing to run.

### Settling it

1. **Get the Developer role.** The Developer Console currently reports *"You do
   not have developer access and need admin approval"*, so the API catalogue is
   not visible at all. An Adobe **System Administrator** grants it in Admin
   Console → Users. **Free** — a role on an existing subscription, not a new
   product. This is the only thing to ask management for at this stage.

2. **Open the API catalogue.** developer.adobe.com/console → confirm the org →
   Create new project → **Add API**.

   **This list is the answer.** Adobe does not allow a bare credential — one
   exists only attached to an API — so what appears here defines what any Adobe
   credential could ever do.

3. **Create an OAuth Server-to-Server credential.** Not JWT; Adobe retired
   those. The credential page then shows the Client ID, a *Retrieve client
   secret* button, the granted scopes, and the Organization ID in
   `XXXXXXXXXXXX@AdobeOrg` form.

4. **Read what it can actually touch:**

   ```
   curl -s -X POST https://ims-na1.adobelogin.com/ims/token/v3 \
     -d grant_type=client_credentials \
     -d client_id=CLIENT_ID \
     -d client_secret=CLIENT_SECRET \
     -d scope=openid,AdobeID,read_organizations
   ```

   The access token is a JWT; its middle segment decodes to JSON with a `scope`
   claim. That claim — not the console, not what anyone was told — is the
   definitive statement of what the credential may do.

### Reading the result

| Scopes include | Meaning |
|---|---|
| assets / storage / files / content | Worth pursuing — read the docs and plan a connector properly |
| only user management, openid, organizations | Manages people, not files. No connector is possible from it |
| AEM scopes | Adobe Experience Manager Assets, which **is** scannable — a different and real path |

### Two things that could make this moot

- **Is it actually AEM Assets?** Adobe Experience Manager Assets is a different
  product with a proper HTTP API over its asset store, and is genuinely
  scannable. "Adobe assets" is used for both and they are routinely confused.
- **Where do the files really sit?** Many organisations have Creative Cloud
  syncing into SharePoint, Google Drive or Box. Where that is true we already
  scan them, and no Adobe connector is needed.

### Settled — checked on a real Adobe organisation, 2026-09-28

The Developer Console's "Add an API" catalogue for `designer-intechhub.com`
contains, in full:

> PDF Services · PDF Embed · Photoshop (Firefly) · Lightroom · Substance 3D ·
> Remove Background · Smart Content · Places · Privacy Service ·
> Primetime Ad Insertion · User Management

**No storage, assets or file API of any kind.** A credential can only ever be
attached to something in that list, so that list is the ceiling on what any
Adobe credential could do.

**User Management was itself disabled** on that organisation — it needs an
entitlement not in every plan. It does not matter: User Management lists
*people*, not files. No Adobe subscription unlocks scanning, so this is not a
licensing gap that can be bought around. The interface does not exist.

**Conclusion: an Adobe connector cannot scan, and that is now established rather
than assumed.** The connector stays registered with `SupportsContentScan =
false`, which is exactly what it was built to express — it reports what a
credential may do and refuses a scan rather than completing one having read
nothing.

The remaining useful question for any customer asking about Adobe is where their
Creative Cloud content actually lives. Where it syncs into SharePoint, Drive, Box
or Dropbox — which is common — it is already covered.

---

---

# Part 2 · Source repositories

Azure DevOps, GitHub, GitLab and Bitbucket. A scan here answers a different
question from Part 1 — not *who can reach this file*, but **is there a live
credential committed in our source**.

---

## What the scan looks for

Credentials in source and configuration: AWS keys, GitHub and Slack tokens,
private key blocks, Azure storage connection strings, passwords assigned in
config files. Then, as a second pass, the same classifications the rest of the
product uses — PII, PHI and the others — because a CSV of real customer data
committed to a repository is the same problem on a drive and in git.

**Two things to say out loud before anyone demonstrates this.**

It reads the **current state of the default branch**. A secret deleted last week
is not in what we scan — but it is still in git history, and the key is still
live. So a clean result means *clean at HEAD*, never *clean*, and the fix for
every finding is **rotate the credential**, not delete the line.

It is **read-only**. Every credential below reads repository contents and
nothing else. We cannot push, cannot open pull requests, cannot change settings
and cannot delete anything.

---

## Azure DevOps Repos

Two ways to connect. **A service principal is the better one** and is what we
recommend — it belongs to the organisation rather than to a person, and it does
not expire. A personal access token is quicker if you just want to see a scan
run.

### Option A — service principal (recommended)

The same kind of Entra app registration you already use for SharePoint and
OneDrive, so this will be familiar.

1. **Entra admin centre → App registrations → New registration.** Name it
   something recognisable; no redirect URI is needed.
2. **Certificates & secrets → New client secret.** Copy the value — it is shown
   once. Note the expiry you chose.
3. Copy the **Application (client) ID** and the **Directory (tenant) ID** from
   the app's Overview page.
4. **In Azure DevOps: Organization settings → Users → Add users.** One dialog
   does all of it:

   | Field | Value |
   |---|---|
   | Users or Service Principals | The app registration's name, or paste its **Application (client) ID** if the name does not resolve |
   | **Access level** | **Basic** |
   | Add to projects | The projects you want scanned |
   | **Azure DevOps Groups** | **Project Readers** |

   Setting the group here means the principal lands read-only in every project
   you ticked. There is no need to visit each project's Permissions page
   afterwards — only to add a project later, or to check what it actually has.

> **Access level must be Basic, not Stakeholder.** Stakeholder is free and has
> **no access to Repos at all**, so the principal authenticates perfectly and
> lists zero repositories — which looks exactly like a clean scan.

> **Step 4 is the one that gets missed entirely.** A valid app registration that
> was never added to the organisation fails the same silent way. The connector
> test catches both by listing what it can actually see, which is why you should
> run the test before trusting any scan.

**No API permissions are needed on the app registration.** Unlike Graph, Azure
DevOps authorises by what the service principal is granted *inside the
organisation*, not by consented scopes. Nothing to approve in Entra beyond
creating the app.

### Option B — personal access token

1. In Azure DevOps, click your profile picture → **Personal access tokens** →
   **New Token**.
2. **Organization:** the one you want scanned.
3. **Expiration:** a PAT lives at most one year. Note the date — when it
   expires, scanning stops and the connector reports an authentication error.
4. **Scopes:** choose *Custom defined*, then tick **Code → Read** and nothing
   else.
5. Create, and copy the token. Azure DevOps shows it **once**.

> **If you use a PAT, create it from a service account rather than a person.** A
> PAT carries its owner's access and dies with their account — so scanning stops
> the day they leave, and the failure arrives weeks later as an authentication
> error with nobody remembering why.

### What to send us

| Field | Option A — service principal | Option B — token |
|---|---|---|
| Organisation | The part after `dev.azure.com/` in your URL | same |
| Directory (tenant) ID | From the app's Overview page | **leave blank** |
| Application (client) ID | From the app's Overview page | leave blank |
| Client secret / token | The client secret | The personal access token |
| Project | A single project, or **blank for all** | same |
| Server URL | Blank for cloud; for Azure DevOps **Server**, your base URL | same |

The **Directory (tenant) ID is what selects the mode.** Fill it in and we connect
as a service principal; leave it blank and the secret is treated as a token.

Leave Project blank unless there is a reason not to: a project added next month
is picked up automatically, where a named one has to be remembered and edited.

### After connecting — what to check

Run the connector test and compare the **repository count** against what you
expect. A credential scoped more narrowly than intended produces a successful
scan of a fraction of the organisation, which looks identical to a clean result.

If the count is lower than it should be:

- **Service principal, zero repositories** — either it was never added to the
  organisation, or its access level is **Stakeholder**, which has no Repos
  access at all. Both authenticate cleanly.
- **Service principal, some repositories** — it is not on every project.
  Organization settings → Users → the principal → *Manage projects*, or add it
  to **Readers** in the missing project's own Permissions page.
- **Token** — its owner cannot see the missing projects. A PAT never grants more
  than the person who made it already has.

---


## GitHub

> **Not yet verified against a real GitHub organisation.** The connector is
> built and tested, but every section above earned its accuracy by being run
> before it was written, and this one has not been. Treat the scope lists as
> correct and the step numbering as approximate; tell us where it is wrong and
> we will fix it the same day. The sections for Box, Google and Azure DevOps
> each gained a correction after their first real connection.

Two ways to connect, and the choice matters more here than anywhere else.

**A GitHub App is the only way to be read-only.** A classic personal access
token's `repo` scope is read **and** write — GitHub offers no read-only
equivalent for private repositories. So a token connector holds a credential
that *could* push, even though we never do. An App can be granted
`Contents: Read-only`, and its access belongs to the organisation rather than to
a person, so it survives someone leaving and carries a far higher rate limit.

A token is still supported, because creating an App needs an organisation owner
and that is not always available on the first afternoon.

### Option A — GitHub App (recommended)

An organisation owner has to do this.

1. **Organisation → Settings → Developer settings → GitHub Apps → New GitHub
   App.** Name it something recognisable. A homepage URL is required; anything
   valid will do.
2. **Webhook → untick *Active*.** We never receive events, and leaving it on
   means GitHub retries deliveries to a URL that does not exist.
3. **Permissions → Repository permissions.** Set exactly two, and nothing else:

   | Permission | Access |
   |---|---|
   | **Contents** | **Read-only** |
   | **Metadata** | **Read-only** *(GitHub selects this automatically)* |

4. **Where can this App be installed** — *Only on this account*.
5. Create it, then on the App's page: **Generate a private key.** A `.pem` file
   downloads. This is the credential, and GitHub keeps no copy.
6. Note the **App ID** — the number near the top of the App's settings page.
7. **Install App → your organisation → All repositories.**

> **Step 7 is the one that gets missed.** An App that exists but is not
> installed authenticates perfectly and can read nothing — the same silent
> failure as a Stakeholder licence in Azure DevOps. Choosing *Only select
> repositories* and then selecting none does the same thing.

> **The App ID is not the Client ID.** They sit near each other on the same
> page. The App ID is a short number; the Client ID begins `Iv1.` or `Iv23`.
> We need the App ID.

### Option B — personal access token

1. **Your profile → Settings → Developer settings → Personal access tokens →
   Tokens (classic) → Generate new token.**
2. **Scope:** tick **`repo`**. Nothing else.
3. Copy the token — GitHub shows it once.
4. **If your organisation enforces SAML SSO**, use *Configure SSO* beside the
   token and authorise it for the organisation. Without this the token works on
   your own repositories and returns nothing for the organisation's.

> **Create it from a service account rather than a person.** A token carries its
> owner's access and dies with their account, so scanning stops the day they
> leave and the failure arrives weeks later as an authentication error.

### What to send us

| Field | Option A — App | Option B — token |
|---|---|---|
| Organisation | The part after `github.com/` — just the name | same |
| App ID / username | The **App ID** number | **leave blank** |
| Private key / token | The whole `.pem`, including its BEGIN and END lines | The token |
| Server URL | Blank for github.com; for **Enterprise Server**, your install's base URL | same |

**The private key is what selects the mode.** Paste a PEM block and we connect
as the App; paste anything else and it is treated as a token.

For Enterprise Server, give the plain base URL — `https://github.company.com`.
Do not add `/api/v3`; we append it.

### After connecting — what to check

Run the connector test and compare the **repository count** against what you
expect.

- **Zero, using an App** — it is not installed on this organisation, or the
  installation grants *selected repositories* and none are selected.
- **Zero, using a token** — the organisation name is wrong, the token lacks
  `repo`, or SAML authorisation was not done.
- **Fewer than expected, using a token** — its owner cannot see the rest. A
  token never grants more than the person already has.

---

## GitLab

One credential type, and one decision: **make it a group token, not a project
token.** A project token has to be added to every project, so a project created
next month is silently never scanned and nothing reports it. A group token
covers the group, its subgroups, and projects that do not exist yet.

### Create it

A group **Owner** has to do this.

1. **Group → Settings → Access tokens → Add new token.**
2. **Role: Reporter.** Lower roles cannot read repository contents.
3. **Scopes — tick both:**

   | Scope | Why |
   |---|---|
   | `read_api` | Lists the group's projects |
   | `read_repository` | Reads their contents |

4. Set an expiry you will remember. GitLab requires one.
5. Create, and copy the token — it is shown once. Group tokens begin `glpat-`.

> **Both scopes, or the failure is confusing.** With `read_api` alone the
> connector lists every project and then fails to read any of them, which looks
> like a problem with the repositories rather than with the token.

> **It must be created from the group's own Settings page**, not from a user's
> profile. A personal access token also works, but it carries that person's full
> access across every group they belong to — far more than this needs, and it
> dies with their account.

### What to send us

| Field | Value |
|---|---|
| Group ID or path | The part after `gitlab.com/` — `acme`, or `acme/platform` for a subgroup. The numeric Group ID from the group's page also works |
| *(client id)* | **Not used — leave blank** |
| Group access token | The `glpat-…` value |
| Server URL | Blank for gitlab.com; for self-managed, your install's base URL |

### After connecting — what to check

Compare the project count with what you expect. We list the group **including
its subgroups**, and skip archived projects — an archived project is read-only
at GitLab, so findings in it cannot be fixed by anyone.

- **Zero projects** — usually the group path. It is the URL path, not the
  group's display name, and they often differ.
- **Fewer than expected** — the token's role is below Reporter on some
  subgroups, or those projects are archived.

---

## Bitbucket

### Create it

A **workspace admin** should create this from their account.

1. **Profile → Personal settings → App passwords → Create app password.**
2. Label it something recognisable.
3. **Permissions — tick only *Repositories: Read*.** Nothing else. Write and
   admin permissions are not needed and should not be granted.
4. Create, and copy it. Bitbucket shows an app password **once**.

> **Two things are easy to get wrong, and both produce the same 401.**
>
> The username is the **Bitbucket username**, not the email address used to sign
> in. They are different, and the email is rejected. Find it under *Personal
> settings → Account settings*.
>
> The workspace is the **workspace ID** from the URL — the part after
> `bitbucket.org/` — not the workspace's display name. A workspace called
> "Acme Corporation" usually has the ID `acme-corp`.

### What to send us

| Field | Value |
|---|---|
| Workspace | The workspace **ID** from your browser's URL |
| Username | The Bitbucket username — **not** the email address |
| App password | The value copied in step 4 |
| Server URL | Blank for bitbucket.org; for **Data Center**, your install's base URL |

On Data Center, use an **HTTP access token** with *Repository Read* in place of
the app password.

### After connecting — what to check

Compare the repository count with the workspace.

- **Zero** — almost always the workspace ID or the username/email confusion.
- **Fewer than expected** — the account has not been granted access to every
  repository in the workspace. An app password never grants more than its owner
  already has.

---

---

# Part 3 · Sending credentials, and what to expect



## How to send the credentials

A cloud credential reads every file in the tenant. A repository token reads
every line of the source. **Neither goes by email or chat.**

Put them straight into the Defensely connect wizard, which encrypts each one
before it is stored — no endpoint ever returns them afterwards.

If one must be handed over instead, use a secret manager or a one-time secret
link, and rotate it once the connector is confirmed working.

There is a certain irony in emailing a token to a tool whose job is finding
tokens people emailed. It is worth not doing.

---

## What we can and cannot see — cloud storage

Worth setting expectations before the screens are demonstrated. Connecting a
provider gives us **what exists, what is in it, and who can reach it**:

- every file, its owner, size, path and type
- classification of the contents — PII, PHI, PFI and the rest
- sharing: public links, org-wide links, named people, and whether any of them
  are outside the organisation
- who created each link and when — **Microsoft only, and only for links created
  in the last 14 days.** See below.

It does **not** give us how far a link has travelled outside the storage
provider. Whether a public Drive link was pasted into Slack, a Teams channel or
an email thread is not something Google, Box or Dropbox knows — it lives in
those other tools. The same applies to counts of how many times an anonymous
link has been opened, which comes from each provider's separate audit feed.

Both are real features; both are separate work from connecting a provider, and
neither arrives just because a connector is switched on.


### Who created a link, and how old it is

Two columns on the Public Links screen — **Link Created By** and **Age** — come
from somewhere different to everything else, and are worth understanding before
anybody asks why they are blank.

Microsoft Graph does **not** return the creator of a sharing link to an
application credential. The field simply is not in the response. So the names
come from the Microsoft 365 **audit log** instead, matched back to each link.

**What that needs:**

| | |
|---|---|
| Permission | `ActivityFeed.Read` under **Office 365 Management APIs**, application permission, with admin consent |
| Subscription | The `Audit.SharePoint` content type must be **started** on the tenant. An unsubscribed feed returns nothing and reports no error |
| Provider | SharePoint and OneDrive only |

> **The audit log serves 14 days. Not 90.** Microsoft refuses a start time more
> than a fortnight in the past (`AF20055`). Retention is how long events are
> kept; this is how far back the content endpoint will hand them over, and they
> are different numbers.
>
> So a link created more than 14 days before its first attribution attempt can
> **never** be given a creator or an age. Nothing recovers it — not a re-scan,
> not a new permission, not more history. Those rows show `--` permanently.

**What this means in practice:** connect a provider and the first scan will show
`--` in both columns for everything it finds, because those links already exist
and were created some time ago. From then on, each scan attributes the links
created since the last one, and the columns fill in going forward.

**Box, Dropbox and Google show `--` always.** Box reports the file's creator
rather than the link's, Google reports the person the file was shared *with*;
both are the wrong person, so neither is shown. A name that is confidently wrong
on a security screen is worse than a blank one.



## What we can and cannot see — source repositories

Connecting a repository host gives us **every file on the default branch as it
stands today**, scanned for committed credentials and for the same
classifications the rest of the product uses. Each finding carries the file, the
line and which rule matched.

It does **not** give us:

- **git history** — a secret deleted from `main` last week is invisible to us
  and still live in the repository
- **unmerged branches** — a secret on a feature branch that never merges is
  never seen
- **whether the key still works** — we report that a credential was committed,
  not that it is currently valid

So a clean result means **clean at the head of the default branch**, and the fix
for every finding is to **rotate the credential**, not to delete the line.
Deleting it removes the finding from the next scan and changes nothing about the
key.

That distinction is worth saying out loud in any demo. It is the difference
between a useful answer and a reassuring one.

