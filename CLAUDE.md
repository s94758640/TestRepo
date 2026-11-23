# Defensely-WebAPI — instructions for AI assistants and new developers

## The shared scanner engine lives IN this repo under `ScannerEngine/`

The detection/scanning libraries are **vendored** here as ordinary projects —
NOT a submodule. (They used to be the `SecureForMe/DLP-RegexFilter` submodule at
`shared/`; that was removed because the gitlink kept breaking QA builds — a pin to
a commit that was never pushed gave `TF401035: object does not exist`, and an
earlier commit staged the gitlink as a deletion. So the whole `shared` submodule
+ `.gitmodules` were dropped and the source was copied into `ScannerEngine/`.)

`ScannerEngine/` contains (referenced via `ProjectReference ..\ScannerEngine\<name>\…`):
`DLP.RegexFilter` (detection engine + patterns), `DLP.Scanning.Shared`
(severity ladder/DPDP), `DLP.Database.Connectors` (SQL Server, PostgreSQL,
MySQL/MariaDB, Oracle, MongoDB), `SFM.DLP.CloudClients` (SharePoint/OneDrive/
Box/Dropbox/Google drive clients).

Rules now:
1. Edit these files **directly in `ScannerEngine/`** like any other project —
   no submodule dance, no `git submodule update`, no PAT, no pinned commit.
2. There is **no `.gitmodules`** in this repo. If something tries to re-add a
   `shared` submodule, don't.
3. The **agent repo (`Defensely-EndpointDC-Agent`) still uses the `shared`
   submodule** for the same engine. Until it is also vendored, a change to the
   engine made here must be mirrored there (and vice versa) — the two copies can
   drift, which is the cost of dropping the submodule. Keep them in step.

## Repo basics

- Solution: `Defensely.sln` (~30 projects, net8.0; Functions apps net10).
  Build: `dotnet build Defensely.sln`. Tests:
  `dotnet test SFM.Platform.Storage.Tests/...` (one known failure: duplicate
  V0022 migration, pre-existing).
- Layering: Controller → Application service → Infrastructure repo →
  `QaSfmDlpContext` (PostgreSQL, **database per tenant**, selected by the
  `ClientGUID` header). Migrations: forward-only SQL in
  `SFM.DLP.API/Migrations/{Dlp,Common,Nucleus}` — next free `V00nn__` number;
  guard CREATEs INSIDE the `DO $$` block (see V0028's comment).
- `SFM.DLP.API` must NEVER reference `DLP.RegexFilter` (native OCR binaries);
  classification runs in `DLPProcessLaterService` via `ICloudContentAnalyzer`.
- The database crawler registration is split on purpose:
  `AddDataDiscovery()` (validator, both hosts) vs `AddDataDiscoveryCrawler()`
  (worker only — depends on the analyzer). Registering the crawler in the API
  crash-loops the container (validate-on-build; happened 18 Nov).
- Every new API route must be **registered in APIM** before QA can reach it —
  an unregistered route returns a gateway 404 that looks like a wrong URL.
- Evidence rules: `SensitiveContent` is masked ON WRITE
  (`SensitiveValueMasker`), sensitive+unclassified rows only, clean items are
  counters; dashboards read precomputed summary tables, never COUNT(*) over
  inventories.
- Pipelines (`azure-pipelines-dlp-{api,worker,functions}.yml`) fetch the
  submodule in an explicit step with a `SHARED_REPO_PAT` secret-variable
  fallback; they trigger on `shared` and `.gitmodules` path changes. Deploy =
  merge to `Deployment/CICD-QA` (then CICD-Prod1/2).

## Key docs (in the agent repo, Defensely-EndpointDC-Agent)

`UNIVERSAL-DATABASE-DLP-SCANNER-ARCHITECTURE.md` · `DEPLOY-AND-TEST-RUNBOOK.md`
· `SHARED-REPO-CONNECTION.html` · `CENTRALIZED-SCANNER-APPROACH.html`
