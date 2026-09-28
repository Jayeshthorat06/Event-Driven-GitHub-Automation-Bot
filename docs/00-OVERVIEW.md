# 00 — Project Overview & Full Requirements

> Give this file to Cursor FIRST. Then 01-BACKEND-SPEC, 02-FRONTEND-SPEC, 03-INTEGRATIONS-SETUP.

## 1. What we are building
An **Event-Driven GitHub Automation Bot**: a deployed web app where a user signs in with GitHub,
connects repositories, and a bot reacts to repo events (issues, pull requests, pushes) by
(a) writing back to GitHub (label / comment) and (b) sending a Slack notification.
A login-protected dashboard shows a live log of everything the bot did and lets the user configure rules.

```
GitHub repo --(webhook: issues/PR/push)--> POST /api/webhooks/github
    verify HMAC -> dedupe by delivery id -> store event -> enqueue job -> 202
                                   |
                        worker (DB-backed queue, retries)
                                   |
        evaluate rules -> [AI triage] -> actions: GitHub label/comment, Slack message
                                   |
                     events + actions tables -> Dashboard (SSE live log)
```

## 2. Requirement checklist (from the assignment)
### Core (mandatory)
| # | Requirement | Where implemented |
|---|---|---|
| C1 | Deployed, public web app (no localhost URLs) | Render free web service |
| C2 | GitHub sign-in; user connects a repo they own | Backend §2, §3 |
| C3 | Webhook endpoint receiving >= 2 event types (issues, pull_request, push) and recording them | Backend §4 |
| C4 | Bot writes back to GitHub for >= 1 event type (label or comment) | Backend §6 |
| C5 | Slack notification on configured event | Backend §7 |
| C6 | Dashboard behind login: events + actions taken | Frontend |
| C7 | README.md (run locally + how deployed) | Workflow doc §5 |

### Stretch (target all: S1, S2, S3, S4, S5)
| # | Requirement | Decision |
|---|---|---|
| S1 | Configurable rules in UI | Yes — rules table + rule editor |
| S2 | AI step (summary / label / priority) | Yes — Gemini or Groq free tier, shown in Slack + dashboard |
| S3 | Authenticate as **GitHub App** (JWT -> installation tokens) | Yes — single GitHub App handles login AND API access |
| S4 | Multi-repository per user | Yes — installation can cover many repos |
| S5 | Observability: structured logs, failures/retries visible | pino JSON logs + jobs/actions status in UI |

### Quality bar (graded heavily — treat as runs unattended)
1. **Not foolable by forged/replayed requests** -> HMAC-SHA256 signature check on raw body with constant-time compare; delivery-id dedupe; OAuth `state` CSRF check.
2. **No duplicate work on duplicate delivery** -> unique `delivery_id`; per-action idempotency key; comment marker check.
3. **No silent event loss** -> persist event BEFORE responding; DB-backed job queue with exponential backoff retries; stale-job recovery on restart; dead-letter state visible in UI with manual retry.
4. **Never expose secrets** -> env only; `.env.example` without values; pino redaction; nothing secret sent to client; per-repo Slack URLs encrypted at rest (AES-256-GCM); httpOnly cookies.

### Constraints
- Everything free, **no credit card anywhere**: GitHub, Neon (Postgres), Render, Slack, Gemini/Groq, UptimeRobot/cron-job.org.
- Any stack. We choose the one below.

## 3. Tech stack (decided — do not substitute)
- **Runtime:** Node.js 20 + TypeScript (strict)
- **Backend:** Express 4, `pg` + Drizzle ORM (migrations via drizzle-kit), `zod` validation, `pino` logging, `octokit` (`@octokit/auth-app`, `@octokit/rest`), `helmet`, `express-rate-limit`, `cookie-session` or `express-session` with PG store
- **Frontend:** React 18 + Vite + TypeScript + Tailwind CSS + React Router + TanStack Query. Built to `web/dist`, served statically by Express (ONE service, ONE URL -> simple OAuth/webhook config)
- **DB:** Neon Postgres (free)
- **Host:** Render free Web Service (persistent process, so the in-process worker works)
- **AI:** Google Gemini (AI Studio key) primary; Groq as alternative via `AI_PROVIDER`
- **Tests:** Vitest (signature verify, rule matcher, dedupe, backoff)

## 4. Repository layout
```
/
├─ AGENTS.md                 # AI context file (also copy to .cursor/rules/project.mdc)
├─ AI_NOTES.md               # required deliverable
├─ README.md                 # required deliverable
├─ .env.example
├─ docs/                     # these spec files
├─ server/
│  ├─ src/
│  │  ├─ index.ts            # app bootstrap, static hosting, worker start
│  │  ├─ config.ts           # zod-validated env
│  │  ├─ logger.ts           # pino + redaction
│  │  ├─ db/ (schema.ts, client.ts, migrations/)
│  │  ├─ auth/ (oauth.ts, session.ts, requireAuth.ts)
│  │  ├─ github/ (app.ts, api.ts, webhookVerify.ts)
│  │  ├─ webhooks/ (route.ts, handlers/)
│  │  ├─ rules/ (matcher.ts, templates.ts)
│  │  ├─ queue/ (enqueue.ts, worker.ts, backoff.ts)
│  │  ├─ actions/ (githubLabel.ts, githubComment.ts, slack.ts)
│  │  ├─ ai/ (triage.ts, providers/)
│  │  ├─ routes/ (repos.ts, rules.ts, events.ts, stream.ts, settings.ts)
│  │  └─ lib/crypto.ts       # AES-GCM helpers
│  └─ test/
└─ web/  (Vite React app)
```

## 5. Environment variables (`.env.example`)
```
NODE_ENV=development
PORT=3000
BASE_URL=http://localhost:3000        # prod: https://<your-app>.onrender.com
DATABASE_URL=                          # Neon connection string (sslmode=require)
SESSION_SECRET=                        # 32+ random bytes hex
ENCRYPTION_KEY=                        # 32 bytes base64 (openssl rand -base64 32)
GITHUB_APP_ID=
GITHUB_APP_SLUG=                       # from app URL github.com/apps/<slug>
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_PRIVATE_KEY_BASE64=             # base64 of the downloaded .pem
GITHUB_WEBHOOK_SECRET=
SLACK_WEBHOOK_URL=                     # optional global default
AI_PROVIDER=gemini                     # gemini | groq | none
GEMINI_API_KEY=
GROQ_API_KEY=
LOG_LEVEL=info
```

## 6. URLs you will register (replace `<BASE>` with the deployed URL)
| Where | Field | Value |
|---|---|---|
| GitHub App | Homepage URL | `<BASE>` |
| GitHub App | Callback URL | `<BASE>/api/auth/github/callback` |
| GitHub App | Setup URL (post-install) + "Redirect on update" | `<BASE>/api/github/setup` |
| GitHub App | Webhook URL | `<BASE>/api/webhooks/github` |
| GitHub App | Webhook secret | value of `GITHUB_WEBHOOK_SECRET` |
| Uptime pinger | URL | `<BASE>/api/health` (every 5–10 min, keeps Render awake) |

Full click-by-click steps: `03-INTEGRATIONS-SETUP.md`.

## 7. Build phases (Cursor should follow in order, commit after each)
1. Scaffold monorepo, config, logger, DB schema + migrations, health route
2. GitHub App login (OAuth), sessions, `/api/me`, protected-route middleware
3. Installation flow + repo sync
4. Webhook endpoint (verify, dedupe, persist, enqueue) + tests
5. Worker + retries + rules engine + GitHub write-back + Slack
6. Frontend: login, dashboard, repos, rules editor, event log w/ SSE
7. AI triage step
8. Hardening: rate limits, redaction audit, stale-job recovery, README, AI_NOTES, deploy

## 8. Definition of done
- Live URL: sign in -> install app on a repo -> open issue in repo -> within ~10 s a label/comment appears on GitHub, Slack message arrives, dashboard row appears with action statuses.
- Re-delivering the same webhook (GitHub "Redeliver") does NOT duplicate label/comment/Slack.
- Invalid signature -> 401 and nothing stored. Killing/restarting the server mid-processing loses no events.
- No secret appears in repo, client bundle, or logs.
