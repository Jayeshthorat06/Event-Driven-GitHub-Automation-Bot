# 03 — Integrations Setup (GitHub App, Slack, Neon, Render, AI)

Order matters: **deploy first to get a public URL, then register the GitHub App with that URL.**
All steps are free / no card.

## Step 1 — Neon Postgres
1. Sign up at neon.tech -> create project -> copy the **connection string** (`sslmode=require`) -> `DATABASE_URL`.
2. Run migrations: `npm run db:migrate --workspace server`.

## Step 2 — Render (get the public URL)
1. Push the repo to GitHub.
2. Render dashboard -> New -> **Web Service** -> connect repo. Runtime Node, Instance type **Free**.
3. Build command: `npm ci && npm run build` (builds web + server). Start command: `npm run start` (runs migrations then `node server/dist/index.js`).
4. Add env vars from `.env.example` (leave GitHub ones empty for now). Set `BASE_URL=https://<service>.onrender.com`.
5. Deploy; open `<BASE>/api/health` -> should return `{"ok":true}`.
6. **Keep-alive:** free Render services sleep after ~15 min idle (cold start ≈ 30–60 s, which can make GitHub time out at 10 s). Create a free monitor at uptimerobot.com or cron-job.org hitting `<BASE>/api/health` every 5 min. The queue design still protects you: a failed delivery can be redelivered from GitHub -> App -> Advanced -> Recent Deliveries.

## Step 3 — Register the GitHub App
GitHub -> Settings -> Developer settings -> GitHub Apps -> **New GitHub App**.

| Field | Value |
|---|---|
| Name | e.g. `yourname-repo-bot` (globally unique) -> this becomes `GITHUB_APP_SLUG` |
| Homepage URL | `<BASE>` |
| Callback URL | `<BASE>/api/auth/github/callback` |
| Expire user authorization tokens | leave default (on) |
| **Request user authorization (OAuth) during installation** | ✅ checked |
| Setup URL | `<BASE>/api/github/setup` |
| Redirect on update | ✅ checked |
| Webhook -> Active | ✅ |
| Webhook URL | `<BASE>/api/webhooks/github` |
| Webhook secret | generate: `openssl rand -hex 32` -> `GITHUB_WEBHOOK_SECRET` |
| SSL verification | Enabled |

**Repository permissions:** Issues: **Read & write** · Pull requests: **Read & write** · Contents: **Read-only** (required for push events) · Metadata: Read-only (automatic).
**Account permissions:** none needed (login uses `read:user` implicitly; email not required).
**Subscribe to events:** Issues, Pull request, Push. (Installation events are delivered automatically.)
**Where can this app be installed?** "Any account" (so reviewers can install on their demo repo) — or "Only on this account" if you'll invite them as collaborators to a repo you own.

After creating:
1. Copy **App ID** -> `GITHUB_APP_ID`.
2. Copy **Client ID** -> `GITHUB_CLIENT_ID`; **Generate a new client secret** -> `GITHUB_CLIENT_SECRET`.
3. **Generate a private key** (.pem downloads). Encode: `base64 -w0 your-app.private-key.pem` (mac: `base64 -i file.pem | tr -d '\n'`) -> `GITHUB_PRIVATE_KEY_BASE64`. Delete/secure the .pem; never commit it.
4. Put all values into Render env vars -> redeploy.

## Step 4 — Slack Incoming Webhook
1. Create a free workspace (or use existing) -> api.slack.com/apps -> **Create New App** -> From scratch.
2. Features -> **Incoming Webhooks** -> On -> **Add New Webhook to Workspace** -> pick channel (e.g. `#github-alerts`).
3. Copy URL (`https://hooks.slack.com/services/...`). Either set as `SLACK_WEBHOOK_URL` (global default) or paste it in the app's Settings page / per-repo override.

## Step 5 — AI provider (free, stretch)
- **Gemini:** aistudio.google.com -> Get API key -> `GEMINI_API_KEY`, `AI_PROVIDER=gemini`.
- **Groq:** console.groq.com -> API Keys -> `GROQ_API_KEY`, `AI_PROVIDER=groq`.
- Free tiers are rate-limited: the code must degrade gracefully (skip AI, continue).

## Step 6 — Local development with webhooks
GitHub can't reach localhost. Options (free):
- **smee.io:** create a channel; run `npx smee-client -u https://smee.io/XXXX -t http://localhost:3000/api/webhooks/github`; use the smee URL as the webhook URL on a SECOND dev GitHub App (with callback `http://localhost:3000/api/auth/github/callback`). Signature verification still works because smee forwards headers/body unchanged.
- **cloudflared:** `cloudflared tunnel --url http://localhost:3000` (no account needed for quick tunnels) and use the printed URL as `BASE_URL` for a dev App.
Recommended: keep a separate "dev" GitHub App and a "prod" GitHub App with different secrets.

## Step 7 — End-to-end verification script (use in README "How to test")
1. Open `<BASE>` -> **Sign in with GitHub**.
2. Repositories -> **Connect GitHub repositories** -> choose the demo repo -> Install.
3. Settings -> paste Slack webhook -> **Send test message** (arrives in Slack).
4. Rules -> Quick start rule (issues.opened, title contains `bug` -> label `bug` + Slack).
5. In the demo repo open an issue titled "Bug: login fails".
6. Within ~10 s: label `bug` on the issue, Slack message, new Activity row (live).
7. GitHub App -> Advanced -> Recent deliveries -> **Redeliver** -> confirm no duplicate label/comment/Slack (dashboard shows no second event).
8. Negative test: `curl -X POST <BASE>/api/webhooks/github -H 'X-GitHub-Event: issues' -H 'X-GitHub-Delivery: x' -H 'X-Hub-Signature-256: sha256=bad' -d '{}'` -> **401**.
9. Failure test: temporarily set a bad Slack URL -> Activity shows failed/retrying with attempts; fix URL -> click Retry -> succeeds.

## Troubleshooting
| Symptom | Cause / fix |
|---|---|
| Webhook deliveries show 401 | Secret mismatch, or JSON body parsed before signature check (must use raw body) |
| 502/timeout on first delivery | Render cold start -> keep-alive monitor; redeliver |
| Login redirect_uri mismatch | Callback URL in App settings != `<BASE>/api/auth/github/callback` exactly |
| Bot comments loop forever | Loop guard missing (ignore `sender.type=Bot`) |
| 403 "Resource not accessible by integration" | Missing permission (Issues/PR write) — update permissions; owner must accept the new permissions on the installation |
| Setup redirect 403 | Installation not in `GET /user/installations` for that user |
