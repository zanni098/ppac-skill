---
name: ppac
description: Post-production app check (PPAC). Run after building or finishing any app, API, site, or bot, and whenever the user types /PPAC or asks to "audit before shipping", "is this production ready", or "security check my app". Runs a tiered checklist (security, reliability, data, code quality), verifies every item against the real code with evidence, fixes blockers, and reports PASS / FAIL / N/A.
---

# PPAC: Post-Production App Check

**Core rule: verify, never assume.** Every item needs evidence (file:line or command output). "Looks fine" is not a pass. If something can't be verified (no deploy access, no DB access), mark it UNVERIFIED and say why.

## How to run

1. **Profile the app.** Detect the stack (package.json, requirements.txt, pyproject, etc.). Note: public or private? Has user accounts? Stores data? Takes payments? Calls paid APIs (LLMs, SMS)? This decides what is N/A.
2. **Run Tier 1 in full.** Run Tier 2 for any app with users or stored data. Run Tier 3 only if the app has real traffic or data volume; otherwise list as "later".
3. **Fix Tier 1 failures immediately**, then re-verify. Fix small Tier 2 failures. For the rest, list them with a proposed fix.
4. **Write tests for critical paths** if none exist (auth, authorization, core business logic, payments), run them, and report results.
5. **Output the report** in the format at the bottom.

## Tier 1: Blockers (never ship with a FAIL)

1. **Secrets not exposed.** No API keys in frontend code or bundle. `.env` is gitignored and never committed. No secret behind `NEXT_PUBLIC_`, `VITE_`, or `REACT_APP_` prefixes.
   *Verify:* `grep -rEn "(sk-|api[_-]?key|secret|token|password) *[:=]" src/`; `git log --all --oneline -- .env`; check `.gitignore`; inspect the built bundle. If a secret was ever committed, tell the user to **rotate it**. Never print the secret itself; redact it.
2. **Authentication on every non-public route.**
   *Verify:* list all routes; confirm each private one checks the session/token server-side.
3. **Authorization / IDOR.** Every route taking an id (`/user/:id`, `/order/:id`) checks the resource belongs to the caller. Role checks happen server-side, not just hidden in the UI.
   *Verify:* read each handler. Where possible test it: log in as user A, request user B's resource, expect 403/404.
4. **No injection.** Parameterized queries or ORM only. No string-built SQL (concatenation or template strings). No `eval`/`exec`/shell calls with user input.
   *Verify:* `grep -rEn "(query|execute)\(.*\+" src/`; grep for template-string SQL, `eval(`, `exec(`, `child_process`, `os.system`, `subprocess` with `shell=True`.
5. **No XSS.** User content is never injected as raw HTML.
   *Verify:* grep `dangerouslySetInnerHTML`, `innerHTML`, `v-html`, `|safe`. Each hit must be sanitized (e.g. DOMPurify) or removed.
6. **Passwords hashed** with bcrypt, argon2, or scrypt. Never logged, never returned in responses. Prefer managed auth over hand-rolled.
   *Verify:* read the signup/login code; check API responses and logs for password fields.
7. **Server-side input validation** (zod, pydantic, joi, etc.) on every endpoint that accepts input. Body size and file upload type/size limits set.
   *Verify:* read handlers; confirm validation runs before logic.
8. **Rate limiting** on login, signup, password reset, and any endpoint that costs money (LLM, SMS, email).
   *Verify:* confirm middleware exists and is applied to those routes.
9. **CORS is an explicit origin allowlist** when cookies or auth are used. `*` only for a truly public, credential-free API.
   *Verify:* read the CORS config.
10. **Sessions and transport.** Cookies are `httpOnly`, `secure`, `sameSite`. HTTPS enforced. Error responses never leak stack traces or internals.
    *Verify:* read cookie config; trigger an error and inspect the response.
11. **Dependencies clean.** No known critical/high vulnerabilities. Lockfile committed.
    *Verify:* `npm audit --omit=dev` / `pip-audit` / `cargo audit`.

## Tier 2: Fix before real users

12. **Error handling.** External calls (DB, APIs, LLMs) wrapped with try/catch and timeouts. User-friendly error messages. No unhandled promise rejections.
    *Verify:* read call sites; grep for bare `await` on network calls.
13. **Logging and error tracking.** Errors and auth events logged in a structured way. No secrets or PII in logs. Error tracker (e.g. Sentry) wired up or recommended.
    *Verify:* grep `console.log` near sensitive data; check logger config.
14. **Data layer.** Schema reviewed: types, NOT NULL, unique and foreign-key constraints. Migrations in version control. Backups enabled and a restore has been tested.
    *Verify:* read schema/migrations; ask the user about backups if not visible.
15. **Tests exist and pass** for auth, authorization, core logic, payments.
    *Verify:* run the test suite and report the actual output.
16. **Config.** Everything env-driven. `.env.example` committed. App fails loudly at startup if a required env var is missing.
    *Verify:* read the config loader.
17. **Deploy safety.** Health check endpoint. Previous version can be redeployed (rollback path).
    *Verify:* check for `/health`; check the deploy setup.
18. **README.** How to run, required env vars, how to deploy.
    *Verify:* read it.

## Tier 3: Scale when needed

Only flag these if the app has real traffic or data volume. Otherwise list under "Later" and move on. Do not over-engineer.

- Indexes on columns used in WHERE / JOIN / ORDER BY and on foreign keys
- N+1 queries (loops that run a query per item)
- Pagination on list endpoints
- Caching for expensive or repeated reads
- Background jobs for slow work
- Monitoring and alerts (uptime, error rate, latency)
- Load test of the critical path

## Report format

```
PPAC REPORT: <app name>
Stack: <...>   Profile: <public/private, users, payments, paid APIs>

TIER 1: BLOCKERS
 1 Secrets ............ PASS  (evidence)
 2 Auth ............... FAIL -> FIXED (what changed, file)
 3 IDOR ............... FAIL -> OPEN (why, proposed fix)
 ...

TIER 2: BEFORE REAL USERS
 ...

TIER 3: LATER
 ...

TESTS: <n> written, <n> passing
VERDICT: SHIP / SHIP AFTER FIXES / DO NOT SHIP
TOP 3 NEXT ACTIONS:
 1. ...
```

Verdict rules: any open Tier 1 FAIL means **DO NOT SHIP**. Open Tier 2 items mean **SHIP AFTER FIXES** for anything with real users.

## Rules

- Evidence for every PASS. No evidence, no pass.
- Mark N/A with a one-line reason (e.g. "no user accounts, so no auth needed").
- Never paste secrets into the report. Redact them.
- Fix what you can, but don't rewrite the app. Make the smallest change that closes the hole.
- Treat the checklist as a floor, not a ceiling: if you notice a serious problem that isn't listed, report it.
