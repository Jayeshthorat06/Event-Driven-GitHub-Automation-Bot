# 02 — Frontend Specification (UI + Client Logic)

Stack: React 18, Vite, TypeScript, Tailwind CSS, React Router v6, TanStack Query.
Build output `web/dist` is served by Express; the SPA fallback returns `index.html` for any non-`/api` route.
All API calls use `fetch` with `credentials: 'include'` and header `X-Requested-With: fetch`. **No secrets in client code**; only relative URLs (`/api/...`).

## 1. Routes
| Path | Access | Page |
|---|---|---|
| `/` | public | Landing + "Sign in with GitHub" button |
| `/login` | public | Same button, shows error param (`?error=`) |
| `/dashboard` | auth | Activity (live log) — default tab |
| `/dashboard/repos` | auth | Repositories (connect/manage) |
| `/dashboard/rules` | auth | Rules editor (per repo) |
| `/dashboard/settings` | auth | Slack + account |

**Auth guard:** on app load call `GET /api/me`. 401 -> redirect to `/login`. While loading show a full-page spinner (no flash of protected content). Nothing from the dashboard is fetched or rendered without a valid session.

"Sign in with GitHub" = plain link/navigation to `/api/auth/github/login` (full page redirect, not fetch).

## 2. Layout
- Top bar: app name, user avatar + login, "Logout" (POST `/api/auth/logout` then go to `/`).
- Left/top tabs: Activity, Repositories, Rules, Settings.
- Responsive; dark-mode friendly with Tailwind `dark:` classes. Every list has **loading (skeleton)**, **empty**, and **error (with retry)** states.

## 3. Activity page (live log) — the main screen
- **Stat cards (24h):** Received, Done, Retrying, Failed (from `/api/stats`, refetch on SSE events).
- **Filters:** repo select (all/one), event type (issues/pull_request/push), status (all/done/partial/failed/processing). Filters go into the query string.
- **Table/list of events** (newest first, keyset "Load more"):
  | Time (relative + tooltip) | Repo | Event (badge: issues/PR/push + action) | Summary (link to GitHub URL) | Sender | AI (priority chip + 1-line summary) | Actions (chips: label ✓, comment ✓, slack ✗) | Status badge |
- Status colors: done=green, processing=blue, partial=amber, failed=red, received=gray. Action chips show tooltip with attempts + error.
- **Row click -> right-side drawer** (`GET /api/events/:id`): summary, delivery id, timeline of attempts (job attempts, last error, next retry time), each action with status/result/error, matched rules, AI output, collapsible raw payload (JSON viewer, read-only). Buttons: **"Retry"** (POST `/api/events/:id/retry`, shown for failed/partial) and "Open on GitHub".
- **Live updates:** open `EventSource('/api/stream')`. On `event.received` prepend row (highlight fade); on `event.updated`/`action.updated` patch the row in TanStack Query cache. Show a small "● Live" indicator; if the SSE connection drops, show "Reconnecting…" and fall back to polling every 5 s until reconnected. Pause polling when tab hidden.

## 4. Repositories page
- Header buttons: **"Connect GitHub repositories"** (navigate to `/api/github/install`) and **"Sync"** (POST `/api/repos/sync`).
- Card/row per repo (`GET /api/repos`): full name, private badge, last event time, rule count, enable toggle (PATCH `enabled`), "Slack override" field (input + Save; shows "Configured ✓" but never the URL itself; "Clear" button), link to Rules filtered by this repo.
- Empty state: explains the GitHub App must be installed, with the connect button.
- After returning from install (`/dashboard/repos`), auto-refetch.

## 5. Rules page (S1)
- Repo selector at top -> list of rules for that repo (`GET /api/repos/:id/rules`) with name, event type, enabled switch, summary sentence ("When **issues.opened** and title **contains** `bug` → label `bug`, Slack"), Edit / Delete (confirm dialog).
- **Rule editor (modal or side panel), fields:**
  1. Name (required, <= 80 chars)
  2. Event type (select: issues.opened, issues.edited, issues.reopened, pull_request.opened, pull_request.synchronize, pull_request.reopened, push)
  3. Match mode: All / Any
  4. Conditions (repeatable rows): field select (title, body, author, labels, branch, ai_priority, ai_label) · operator select (contains, not_contains, equals, not_equals, starts_with, in) · value input. Add/remove rows. Zero conditions allowed = matches every event of that type.
  5. Actions (repeatable, max 5): type select
     - **Add label(s):** tag input, or checkbox "Use AI-suggested label"
     - **Post comment:** textarea with template variable chips (`{{title}} {{author}} {{number}} {{repo}} {{url}} {{branch}} {{ai_summary}} {{ai_priority}}`) that insert at cursor
     - **Send Slack alert:** textarea message template
     Push events: only Slack action is selectable (disable others with hint).
  6. Enabled toggle
- Validate client-side with zod (same shape as server); show inline server validation errors.
- "Quick start" button: adds example rule "Issues containing 'bug' → label bug + Slack alert".

## 6. Settings page
- Default Slack webhook URL: input (type=password style), Save, **"Send test message"** (POST `/api/settings/slack/test`, shows success/error toast). Never display the stored value; show "Configured ✓".
- Account: GitHub login, avatar, "Log out".
- Link: "Manage GitHub App installation" -> `https://github.com/settings/installations`.

## 7. Client architecture
```
web/src/
  main.tsx, App.tsx, routes.tsx
  lib/api.ts          # fetch wrapper: JSON, error normalize, 401 -> redirect /login
  lib/sse.ts          # EventSource hook with reconnect + polling fallback
  hooks/ (useMe, useEvents, useRepos, useRules, useStats)
  components/ (Layout, StatusBadge, ActionChip, EventDrawer, RuleEditor, ConditionRow,
               ActionRow, EmptyState, ErrorState, Skeleton, Toast, ConfirmDialog, JsonViewer)
  pages/ (Landing, Login, Activity, Repos, Rules, Settings)
  types.ts            # shared DTO types (mirror server zod schemas)
```
- TanStack Query keys: `['me']`, `['events', filters]`, `['event', id]`, `['repos']`, `['rules', repoId]`, `['stats']`.
- Mutations invalidate relevant keys; optimistic update for enable toggles with rollback on error.
- Accessibility: labelled inputs, focus trap in drawer/modals, keyboard-closable, sufficient contrast, status conveyed by text not only color.
- Security: render all GitHub-sourced text as plain text (React escaping; never `dangerouslySetInnerHTML`); only make links for `https://github.com/...` URLs.

## 8. Acceptance checks (manual)
1. Visiting `/dashboard` logged-out redirects to `/login`.
2. After login, Activity shows a new row within seconds of opening an issue, without page refresh.
3. Failed action shows red chip + error tooltip; Retry button works and row turns green.
4. Creating a rule in UI changes bot behavior on the next event (no redeploy).
5. Slack URL is never visible in the network tab responses or page source.
