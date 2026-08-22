# monad-bill — handoff

Working directory: `D:\monad-bill` (Windows, PowerShell). Repo is public:
<https://github.com/v3nom-95/monad-bills>. **Never commit a credential value
here — variable names only.**

Architecture decision behind all of this: [ADR-0001](adr/0001-auth-architecture.md).

## How to run

### Lago (billing engine)

From `D:\monad-bill\lago`:

```powershell
docker compose up -d
```

- Front (admin console): <http://localhost/>
- API: <http://localhost:3001> — currently healthy, v1.51.0

Ports were remapped because 3000, 5432 and 6379 were already occupied on this
machine. Set in `lago/.env`:

| Service    | Port |
| ---------- | ---- |
| Lago API   | 3001 |
| Postgres   | 5433 |
| Redis      | 6381 |
| Lago front | 80   |

monad-bill itself will run on **3002**.

### Recreating `lago/`

`lago/` is intentionally **untracked** (it is listed in `.gitignore`): it is a
third-party clone with its own `.git`, its own submodules, and a `.env` holding
live secrets. To recreate:

```powershell
git clone --recurse-submodules https://github.com/getlago/lago.git
```

Two notes:

- `lago/.gitmodules` uses SSH URLs (`git@github.com:getlago/...`). On a machine
  with no SSH key, add an HTTPS override in the clone's local `.git/config`.
- The top-level `docker-compose.yml` runs **prebuilt images**, so the submodules
  are not needed just to run it. They are only needed to read/modify the source.

## What's done

- Lago v1.51.0 running locally on prebuilt images, API healthy.
- Auth architecture decided and recorded (ADR-0001).
- Supabase project configured for email auth.
- `app/` — Next.js scaffold in progress (owned by Developer; not documented here
  yet, it would be stale by tomorrow).

## What's open

- **Blocker: `LAGO_API_KEY` does not exist yet.** Lago has zero users, zero
  organizations and zero API keys (verified directly against `lago-db`). The key
  cannot exist until someone signs up at <http://localhost/> and creates the first
  organization. Nothing that calls Lago's REST API can work before that.
- **`LAGO_DISABLE_SIGNUP` is unset**, so Lago's admin signup is currently open.
  This is deliberate: setting it while there are zero users would lock the admin
  UI permanently. Set it immediately after the first account exists.
- **Supabase redirect allow-list.** The email-confirmation redirect URL must be
  allow-listed in the Supabase dashboard. This is a manual dashboard step; it
  cannot be done from code.

## Known gotchas

- **Supabase: email is the only enabled provider.** No OAuth, phone, passkeys or
  anonymous sign-in. `mailer_autoconfirm: false`, so signup returns **no session**
  until the user confirms by email — a user row exists before confirmation. This
  is exactly why Lago customer provisioning is lazy (ADR-0001, rule 2).
- **The Lago API key is organization-wide, read+write.** It must never reach a
  browser. All Lago calls go through our server tier.
- **Lago's admin JWT has no refresh token.** Renewal is sliding, via the
  `x-lago-token` response header. Irrelevant to our app (we don't use that path),
  but it surprises people reading the Lago source.
- **monskills install did not populate `.opencode\skills\`.** It was copied in
  manually, and agents needed a context reset to pick it up. On Windows these
  landed as **real directory copies, not symlinks**, so `.opencode\skills\` will
  go stale silently if the skills are ever updated — `.agents/skills/` is the
  tracked source of truth, and re-syncing is the lead's job. Nobody else should
  run the installer.
- **The monskills router advertises a `wallet/` skill that is not in the
  published package** (Alchemy Agent Wallet sessions, CREATE2/CreateX deploys).
  If a task needs it, flag it — do not improvise from training memory.
  `wallet-integration` (Para) is a different thing and is not a substitute.
