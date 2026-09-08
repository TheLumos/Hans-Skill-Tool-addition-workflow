# NanoClaw setup notes

For whoever is building an agent in this install — human or Claude Code. Five
things that cost real time the first time round. Read before you start.

---

## 1. The persona file is `instructions.prepend.md`

Per-agent persona goes in:

```
nanoclaw-v2/groups/<folder>/instructions.prepend.md
```

That is `PERSONA_PREPEND_FILE` in `src/group-persona.ts`, and it is prepended to
the system prompt on every spawn.

**Do not use `groups/<folder>/CLAUDE.md`.** It looks like the obvious choice and it
is regenerated on every spawn, so anything written there is silently destroyed.
`CLAUDE.local.md` is an older fork convention and is not read by this version
either.

Writing the file before creating the agent is safe: the group-init helper uses an
exclusive-create flag, so it never overwrites an existing persona.

---

## 2. `ncl groups config add-mount` cannot create a writable mount

This one is not obvious and there is no error. The CLI writes `readonly: true` when
given `--ro` and **omits the key entirely otherwise**
(`src/cli/resources/groups.ts`), while the validator grants read-write only on a
literal `false` (`src/modules/mount-security/index.ts`):

```js
const requestedReadWrite = mount.readonly === false;   // undefined !== false
```

So every mount added through the CLI comes out read-only no matter what the
allowlist permits. The symptom is an agent correctly reporting it cannot write to a
folder you believe is writable.

**Fix — set the row directly.** Save as `set-mounts.ts` in the nanoclaw-v2 root and
run with `pnpm exec tsx set-mounts.ts`:

```ts
import { CENTRAL_DB_PATH } from './src/config.js';
import { getContainerConfig, updateContainerConfigJson } from './src/db/container-configs.js';
import { initDb, closeDb } from './src/db/connection.js';
import { runMigrations } from './src/db/migrations/index.js';

const GID = '<group id from `ncl groups list --json`>';
const db = await initDb(CENTRAL_DB_PATH);
await runMigrations(db);
await updateContainerConfigJson(GID, 'additional_mounts', [
  { hostPath: '<absolute host path>', containerPath: '<relative name>', readonly: false },
]);
console.log((await getContainerConfig(GID))!.additional_mounts);
closeDb();
```

Delete the script afterwards. Note this replaces the whole list, so include every
mount the agent should have.

---

## 3. Mounts outside the allowlist are silently dropped

`~/.config/nanoclaw/mount-allowlist.json` ships with `allowedRoots: []`. Any mount
whose host path is not under an allowed root is discarded **with no error** — the
agent simply cannot see the folder. This is the most common reason an agent insists
a file is not there.

Write it through the supported path:

```bash
pnpm exec tsx setup/index.ts --step mounts --force -- --json '{"allowedRoots":[{"path":"<absolute host path>","allowReadWrite":true,"description":"<what it is>"}],"blockedPatterns":[]}'
```

Two rules that will otherwise bite:

- **`containerPath` must be relative.** It is force-prefixed with
  `/workspace/extra/`, and absolute paths are rejected.
- **Avoid blocked substrings in the host path.** Defaults include `credentials`,
  `.env`, `.secret`, `.ssh`, `.aws`, `.gcloud`, `.docker`, `.config/nanoclaw`,
  `.local/bin`. A folder named e.g. `my-credentials` will be refused.

---

## 4. Verify mounts on the live container, not with `docker run`

A hand-rolled `docker run -v ...` test bypasses NanoClaw's mount composition
entirely, so it will happily report success on a mount the agent cannot actually
use. It also runs as a different uid than the real spawn.

Check the container NanoClaw actually started:

```bash
docker inspect <container> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} [RW={{.RW}}]
{{end}}'
```

`RW=true` is the only proof that a writable mount is writable. Get the container
name from `docker ps` after sending the agent a message — containers only exist
while an agent is working.

---

## 5. Restart vs rebuild

| What changed | What it needs |
|---|---|
| Persona, memory, skill files | Nothing — read fresh at next spawn |
| Mounts | `ncl groups restart --id <id>` |
| apt / npm packages | `ncl groups restart --id <id> --rebuild` |

Caveat that wastes time: if a container is **already running**, it will not pick up
edited persona or skill files until it is reaped (30 minutes idle) or restarted. So
during setup, restart after edits regardless.

---

## Useful paths

| Inside the container | On the host |
|---|---|
| `/workspace/agent` | `nanoclaw-v2/groups/<folder>/` — RW, persists |
| `/workspace/extra/<name>` | whatever you mounted |
| `/app/skills/<name>` | `nanoclaw-v2/container/skills/<name>/` — read-only |

Put deliverables under `/workspace/agent/`. It persists across container restarts
and needs no mount, since the group folder is already bind-mounted.

Skills are auto-discovered from `container/skills/` when the agent's config has
`skills: "all"` (the default), so a new skill needs no registration step.
