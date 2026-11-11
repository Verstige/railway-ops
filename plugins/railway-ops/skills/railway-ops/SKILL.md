---
name: railway-ops
description: "Railway multi-service operations: managing separate services within one project, per-service environment variables, service identification, restart/redeploy patterns, and cross-service wiring (when one service needs to call another). Use when working with Railway projects that have multiple services (web, telegram-listener, api, bot, etc.) \u2014 especially when adding a new env var to one service and needing to check/set it on others."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# Railway Operations — Multi-Service Patterns

Railway projects with separate services (web frontend, telegram-listener bot,
the broker-api, etc.) require per-service management. Each service has
isolated environment variables — a variable set on `web` does NOT exist
on `telegram-listener`.

## List all services

```bash
railway service status --all
# Output:
# Services in production:
# telegram-listener    | 2ff5ff76-35d8-4aea-bd8f-92b01dbbf169 | SUCCESS
# web                  | 2e4167bd-a044-4870-a0ac-0c1dbfabf0f0 | SUCCESS
# the broker-api      | 9bcea051-1fa9-4d1f-8eaa-6d5f21b7da82 | SUCCESS
```

## Check / set per-service environment variables

```bash
# List
railway variables --service <name>

# Set — use the EXPLICIT subcommand with positional KEY=VALUE
railway variable set KEY=value --service <name>
# (NOT `railway variables set ...` — that form silently no-ops, exits 0, doesn't change anything)

# Filter
railway variables --service web | grep -i "vapid\|push"
railway variables --service telegram-listener | grep -i "push\|app_url"
```

**⚠️ The `railway variable set` vs `railway variables set` trap (Jul 31 2026, silent no-op).** The top-level `railway variables` command has a legacy `--set KEY=VALUE` flag that returns exit 0 but doesn't actually set the variable. **Always use the explicit subcommand: `railway variable set KEY=VALUE --service NAME`** (singular `variable`, not `variables`). If you ever see `railway variables --set X=Y` produce no error AND no change, that's the bug — the CLI's flag is a no-op. The CLI also doesn't echo errors for many operations, so always verify with a second `railway variables --service NAME | grep KEY` after setting. The `set` subcommand is documented under `railway variable --help` (singular).

## Restart to pick up new env vars

Railway only loads new variables on restart/redeploy.

```bash
railway service restart -s <name> --yes
railway service redeploy -s <name>
```
If `redeploy` fails ("cannot redeploy, service is building"), the service is mid-deploy — wait and retry.

**⚠️ `railway service restart` times out in non-interactive mode (Jul 29, 2026).**
When running from an automated script or background process without a real TTY, `restart` and `redeploy` both time out after ~30s and return non-zero exit — even when the command would eventually succeed. The Railway CLI requires an interactive session to poll for completion.

**Workaround:** Set env vars via `railway variables set` (works fine non-interactively), then trigger restart via the Railway dashboard: Project → Service → Settings → Restart. Or push a new commit to `main` to trigger Railway's automatic redeploy.

**Always verify env vars are present** before assuming the service has picked them up:
```bash
railway variables --service <name> | grep -i "KEY_NAME"
```
Empty output = not set on that service.
If `redeploy` fails ("cannot redeploy, service is building"), the service is mid-deploy — wait and retry.

## Deploy to a specific service

```bash
railway up --service web
```

**⚠️ `railway redeploy` and `railway up` operate on the LINKED service, silently defaulting to whichever service the CLI last linked (Sep 6 2026, hw888 / the target platform)**

When a Railway project has multiple services (e.g. `web`, `telegram-listener`, `the broker-api`), `railway redeploy --yes` and `railway up` (with no `--service` flag) silently operate on whichever service your CLI is currently linked to. **If you assume "I just deployed to web" because web is what you've been working on, but your CLI link points to `telegram-listener` from a previous session, you'll rebuild telegram-listener and your web chunk hashes will not change.** The deploy appears to succeed (the new build image digest lands), but the chunks the browser actually loads come from a stale build.

**Real session (Sep 6 2026, the target platform):** I issued `railway redeploy --yes` seven times in a row, watching the deploy list update with new image digests. None of them included the source changes I'd made. After 30 minutes of debugging I noticed the deployed `commitHash` field was always `?` (empty) — the hallmark of `railway up` (which uploads a local tarball) rather than `git push` (which has a commit hash). The CLI was silently deploying to `telegram-listener` (the first service in `railway status --json`) instead of `web` (the service I thought I was deploying).

**The two tell-tale signs of a misrouted deploy:**
1. The latest deploy's `meta.commitHash` is `?` (no commit metadata) instead of the SHA you just pushed. `railway up` zips the working tree and uploads directly → no git context → commitHash is empty. `git push`-triggered deploys always carry the commit SHA.
2. The latest deploy's `meta.imageDigest` keeps changing even though you didn't change source — proving Railway is rebuilding something, but it's not what you thought.

**Always pass `--service <name>` explicitly AND verify the target before the deploy:**

```bash
# 1. Confirm which service is linked
railway status --json | python3 -c "import sys,json; d=json.load(sys.stdin); print('env:', d['environments']['edges'][0]['node']['name']); [print(' service:', s['node']['serviceName'], s['node']['id']) for s in d['environments']['edges'][0]['node']['serviceInstances']['edges']]"

# 2. Verify the latest deploy for THAT service
railway deployment list --service web --limit 1 --json | python3 -c "import sys,json; d=json.load(sys.stdin); print('commit:', d[0].get('meta',{}).get('commitHash','?')[:12], 'image:', d[0].get('meta',{}).get('imageDigest','?')[:20])"

# 3. THEN redeploy with explicit service flag
railway redeploy --service web --yes
# OR for a fresh build:
railway up --service web --ci --detach

# 4. Re-verify the new deploy landed on the right service + the right commit
railway deployment list --service web --limit 1 --json | python3 -c "import sys,json; d=json.load(sys.stdin); print('commit:', d[0].get('meta',{}).get('commitHash','?')[:12], 'image:', d[0].get('meta',{}).get('imageDigest','?')[:20])"

# 5. Verify the BROWSER is loading the new chunks (not the cached ones)
SAMPLE=$(curl -s --max-time 15 https://www.<domain>/dashboard/trading \
    -H 'User-Agent: Mozilla/5.0' \
    | grep -oE '/_next/static/chunks/[a-z0-9_-]+\.js' | sort -u | head -1)
curl -s --max-time 10 "https://www.<domain>$SAMPLE" | grep -c '<YOUR_NEW_STRING_MARKER>'
```

If step 5 returns 0 but step 4 shows the new image, the build DID complete but the CDN is still serving the old chunks. That's a separate issue — see `references/stale-static-assets-after-deploy.md` for the empty-commit workaround.

**Pair this with the `railway up --service web` deployment pattern:** always specify `--service`, never rely on the implicit link. The link in `~/.railway/config.json` is per-directory AND per-service — it persists across sessions and can drift.

### ⚠️ `railway up` deploys to the CLI's linked service, not the directory you're in (Aug 13 2026, hw888)

The CLI keeps a per-directory service link in `~/.railway/config.json`. If that link is stale from a previous session, `railway up` will deploy to the wrong service — silently, no warning. **Example failure:** CLI was linked to Postgres; a subsequent `railway up` from an application worktree deployed the Next.js build to Postgres, overwriting the database. The diagnostics all pointed at "Prisma can't reach the DB" because the hostname was now resolving to a Next.js container, not a Postgres daemon.

**Always:** pass `--service <name>` explicitly AND `railway status --json | jq -r '.service'` before each deploy. The auto-link is a foot-gun in multi-service projects.

**If you discover you overwrote a Postgres/Redis service:** the `serviceManifest.build.buildCommand` on the corrupted service's latest deployment will be the web app's build command (e.g. `npx prisma generate && npm run build`), not a Postgres image ops string. Recovery: `serviceDelete` the corrupted service + `templateDeploy` a fresh Postgres + re-link the CLI to the web service before re-deploying. See `railway-graphql-api` § "railway up deploys to the link destination" for the full recovery recipe.

## Cross-service wiring pattern

When `telegram-listener` needs to call `web`'s `/api/signals/notify`, it needs:
- `PUSH_API_TOKEN` (the same token that `web` uses — set on telegram-listener separately)
- `APP_URL` (set to `https://www.the-org.example` — not the listener's own URL)

Each service is isolated. Set both on `telegram-listener` for it to POST to `web`'s endpoints.

## Service identification

- From CLI: `railway service status --all`
- From `.env.railway` in a service directory: `RAILWAY_SERVICE_NAME`, `RAILWAY_PUBLIC_DOMAIN`
- The `.env.railway` file in `web`'s environment does NOT reflect `telegram-listener`'s environment — each service must be queried separately

## Restart policy for long-running listeners

Set `restartPolicyType = "ALWAYS"` in Railway service settings for services running long-poll loops or webhook listeners (e.g., telegram-listener). Default is `ON_FAILURE`, which won't restart a process that's alive but hung on a stuck connection.

## Railway Cron Jobs

Railway supports native cron triggers. Services configured as cron jobs **must exit cleanly** after their task — Railway skips the next run if the previous one is still active.

### Cron Configuration

**`*/5 * * * *`** — every 5 minutes (Railway's minimum interval)

Set cron via the Railway dashboard:
1. Select the service → Settings → Cron Schedule
2. Enter a [crontab expression](https://docs.railway.com/cron-jobs#crontab-expressions)

Or via `railway.toml` (per-service, in the service subdirectory):

```toml
[service]
  name = "genesis-sweep"
  source = "."

[cron]
  schedule = "*/5 * * * *"
```

> Note: The `[cron]` block in railway.toml is recognized by Railway. If auto-application doesn't take effect, set it manually in the dashboard.

> **Aug 17 2026 update:** new pitfall added — "Build hangs in the Railway queue" — symptom, diagnosis, and the empty-commit re-trigger recipe. Search `Build hangs` to find the section.

> **Aug 17 2026 update:** new pitfall added — "Build hangs in the Railway queue" — symptom, diagnosis, and the empty-commit re-trigger recipe. Search `Build hangs` to find the section.

### Key constraints

- Minimum interval: **5 minutes** — Railway will not accept shorter intervals
- Cron schedule is in **UTC**
- Service must **exit with code 0** when done — no open DB connections, no hanging processes
- If a previous run is still `Active`, Railway **skips the next scheduled run**

### Cron expression reference

| Expression | Meaning |
|---|---|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 6 * * *` | Daily at 06:00 UTC |
| `0 8 * * 1-5` | Weekdays at 08:00 UTC |
| `30 14 * 1 0` | Sundays in January at 14:30 UTC |

## Adding a New Railway Service (Discord bot, cron worker, etc.)

When adding a new service (e.g., a Discord role sync worker):

### Step 1 — Create the service via CLI or dashboard

```bash
# Via Railway CLI
railway service add
# Follow prompts to name the service and select the environment

# Or via Railway dashboard: Project → Add Service → Empty Service
```

### Step 2 — Configure the service

In the Railway dashboard for the new service:
- **Build command:** `npm run build` (for Node services) or leave blank if using Dockerfile
- **Start command:** e.g. `node services/discord-role-sync/index.mjs`
- **Dockerfile path:** if using a custom Dockerfile, point to `services/discord-role-sync/Dockerfile`
- **Root directory:** set to the repo root (so the Dockerfile can reference scripts at the repo root)

### Step 3 — Set environment variables on the new service

```bash
railway variables --service <new-service-name>
```

**Critical: Environment variables are NOT shared between Railway services.** A variable set on `web` or `telegram-listener` does NOT exist on a new service — even within the same Railway project. You must explicitly set all required vars on each service.

Common vars that need explicit setting on new services:
```
DISCORD_BOT_TOKEN=        # copy from Discord Developer Portal
DISCORD_GUILD_ID=         # copy from Discord Developer Portal
DISCORD_ROLE_ID_<RANK>=   # one per rank: DISCORD_ROLE_ID_PRESTIGE=123456
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_SUPABASE_URL=
```

### Step 4 — Add cron trigger (if applicable)

In the new service's Railway dashboard → Settings → Cron Schedule:
```
0 6 * * *   # Daily at 06:00 UTC
```

### Step 5 — Redeploy to pick up new service configuration

```bash
railway service redeploy -s <new-service-name>
```

### Discord Bot Token Pattern

**Discord bot tokens are per-bot, not per-service.** The same `DISCORD_BOT_TOKEN` can be set on multiple Railway services (e.g., telegram-listener + discord-role-sync) that need to use the same Discord bot. However, **each service must have the variable explicitly set** — Railway does not auto-share variables between services.

To get a bot token:
1. Go to https://discord.com/developers/applications
2. Select the application → Bot → Reset Token (copy the new token)
3. Add the bot to your server with `SERVER MEMBERS INTENT` and `MANAGE ROLES` permissions

The bot needs these **Privileged Gateway Intents** (Discord Developer Portal → Bot page):
- **SERVER MEMBERS INTENT** — required for `GET /guilds/{guild_id}/members/search`
- **PRESENCE INTENT** — optional, for checking online status

## Railway API Token

Railway CLI tokens are stored in `~/.railway/config.json` under `user.token`. If API calls return `403 Forbidden`, the token may be expired — re-authenticate with `railway login`.

The Railway GraphQL API base: `https://backboard.railway.app/api/graphql`
The Railway REST API base: `https://backboard.railway.app/api/v2`

## Common the target platform service names

| Service | Purpose |
|---|---|
| `web` | Next.js frontend — **linked to `the target platform` Railway project, NOT `the org-os-frontend`** |
| `telegram-listener` | Telegram signal ingestion bot |
| `the broker-api` | the broker API proxy |
| `genesis-sweep` | Genesis signals sweep + attribution cron |
| `postgres` | Managed PostgreSQL |

### ⚠️ VAPID Key Triple-Set Rule (Aug 3 2026)

When setting VAPID keys on Railway, set ALL THREE variables on the SAME service at the SAME time:

```
VAPID_PUBLIC_KEY= BJnls0RN...   (public half — browser uses NEXT_PUBLIC_ prefix)
VAPID_PRIVATE_KEY= FIBUzgGL...  (private half — server-side web-push signing ONLY)
VAPID_SUBJECT= mailto:<admin-email>
NEXT_PUBLIC_VAPID_PUBLIC_KEY= BJnls0RN...   (SAME as VAPID_PUBLIC_KEY)
```

**`NEXT_PUBLIC_VAPID_PUBLIC_KEY` and `VAPID_PUBLIC_KEY` must be identical.** The browser code reads `NEXT_PUBLIC_VAPID_PUBLIC_KEY` (exposed via `NEXT_PUBLIC_` prefix). The server reads `VAPID_PRIVATE_KEY`. A subscription created with `NEXT_PUBLIC_VAPID_PUBLIC_KEY = key-A` can ONLY be authenticated with `VAPID_PRIVATE_KEY = private-key-A`. If they come from different pairs, all web-push sends silently fail.

Generate a fresh matching pair with `npx web-push generate-vapid-keys`, set all three on Railway, redeploy, delete all `push_subscriptions` rows, have users re-subscribe.

### ⚠️ Railway Project Identity Trap (Aug 3, 2026 — wasted 20+ min)

There are TWO similarly-named Railway projects for the org:
- **`the target platform`** — the production app. Hosts `www.the-org.example` → `example-service.up.railway.app`. Service is `web`. **This is the real production project.**
- **`the org-os-frontend`** — a different/dormant project. Service is `the org-os-frontend`. Returns `{"error":"Application not found"}` on API routes. **Wrong project.**

**Never run `railway link` without specifying project and service explicitly for the target platform:**

```bash
# WRONG — prompts interactive selector, easy to pick the wrong project
railway link

# RIGHT — explicit, unambiguous
railway link --project "the target platform" --environment "production"
railway service "web"   # service name, NOT project name

# Verify after linking
railway status
# Expected: Project: the target platform, Service: web, Domain: https://www.the-org.example
```

**Finding the correct deployment URL when you only have the project name:**
```bash
# List all projects and find the target platform
railway list | grep -i the org
# The correct project shows TWO entries: the target platform + the org-os-frontend

# Check domain on the correct project after linking
railway domain
# Expected: https://www.the-org.example  +  https://web-production-XXXX.up.railway.app

# If domain shows no custom domain AND the API returns 404 "Application not found",
# you are on the org-os-frontend — re-link with the right project name
```

**The `the target platform` project domains (confirmed Aug 3, 2026):**
```
https://www.the-org.example                      ← CNAME target
https://example-service.up.railway.app ← Railway-provided URL
```

### ⚠️ "Application not found" from a Railway hostname means the app isn't bound to it (generalized)

When a Railway-hosted hostname answers with a JSON body shaped like:
```json
{"status":"error","code":404,"message":"Application not found","request_id":"UTuIjcEQS6S8DAxBqmzx2A"}
```
the request **never reached your container** — Railway's edge refused it because no live app is bound to that hostname. **This is not a route bug, not a middleware bug, and not a crashed deployment.** Don't debug the Next.js app; fix the hostname↔service binding.

**Why it matters:** a stale custom domain and a live service coexist all the time. The CNAME still points at a Railway app that was renamed or removed while the service lives on under its Railway-provided domain — and the reverse also holds (Railway-provided domain retired, custom domain still resolving). A hardcoded live URL in a skill, memory entry, README, or deploy script is the fastest thing to go stale.

**Resolution procedure (run before touching code):**
1. `railway list | grep -i <project>` — find the project that actually owns the service.
2. `railway link --project "<name>" --environment "<env>"` then `railway service "<service>"` — link explicitly; never rely on the last link (see the misrouted-deploy section above).
3. `railway status --json` — read the service's current domain and status.
4. `railway domain` — print both the custom domain and the Railway-provided domain.
5. Compare the returned domain against the URL you were curling. Different → you were curling a retired hostname.
6. If **both** the custom and Railway-provided domains return "Application not found", the service itself is gone or renamed — list services with `railway service status --all` and pick the right one.

**Do not conclude the app route is broken from a bare 404.** Check the response body shape first: Railway-edge refusals carry `request_id` and the `Application not found` message. A real app-level 404 carries your app's own 404 shape (HTML page, or your JSON error body).

**Railway `variable set` command form — singular, not plural:**
```bash
# WRONG — silent no-op, exits 0, variable is NOT set
railway variables set KEY=value
railway variables set KEY=value --service web

# RIGHT — explicit subcommand
railway variable set KEY=value --service web
```

Always verify after setting:
```bash
railway variables --service web | grep -i "KEY_NAME"
# Empty output = not set, re-run with correct syntax
```

## Static assets stale after deploy (Next.js + Turbopack)

If the served HTML references chunk hashes that 404, or a pushed code change isn't visible after Railway reports `SUCCESS`, the static asset handler is holding old artifacts. Diagnosis: locally `ls .next/static/chunks/` for new hashes, then `curl` the served HTML and the new chunk URL — they'll be out of sync. Fix: `git commit --allow-empty -m "chore: force redeploy to pick up new chunks" && git push origin main` forces a fresh container; `railway redeploy --yes` and `railway service restart` are less reliable. See `references/stale-static-assets-after-deploy.md` for full diagnosis + prevention.

## Build hangs in the Railway queue (Aug 17 2026 — observed)

**Symptom:** `railway deployment list` shows the new deploy as `BUILDING` for 10+ minutes, but `railway logs <id> -b` shows only one line: `scheduling build on Metal builder "builder-XXXX"`. No `npm install`, no `npm run build`, no error. The previous successful deploy had full build logs (Nixpacks phases, `npm install`, `npm run build`). This is a build **queue hang**, not a build failure.

**Diagnosis:** the build is queued on a Metal builder but the worker never picked it up. Common during periods when the same project has been deploying frequently or when Railway's builder pool is overloaded. Not a code problem — the same commit builds clean locally.

**Recipe:**

```bash
# 1. First verify the code is actually fine (rules out a real code bug)
cd ~/Projects/<project> && npx tsc --noEmit && npx next build

# 2. If the build is clean locally, retrigger via empty commit
git commit --allow-empty -m "chore: retrigger deploy after builder queue hang"
git push origin main

# 3. Monitor the next attempt — does it land?
railway deployment list
# The new id should go BUILDING → SUCCESS within 2-3 minutes
```

**Anti-patterns (don't do these):**

- **`railway deployment redeploy` on the failed deployment.** The CLI refuses with "currently building, deploying, or was removed" once a deploy has been FAILED for more than a few minutes. The empty-commit path works around it by giving Railway a fresh deployment.
- **`railway up` to bypass the git pipeline.** Violates the "git push triggers deploy" workflow. The empty commit reuses the same webhook that all your other deploys use.
- **Assume the code is broken and start debugging.** The deployment is queued, not failed. The build logs are *empty* because the build hasn't started — there is no failure to diagnose.

**If the second attempt also hangs in the same way:** contact Railway support. The Metal builder pool itself is exhausted, not just this project. No code change will fix it.

## Cleaning Vercel artifacts when migrating to Railway-only deploy (Jul 31 2026)

When a project moves from Vercel-as-deploy to Railway-as-deploy, the local repo accumulates Vercel CLI artifacts that need cleaning:

| File | Created by | What to do |
|---|---|---|
| `.vercel/` | `vercel` CLI | Delete the directory; add `.vercel` to `.gitignore` if not already there |
| `.env.prod` | `vercel env pull` | Replace with a one-line deprecation stub OR delete; the Vercel-specific vars (`VERCEL_*`, `VERCEL_OIDC_TOKEN`) are useless on Railway |
| `.env.pull` | `vercel env pull` | Same — delete or deprecation-stub |
| `.env.development.local` | Vercel CLI | Same |
| `.env.local` | Vercel CLI or manual | KEEP — but strip `VERCEL_OIDC_TOKEN`; keep the actual app keys (Supabase, Gemini, etc.) |
| `the org.md` / `AUDIT.md` / similar docs | manual | Search-replace any `the org-os-official.vercel.app` URLs to the Railway domain (e.g. `www.the-org.example`) |
| `e2e/*.spec.ts` | Playwright | Search-replace the `BASE_URL` default from the Vercel URL to the Railway domain |

**Verification before committing the cleanup:**

```bash
# Should be zero matches after cleanup
grep -rln "vercel\|VERCEL_" . \
  --exclude-dir=node_modules \
  --exclude-dir=.next \
  --exclude-dir=.git \
  --exclude="package-lock.json"
```

Ignore matches in:
- `package-lock.json` (npm packages with "vercel" in their name — e.g. `node-fetch` internals)
- `tsconfig.tsbuildinfo` (TypeScript build cache, regenerated)
- `docs/research/*.html` (research artifacts, not code)
- The four `.env*` files if you replaced them with deprecation stubs that mention "Vercel" in a "this is no longer used" context

**Critical security check — search git history for committed secrets:**

```bash
# Has the Supabase anon key OR a Vercel OIDC token ever been committed?
git log --all -S "VERCEL_OIDC_TOKEN" --oneline
git log --all -S "NEXT_PUBLIC_SUPABASE_ANON_KEY" -- '.env.prod' --oneline
git log --all --full-history --oneline -- .env.prod
```

If secrets appear in any public-repo commit, they need to be:
1. **Rotated** in the source (Supabase dashboard, Vercel dashboard)
2. **Scrubbed from history** via `git filter-repo` or BFG Repo-Cleaner

A future commit that removes them is not enough — they're still visible to anyone with repo access at the old commit hash.
