---
layout: post
title: "Leaving Railway for self-hosted Coolify, with Claude Code doing the moving"
date: 2026-10-22
categories: [Self-hosting]
tags: [railway, coolify, claude-code, cloudflare, migration]
---

## Why I'm leaving

On October 21, 2026, Railway disclosed a security incident: an unauthorized party may have accessed encrypted customer environment variables and, for some of them, obtained the decrypted contents. Railway paused builds across the platform, invalidated all API tokens, rotated bucket credentials, and told every customer to rotate everything stored in Railway.

- Status page: [Railway incident QH8L8B3E](https://status.railway.com/incident/QH8L8B3E)
- Disclosure: [Security Disclosure: Wednesday October 21st](https://blog.railway.com/p/security-disclosure-wednesday-october-21st)

For me, "everything" meant bank-sync access tokens for my budgeting app, Telegram bot tokens, database passwords, JWT signing keys, and payment and AI provider keys. I rotated all of it the same day.

Railway was cheap. Five small apps and five Postgres databases fit under the $5/month Hobby plan. Cost wasn't the problem. Where my secrets live was. I want them on hardware I control.

## Options I looked at

My five apps average about 1.3 GB of RAM in total and almost no CPU. They sleep most of the day.

| Option | Cost / month | Catch |
|---|---|---|
| Railway (stay, rotate) | ~$5 | Same platform that just leaked secrets |
| Cloudflare Containers | $5 minimum + usage, ~$10–25 for me | Requires Workers Paid. Memory is billed at the full instance size while awake, not actual usage. No managed Postgres. Container disk is wiped on restart, so volumes need rework. |
| Google Cloud | $0 only on paper | The free tier holds only if every app scales to zero. My apps are .NET: cold starts take seconds, and Cloud Run throttles CPU after the response is sent, so in-process background jobs and queues stall. Keeping a .NET service warm with always-on CPU switches it to instance-based billing, which alone costs more than my whole Railway bill. Cloud SQL has no free tier either. |
| **Own Ubuntu server + Coolify + Cloudflare Tunnel** | **~$0 extra** | You patch the OS. Home power and ISP outages take sites down. |
| Rented VPS + Coolify | ~€6–10 | Same setup on someone else's hardware. See below. |

I picked the last one. [Coolify](https://coolify.io) gives you the Railway workflow on your own box: push to `main` deploys, env vars live in a UI, one-click Postgres, and scheduled database backups to S3-compatible storage (I use Cloudflare R2). Cloudflare Tunnel publishes the sites without opening any ports on my router.

## No home server? Rent a VPS

Everything below works the same on a rented VPS. Coolify's documented minimum is 2 CPU cores, 2 GB RAM and 10 GB of disk. Coolify itself takes 0.5–1 GB, so for a handful of small apps plus their databases get **4 GB of RAM** and 40 GB or more of disk.

| Provider | Plan (approx.) | Specs | Price / month |
|---|---|---|---|
| Hetzner Cloud | Cost-Optimized (CX23) | 2 vCPU, 4 GB, 40 GB | from €5.99 + VAT |
| Contabo | Entry Cloud VPS | ~3 vCPU, 8 GB | ~€5–6 (cheapest on longer terms); slower disk and support |
| Netcup | RS 1000 | dedicated cores, 8 GB | ~€9 |
| DigitalOcean | Basic Droplet | 2 vCPU, 4 GB | $24 |
| Oracle Cloud | Always Free Ampere A1 | up to 4 ARM cores, 24 GB | $0, but capacity is hard to get and idle instances can be reclaimed |

Prices as of October 2026, excluding VAT. They change often, so check the provider's pricing page. For most people, Hetzner's 4 GB plan is the sweet spot: about €6–7 a month all-in, or roughly what Railway Hobby plus a little usage costs.

VPS-specific notes:

- You have a public IP, so you can skip Cloudflare Tunnel and let Coolify issue Let's Encrypt certificates on ports 80/443 ("direct mode" in the prompt). The Tunnel still works if you'd rather keep every port closed.
- Firewall everything except 22, 80 and 443. Don't expose Coolify's dashboard on port 8000: reach it through an SSH tunnel (`ssh -L 8000:localhost:8000 you@server`) or the provider's firewall allow-list.
- Provider snapshots (usually ~20% of the server price) are a nice extra, but keep the off-site database backups to R2/S3 anyway.

## Rotate first, migrate second

Moving hosts doesn't un-leak anything. Before anything else, do what Railway's guidance says:

- Rotate every environment variable you stored in Railway: API keys, database credentials (Postgres, MySQL, MongoDB, Redis), private registry credentials, webhook secrets, JWT keys.
- Create new Railway API tokens. All old ones were invalidated.
- Review usage on connected services (AI providers, Stripe, GitHub, cloud accounts) for anything you didn't do.
- Railway reset the GitHub tokens it held. If you're leaving, also revoke the Railway GitHub App's access to your repos once you're done.

## How the migration works

Claude Code runs **on the target server** and does the moving: it inventories your Railway account, installs Coolify, recreates each app and database, copies env vars, DB data and volumes, and switches DNS through a Cloudflare Tunnel. Your part is to create a few tokens, click through Coolify's first-run setup, and approve downtime. Below is everything you need: what to prepare, two small scripts, and the prompt to paste.

## What you need

- **A Linux server** (Ubuntu 22.04/24.04) with ≥4 GB RAM, ≥20 GB free disk, `sudo`. Home server or VPS both work.
- **Claude Code installed on that server.** It does the work there, not on your laptop.
- **A Railway account token**: railway.com → Account Settings → Tokens. Railway invalidated all old tokens on October 21, so create a fresh one just for this and revoke it afterwards.
- **Your domains on Cloudflare DNS** *if* you want Cloudflare Tunnel (recommended for home servers: no open ports, free TLS). Without Cloudflare, you'll need ports 80/443 open and you'll update DNS records yourself.
- **Cloudflare API token** (tunnel mode only), with exactly:
  - Zone › DNS: Edit — only the zones you're migrating
  - Zone › Zone: Read — same zones
  - Account › Cloudflare Tunnel: Edit
- **Your Cloudflare account ID**: 32 hex chars, no dashes. Dashboard → right sidebar.
- **Optional, for automated DB backups**: an S3-compatible bucket (Cloudflare R2, Backblaze B2, AWS S3) plus a key scoped to that bucket only. For R2 the endpoint is `https://<account-id>.r2.cloudflarestorage.com`.

## Step 1: create the secrets file (on the server)

Values are typed hidden. Nothing lands on screen or in your shell history. Leave the optional ones empty if you don't need them.

```bash
mkdir -p ~/migration && chmod 700 ~/migration && umask 077 && : > ~/migration/secrets.env
ask(){ read -rsp "$1 ($2): " v; echo; printf '%s=%s\n' "$1" "$v" >> ~/migration/secrets.env; }
ask RAILWAY_API_TOKEN  required
ask CF_API_TOKEN       "tunnel mode only, else empty"
ask CF_ACCOUNT_ID      "tunnel mode only, else empty"
ask BACKUP_S3_ENDPOINT "optional"
ask BACKUP_S3_BUCKET   "optional"
ask BACKUP_S3_KEY_ID   "optional"
ask BACKUP_S3_SECRET   "optional"
echo "COOLIFY_API_TOKEN=" >> ~/migration/secrets.env   # fill in after Coolify is installed
chmod 600 ~/migration/secrets.env && cut -d= -f1 ~/migration/secrets.env
```

## Step 2: check it (run again after adding the Coolify token)

Prints names and status only, never values.

```bash
f=~/migration/secrets.env; set -a; . "$f"; set +a; bad=0
chk(){ [ -n "${!1}" ] && echo "OK      $1" || { echo "MISSING $1"; bad=1; }; }
chk RAILWAY_API_TOKEN; chk COOLIFY_API_TOKEN
if [ -n "$CF_API_TOKEN$CF_ACCOUNT_ID" ]; then
  chk CF_API_TOKEN; chk CF_ACCOUNT_ID
  [[ "$CF_ACCOUNT_ID" =~ ^[0-9a-f]{32}$ ]] || { echo "BAD     CF_ACCOUNT_ID (32 hex, no dashes)"; bad=1; }
else echo "SKIP    Cloudflare (no tunnel mode)"; fi
if [ -n "$BACKUP_S3_ENDPOINT$BACKUP_S3_BUCKET$BACKUP_S3_KEY_ID$BACKUP_S3_SECRET" ]; then
  for v in BACKUP_S3_ENDPOINT BACKUP_S3_BUCKET BACKUP_S3_KEY_ID BACKUP_S3_SECRET; do chk $v; done
else echo "SKIP    backup storage (optional)"; fi
[ "$(stat -c %a "$f")" = 600 ] || { echo "BAD     file mode, run: chmod 600 $f"; bad=1; }
[ $bad = 0 ] && echo "All good." || echo "Fix the above."
```

`COOLIFY_API_TOKEN` showing MISSING is expected until Claude installs Coolify and you create the token.

## Step 3: paste this prompt into Claude Code on the server

```
# Migration: Railway → self-hosted Coolify (this server)

You are Claude Code running on the target Linux server. Move my Railway projects (apps, databases, volumes, env vars, custom domains) to Coolify on this machine with minimal interaction from me. Work phase by phase, verify every step, and keep a running log in ~/migration/LOG.md (done / verified / left).

## Hard rules
1. Never print secret values: not in tool output, the log, or replies. Pipe secrets between tools (jq, curl --data @file). Keep intermediate files in ~/migration/ with chmod 600. If a command would echo a secret, redirect it to a file.
2. Railway is read-only, except: `railway down` on an app service during its cutover (ask me once for blanket approval before Phase 3), temporary TCP proxies you create (delete them afterwards), and removing a custom domain from a Railway service after the new one is verified. Never delete Railway projects, services, volumes or databases. Railway stays as rollback until I delete it.
3. Before changing any DNS record, save the original to ~/migration/dns-backup.json.
4. Ask me only when blocked on something only I can do (browser clicks, approvals, choices). Batch questions.
5. Don't push to any git repo without asking.

## Secrets
~/migration/secrets.env (chmod 600) holds RAILWAY_API_TOKEN, COOLIFY_API_TOKEN (filled after Phase 1), optionally CF_API_TOKEN + CF_ACCOUNT_ID (tunnel mode) and BACKUP_S3_ENDPOINT/BUCKET/KEY_ID/SECRET. Load inside each command with `set -a; . ~/migration/secrets.env; set +a`. Never cat it.

## Phase 0: preflight and inventory (no changes)
1. Report OS, CPU, RAM, free disk, whether Docker is installed, and what listens on ports 80, 443, 8000, 6001, 6002. Need ≥4 GB RAM, ≥20 GB free. If 80/443 are taken by something else, stop and ask.
2. Install if missing: jq, Railway CLI, a PostgreSQL client matching or newer than the newest Railway Postgres major version (PGDG repo), and cloudflared if CF vars are set. Verify the tokens: `railway whoami`, and Cloudflare token verify if set.
3. Build the inventory with the Railway CLI. For every project, environment and service, record: name/ID, source (repo + branch, or image), builder/Dockerfile path, whether it's a database (Postgres/MySQL/Redis/Mongo…), volumes and mount paths, custom domains and target ports, cron schedule, replicas, and variable *names* only. Also pull 7-day CPU/RAM metrics if available.
4. Show me the inventory as a table and propose: what to migrate, what to skip (unused services), the order (least critical first), and whether to use Cloudflare Tunnel (only if CF vars are set and the domains' NS are on Cloudflare) or direct ports 80/443. Wait for my confirmation.

## Phase 1: install Coolify
1. If not installed: `curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash`.
2. Stop and ask me to do this in a browser at http://<server-LAN-IP>:8000 (print the IP):
   - register the admin account
   - Settings → enable API access
   - Keys & Tokens → create a root API token → put it in secrets.env as COOLIFY_API_TOKEN
   - Sources → add a GitHub App and install it on the repos being migrated (this gives auto-deploy on push)
3. Use a Coolify MCP server if one is connected; otherwise the REST API (http://localhost:8000/api/v1, Bearer token). Verify by listing servers and the GitHub source.
4. Keep the Coolify dashboard LAN-only. Don't expose it publicly.

## Phase 2: ingress
Tunnel mode:
1. Create a remotely-managed Cloudflare Tunnel via API (config_src: cloudflare), install it as a systemd service with `sudo cloudflared service install <token>` (don't echo the token).
2. Ingress starts with only the http_status:404 catch-all. Each hostname is added during its app's cutover, routed to http://localhost:80 (Coolify's Traefik).
3. Back up the current DNS records of all hostnames being migrated to dns-backup.json.
Direct mode: confirm 80/443 are reachable from the internet. List the DNS changes I'll have to make at my DNS provider per app, and tell me when to make them.

## Phase 3: per app, in the agreed order

### A. Stage (no downtime)
1. `railway link` to the project/environment/service in a scratch dir. `railway variables --json > ~/migration/<app>/railway-vars.json` (chmod 600). Same for the app's database service, to get its public connection URL (e.g. DATABASE_PUBLIC_URL). If the DB has no public TCP proxy, create a temporary one.
2. In Coolify: create a project, then a database of the same engine and **same major version** as on Railway. If the app's code forces SSL to the DB (grep the repo for `SSL Mode=Require`, `sslmode=require`, `ssl: true`), enable SSL on the Coolify DB.
3. Create the application from the GitHub App source: same repo/branch, auto-deploy on. Build pack: the Dockerfile Railway used (check railway.toml / railway.json), otherwise Nixpacks. Set the exposed port to the port the app listens on. Domain: `http://<domain>` in tunnel mode (TLS ends at Cloudflare; https here causes redirect loops), `https://<domain>` in direct mode. Health check path if one exists. **Don't deploy yet**: starting before the restore can let migrations create an empty schema.
4. Add persistent storage at the same mount paths as the Railway volumes.
5. Translate the env vars from railway-vars.json:
   - drop all RAILWAY_* vars
   - DATABASE_URL (and PG*/MYSQL*/REDIS* equivalents) → the Coolify database's internal URL
   - Railway injects PORT automatically and Coolify doesn't: set PORT explicitly if the app reads it
   - any value containing railway.app, rlwy.net or railway.internal → rewrite to the new host and tell me
   - grep the repo for code that reads RAILWAY_* (e.g. RAILWAY_PUBLIC_DOMAIN to build webhook/callback URLs) and add an explicit replacement var
   - copy everything else verbatim. Push in bulk from the file; never put values inline in commands you show.
6. Railway cron services → Coolify Scheduled Tasks with the same schedule.

### B. Cutover (downtime starts)
1. Copy volumes first, while the Railway service is still up: wake it (HTTP GET) and stream it out, e.g. `railway ssh … -- tar czf - -C <mount> . | base64` → decode → ~/migration/<app>/volume.tgz (check `railway ssh --help` for flags). Verify file count and total bytes on both sides. If streaming fails, stop and ask me.
2. `railway down` on the app service only (not the DB). This stops writes, and stops the old instance fighting the new one over bot webhooks or cron.
3. Dump: Postgres `pg_dump -Fc --no-owner --no-acl`; MySQL `mysqldump --single-transaction`; Mongo `mongodump --archive`. Redis: ask whether the data matters; usually start empty. Record table count and the row counts of the 5 largest tables.
4. Restore into the Coolify DB (`pg_restore --no-owner --no-acl`, etc.). Check extensions on the source first (`\dx`) and create them on the target. Compare counts.
5. Load the volume: `docker run --rm -v <vol>:/dst -v ~/migration/<app>:/src alpine tar xzf /src/volume.tgz -C /dst`, then chown to the container's runtime UID (check with `docker run --rm --entrypoint id <image>`).
6. Deploy in Coolify. Wait until healthy. Check logs for DB connection, migrations (should be no-ops) and errors.
7. Local test: `curl -sS -H "Host: <domain>" http://localhost/` → 2xx/3xx.
8. Switch traffic. Tunnel: add the hostname to the ingress, replace the DNS record with a proxied CNAME to <tunnel-id>.cfargotunnel.com (delete the old record first). Direct: tell me the exact record to set and wait for me.
9. Verify from outside: `curl -sS https://<domain>/` hits the new app. If the app registers webhooks (Telegram, Stripe, GitHub…), check they point at the right URL and are healthy, without printing tokens.
10. Remove the custom domain from the Railway service. Log the app as done.

### Rollback (if B6–B9 fails and isn't fixable in ~15 min)
Restore DNS from dns-backup.json, remove the tunnel hostname, `railway redeploy` the app on Railway, tell me. No data loss: the Railway DB was never modified.

## Phase 4: wrap-up
1. If BACKUP_S3_* is set: add S3 storage in Coolify and enable daily backups (keep 14) on every migrated database. Run one now and confirm the object lands in the bucket.
2. Enable unattended-upgrades.
3. Delete temporary Railway TCP proxies.
4. `shred -u` the dumps, volume tarballs and railway-vars.json once everything is verified.
5. Final report: per app, status, URL, Coolify project, env vars you changed, count checks. Remind me to revoke the Railway and Cloudflare tokens used here, delete the Railway projects after a week of good operation, and keep Coolify off the public internet.
```

## What you'll actually do by hand

1. Confirm the inventory and order Claude proposes.
2. About 5 minutes in Coolify's UI: admin account, enable API, create token, connect the GitHub App.
3. Approve downtime. Each app is down for roughly 10 minutes during its cutover.
4. Direct mode only: update DNS records when Claude tells you to.

## Gotchas I found while planning (the prompt handles them)

- **App domain must be `http://` in Coolify behind Cloudflare Tunnel.** With `https://` you get redirect loops.
- **Apps that hardcode DB SSL** (e.g. `SSL Mode=Require`) fail against a default Coolify Postgres. Turn on SSL in the DB's settings in Coolify.
- **`RAILWAY_PUBLIC_DOMAIN` disappears.** If your code builds webhook URLs from it, set the URL explicitly.
- **Don't let the app boot before the DB restore,** or EF/Prisma/etc. migrations create an empty schema and the restore collides with it.
- **Take the old instance down before the new one starts** if it registers bot webhooks, or the two keep overwriting each other.
- **Railway volumes can't be downloaded directly.** `railway ssh` + tar is the way. Verify file counts.

## Disclaimer

You're giving an AI agent tokens to your infrastructure. Read the prompt before running it, scope every token to the minimum, use fresh tokens and revoke them afterwards. Railway stays untouched as a rollback until you delete it yourself. Test with your least important app first.
