# 04 — How to Build This in Cursor (step by step)

## 1. One-time setup
1. Create a new empty GitHub repo (e.g. `github-automation-bot`) and clone it.
2. Copy this pack into it so you have: `AGENTS.md` in the root and the four spec files in `docs/`.
3. Also create `.cursor/rules/project.mdc` with the front matter below and paste the body of `AGENTS.md` (Cursor reads `AGENTS.md` too, but `.mdc` lets you force it on):
   ```
   ---
   description: Project rules for the GitHub automation bot
   alwaysApply: true
   ---
   ```
4. Open the folder in Cursor (File -> Open Folder). Use **Agent mode** (Cmd/Ctrl+I) with a strong model.
5. Add `.gitignore` first: `node_modules, dist, .env, *.pem, .DS_Store`.
6. Commit: `docs: add specs and AI context`. (The assignment wants your AI context files committed exactly as used.)

## 2. Golden rules when prompting Cursor
- Reference files with `@` (e.g. `@docs/01-BACKEND-SPEC.md`) so they enter the context.
- **One phase per chat.** Start a new chat for each phase; the docs carry the context.
- Ask for a plan first, approve it, then let it code. Review the diff before accepting.
- After each phase: run the app/tests yourself, commit, and write down any AI mistake in `AI_NOTES.md` (the graders read that part most closely).
- Never paste real secrets into chat; use `.env` (git-ignored).

## 3. Phase prompts (copy/paste, one per new chat)

**Phase 0 — Kickoff**
> Read @AGENTS.md and @docs/00-OVERVIEW.md fully. Summarize the architecture and list any ambiguities or risks you see. Do not write code yet.

**Phase 1 — Scaffold**
> Following @docs/00-OVERVIEW.md §3–§5 and @AGENTS.md, scaffold the npm-workspaces monorepo (`server`, `web`), TypeScript strict, ESLint, Vitest, zod-validated `config.ts`, pino logger with redaction, Drizzle schema exactly as in @docs/01-BACKEND-SPEC.md §1 with the first migration, `GET /api/health` with DB ping, and `.env.example`. Show a plan first.

**Phase 2 — Auth**
> Implement @docs/01-BACKEND-SPEC.md §2: GitHub App OAuth login with state cookie CSRF check, session handling, `/api/me`, logout, and `requireAuth`. Add minimal Landing/Login pages and auth guard per @docs/02-FRONTEND-SPEC.md §1. Explain how I test it locally.

**Phase 3 — Installation & repos**
> Implement @docs/01-BACKEND-SPEC.md §3: GitHub App JWT/installation-token client, install redirect, setup callback with installation-ownership verification, repo sync, repos APIs, and the Repositories page from @docs/02-FRONTEND-SPEC.md §4.

**Phase 4 — Webhook receiver**
> Implement @docs/01-BACKEND-SPEC.md §4 exactly (raw body, constant-time HMAC, delivery-id dedupe with ON CONFLICT, loop guard, 202 fast). Write Vitest tests for: valid signature, invalid, tampered body, missing headers, duplicate delivery, bot sender ignored.

**Phase 5 — Worker, rules, actions**
> Implement @docs/01-BACKEND-SPEC.md §5, §6, §7: DB-backed queue with SKIP LOCKED claim, backoff+jitter, stale-job recovery, rules matcher, templates, idempotent GitHub label/comment actions (comment marker), Slack action with error classification, retry endpoint. Add tests for matcher, backoff, and idempotency with mocked Octokit/fetch.

**Phase 6 — Dashboard**
> Implement @docs/02-FRONTEND-SPEC.md §3, §5, §6 plus the read APIs and SSE in @docs/01-BACKEND-SPEC.md §9: Activity page with live updates + polling fallback, event drawer with retry, rules editor, settings page with Slack test.

**Phase 7 — AI triage**
> Implement @docs/01-BACKEND-SPEC.md §8 with Gemini and Groq providers behind `AI_PROVIDER`, zod-validated output, label whitelist, prompt-injection-safe prompt, graceful failure. Show AI summary/priority in Slack and dashboard.

**Phase 8 — Hardening & docs**
> Audit against the quality bar in @docs/00-OVERVIEW.md §2 and @docs/01-BACKEND-SPEC.md §10: helmet, rate limits, log redaction, secret-scan check, graceful shutdown. Then write README.md (what it does, run locally, env vars, deployment on Render, how to test using @docs/03-INTEGRATIONS-SETUP.md Step 7, known limitations).

## 4. Order of your own manual work
1. Do Phases 0–1, then **deploy an empty service to Render** to get `<BASE>` (see @docs/03-INTEGRATIONS-SETUP.md Steps 1–2).
2. Register the GitHub App with your URLs (Step 3) and add env vars in Render + local `.env`.
3. Continue Phases 2–8; push to `main` to auto-deploy; test on the live URL after each phase.
4. Set up Slack (Step 4) before Phase 5, AI key (Step 5) before Phase 7, uptime monitor at the end.

## 5. Deliverables checklist (from the assignment)
- [ ] GitHub repo with clear commit history (commit per phase)
- [ ] Deployed URL working and reachable
- [ ] `README.md`: what it does, run locally, env vars, `.env.example`, how/where deployed
- [ ] How to test: instructions + demo repo (invite reviewers or make it public) + Slack channel screenshot/invite
- [ ] `AGENTS.md` / `.cursor/rules` exactly as used (commit them)
- [ ] `AI_NOTES.md` (~1 page)

## 6. `AI_NOTES.md` template (fill in honestly as you go)
```md
# AI Notes

## Tools & split of work
- Tools/models: Cursor (Agent mode) with <model name>; ChatGPT/Claude for spec drafting.
- Me: architecture, data model, service choices, reviewing every diff, live debugging, deployment.
- AI: boilerplate, endpoint implementations, tests, UI components, first drafts of docs.

## Key decisions I made
1. GitHub App instead of OAuth App: one registration gives login + installation tokens + multi-repo + app-level webhook.
2. Postgres-backed job queue (SKIP LOCKED) instead of Redis/in-memory: survives restarts, free, visible in UI.
3. Single Express service serving the SPA: one URL to register for OAuth/webhooks on a free host.

## Hardest bug / wrong turn the AI led me into
- What the AI did: <e.g. parsed JSON before verifying the HMAC, so signatures always failed>
- How I noticed: <symptom, logs, GitHub "Recent deliveries" showing 401>
- How I fixed it: <raw body middleware order + test that locks it in>

## What I'd improve with more time
- Per-org rule templates, dedicated worker process/Redis, Slack OAuth app with per-channel routing, metrics/alerts, e2e tests.
```

## 7. README skeleton for Cursor to fill
Sections: Overview · Architecture diagram · Features (core + stretch) · Prerequisites · Local setup (install, `.env`, migrate, smee/cloudflared tunnel, dev GitHub App) · Environment variables table · Deployment (Neon + Render + keep-alive) · How to test (Step 7 script) · Security & reliability notes (HMAC, dedupe, retries, idempotency, secret handling) · Known limitations · Project structure.
