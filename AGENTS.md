# AGENTS.md — Project rules for AI assistants (Cursor)

## Project
Event-Driven GitHub Automation Bot. Read `docs/00-OVERVIEW.md` first, then the spec for the area you are touching:
`docs/01-BACKEND-SPEC.md`, `docs/02-FRONTEND-SPEC.md`, `docs/03-INTEGRATIONS-SETUP.md`.
The specs are the source of truth. If you must deviate, say so and explain why before changing code.

## Stack (do not substitute)
Node 20 + TypeScript strict · Express · Drizzle + Postgres (Neon) · pino · zod · octokit (GitHub App auth) ·
React 18 + Vite + Tailwind + TanStack Query · Vitest. One deployable service (Express serves `web/dist`).

## Non-negotiable rules
1. **Webhook route uses the RAW body** for HMAC-SHA256 verification (`crypto.timingSafeEqual`). Never parse JSON before verifying.
2. **Persist first, process later:** insert event (unique `delivery_id`, `ON CONFLICT DO NOTHING`) + job in one transaction, then return 202. Slow work only in the worker.
3. **Idempotency everywhere:** every action has a unique `idempotency_key`; GitHub comments carry a hidden marker and are checked before posting. Retries must never duplicate output.
4. **Retries:** exponential backoff with jitter, max attempts, stale `running` job recovery, dead-letter state visible in UI, manual retry endpoint.
5. **Loop guard:** ignore events whose sender is a Bot / our app.
6. **Secrets:** only via env (validated by zod in `config.ts`). Never commit `.env` or `.pem`, never log tokens/URLs/keys (pino `redact`), never send secrets to the client, encrypt stored Slack URLs (AES-256-GCM).
7. **Authorization:** every `/api` route except health/auth/webhook requires a session; every `:id` resource must belong to the session user (return 404 otherwise).
8. **Untrusted input:** GitHub issue/PR text is untrusted — render as plain text, no user regex, AI output validated by zod and label whitelist.
9. **No paid services, no credit cards.** Free tiers only (Neon, Render, Slack, Gemini/Groq).
10. No `any`; no `console.log` (use the logger); no dead code; small focused modules.

## Working style
- Work in the phases listed in `docs/00-OVERVIEW.md §7`. One phase per task; commit at the end with a conventional message (`feat:`, `fix:`, `test:`, `docs:`).
- Before writing code for a phase, output a short plan (files to create/change). After coding, run typecheck, lint and tests and report results.
- Write tests for security/reliability-critical logic (signature verify, dedupe, matcher, backoff, idempotency).
- Ask before adding a new dependency; prefer stdlib.
- Keep `.env.example` and README in sync with any new env var.
- Append notable AI mistakes/corrections to `AI_NOTES.md` under "Wrong turns" as they happen (what was wrong, how detected, fix).

## Commands (keep updated)
```
npm run dev          # server (tsx watch) + web (vite) concurrently
npm run build        # web build + server tsc
npm run start        # migrate + node server/dist/index.js
npm run db:generate  # drizzle-kit generate
npm run db:migrate   # apply migrations
npm test             # vitest
npm run lint && npm run typecheck
```
