# Changing the database

> **Someone asked for a database change?**
> It goes in `SFM.DLP.API/Migrations/Dlp/` as `V000n__name.sql`.
> Not in `scripts/`. Not as SQL pasted into a chat. Not "I'll run it on QA".
>
> **New here?** That one line is the whole convention. The rest of this file is
> why, and the two ways it goes wrong.

**A schema change ships as a migration. Always. That is the default and there is
no second option to weigh up.**

Writing the SQL into `scripts/` and telling someone to run it reaches exactly the
one database that person remembers to point at. We are database-per-tenant —
every customer has their own DLP database — so a hand-run script is a change that
landed on one tenant and silently did not land on the others. The runner exists
to make that impossible.

---

## The one thing to do

Add a numbered file to `SFM.DLP.API/Migrations/Dlp/`:

```
V0004__cloudfile_listitemuniqueid.sql
```

- `V` + **four digits**, then `__`, then a short name. The runner matches
  `^V(\d{4})__(.+)\.sql$` — a file that does not match is ignored silently, so
  check the shape before you wonder why nothing happened.
- Next number after the highest already there. Never reuse or renumber: the
  version is recorded per database once applied, and a reused number is treated
  as already done.
- It is embedded automatically (`<EmbeddedResource Include="Migrations\**\*.sql" />`).
  Nothing to register.

Then after deploy:

```
POST /admin/migration/run
```

That applies every unapplied script to every tenant. `GET /admin/migration/status`
shows where each database has got to.

Three sets exist and they go to different databases — put the file in the right
one:

| Folder | Database |
|---|---|
| `Migrations/Dlp/` | each tenant's DLP database |
| `Migrations/Common/` | each tenant's user database |
| `Migrations/Nucleus/` | the shared Nucleus database |

---

## Never edit a migration that has already run

Got `V0004` wrong? **Ship `V0005` that corrects it.** Do not edit `V0004`.

The runner checksums every script. Change one after it has been applied and it
does not re-run — it logs this and moves on:

```
Migration [db/Dlp] script V0004__... checksum changed — skipping
```

Skipping is the safe half. The dangerous half is that it skips **per database**:

| Tenant | What it ends up with |
|---|---|
| Already applied `V0004` | The **old** content. Warned, then skipped |
| New, or never applied it | The **new** content |

One file, two different schemas, and the only sign is a warning line in a deploy
log nobody reads top to bottom. Six months later two customers behave differently
and nothing in the repository explains why.

The same applies to renaming a file: the runner keys on the name, so a rename is
a brand new script to every database that already ran the old one, and the
original stays applied forever under a name that no longer exists.

## Write it so it can run twice

The runner records what it has applied, but a migration still gets re-run in
practice — a restored database, a tenant added mid-release, a failed run retried.
Use `IF NOT EXISTS` and `ADD COLUMN IF NOT EXISTS` rather than assuming a clean
slate.

## Guard anything a tenant might not have

Most tables exist everywhere. Some do not — `discovery."CloudFile"` only exists
once a tenant has connected a cloud app.

A migration that assumes such a table **fails for every tenant without it**, and
one failing tenant stops the whole fleet receiving any LATER migration. So skip
rather than fail:

```sql
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'discovery' AND table_name = 'CloudFile'
    ) THEN
        RAISE NOTICE 'discovery."CloudFile" not present; skipping V0004.';
        RETURN;
    END IF;

    ALTER TABLE discovery."CloudFile"
        ADD COLUMN IF NOT EXISTS "ListItemUniqueId" text NULL;

    CREATE INDEX IF NOT EXISTS "IX_discovery_CloudFile_ListItemUniqueId"
        ON discovery."CloudFile" ("ListItemUniqueId")
        WHERE "ListItemUniqueId" IS NOT NULL;
END
$$;
```

That is the whole of `V0004`, and it is the shape to copy.

---

## What `scripts/` is still for

Not nothing, but not migrations either:

- **Fresh installs** — `create_cloud_discovery.sql` builds the whole schema for a
  new tenant in one go. Keep the new column in there too, so a fresh database and
  a migrated one end up identical.
- **One-off data repairs** — `migrate_promptlog_status.sql`, a backfill, a
  cleanup. Run deliberately, once, by a person who knows why.
- **Diagnostics** — the verification queries at the foot of most scripts.

If it changes the shape of the schema, it is a migration. If it changes data on
purpose, once, it is a script.

---

## Checklist

1. `Migrations/Dlp|Common|Nucleus/V000n__short_name.sql`, next number up
2. **Never edit or rename one that has already run** — ship the next number
3. `IF NOT EXISTS` everywhere
4. Guarded if the table might be absent on some tenants
5. Same change added to the matching `scripts/create_*.sql` for fresh installs
6. Entity + `QaSfmDlpContext` updated to match
7. `POST /admin/migration/run` after deploy, then
   `GET /admin/migration/status` to confirm every tenant took it

---

## If you are reviewing someone else's change

Three things reject it on sight:

- **SQL in `scripts/` that changes the schema** — it reaches one database, the
  one they remember to point at. Ask for a migration.
- **An edit to an existing `V000n` file** — check `git log` on it. If it has ever
  been deployed, it is already applied somewhere and the edit will not reach
  those tenants.
- **A migration that assumes a table exists** — ask whether every tenant has it.
  `discovery.*` tables do not; a tenant with no cloud connector will fail the run
  and block the whole fleet from every later migration.
