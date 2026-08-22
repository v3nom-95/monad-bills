# monad-bill

A billing app built on [Lago](https://github.com/getlago/lago) with Supabase auth.

Nothing beyond the local Lago bring-up exists yet — this repo currently tracks
the team's agent skill set and this README.

## What's tracked here

| Path | What it is |
|---|---|
| `.agents/skills/` | The [monskills](https://github.com/therealharpaljadeja/monskills) set — source of truth for the agent skills. |
| `skills-lock.json` | Provenance/lock file for the above. |

`.claude/` and `.opencode/` are per-backend copies of `.agents/skills/` and are
not tracked. `lago/` is not tracked either — see below.

## Running Lago locally

`lago/` is a clone of the upstream project, deliberately left untracked: it
carries its own `.git`, its own submodules, and a `.env` with live secrets.
Recreate it from scratch:

```bash
git clone --recurse-submodules https://github.com/getlago/lago.git lago
cd lago
```

If the submodule clone fails with `Host key verification failed`, it's because
`.gitmodules` uses `git@github.com:` SSH URLs. Override them locally:

```bash
git config submodule.api.url https://github.com/getlago/lago-api.git
git config submodule.front.url https://github.com/getlago/lago-front.git
git submodule update --init --depth 1
```

The submodules are only needed for source work — the top-level
`docker-compose.yml` runs prebuilt `getlago/*` images.

Generate the secrets Lago needs into `lago/.env` (never commit this file):

```bash
echo "LAGO_RSA_PRIVATE_KEY=\"$(openssl genrsa 2048 | openssl base64 -A)\"" >> .env
echo "SECRET_KEY_BASE=$(openssl rand -hex 64)" >> .env
echo "LAGO_ENCRYPTION_PRIMARY_KEY=$(openssl rand -hex 32)" >> .env
echo "LAGO_ENCRYPTION_DETERMINISTIC_KEY=$(openssl rand -hex 32)" >> .env
echo "LAGO_ENCRYPTION_KEY_DERIVATION_SALT=$(openssl rand -hex 32)" >> .env
```

Then bring the stack up:

```bash
docker compose up -d
```

The UI is served on <http://localhost/> and the API on <http://localhost:3000>.
Sign up on the login page to create the first user and organization.

If those ports are already taken, the compose file exposes overrides — add them
to `lago/.env` before starting. `LAGO_API_URL` must match whatever the browser
will actually reach:

```
API_PORT=3001
POSTGRES_PORT=5433
REDIS_PORT=6381
LAGO_API_URL=http://localhost:3001
```

Stop with `docker compose down`, or `docker compose down -v` to also wipe the
data volumes.
