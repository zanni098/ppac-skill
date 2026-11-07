# PPAC: Post-Production App Check

> A Claude skill that audits any app you build before you ship it.

AI can generate a working app in minutes. It rarely generates a *safe* one, because nobody asks it to add auth checks, validate input, or write tests, and nobody verifies the output. PPAC turns that verification into a repeatable step: run it after you build, and it checks the code against a tiered checklist, fixes the blockers, and gives you a clear ship / don't-ship verdict.

## Why this exists

Vibe-coded apps tend to fail in the same few ways:

- API keys sitting in frontend code
- Routes with no auth, or auth with no ownership check (`/user/2` leaks someone else's data)
- SQL built from strings, unsanitized HTML, unhashed passwords
- `.env` committed to GitHub
- No tests, so every prompt-fix is a gamble

These are not exotic bugs. They are the default output when the person shipping the app does not review what was generated. PPAC is the review.

## What it does

When you run PPAC on a project, Claude will:

1. **Profile the app.** Detect the stack and work out what matters: public or private, user accounts, stored data, payments, paid APIs (LLMs, SMS).
2. **Run a tiered checklist** against the real code, with evidence (file and line, or command output) for every item.
3. **Fix the blockers** with the smallest change that closes each hole, then re-verify.
4. **Write tests** for the critical paths (auth, authorization, core logic, payments) and run them.
5. **Report** PASS / FAIL / N/A per item, a verdict, and the top 3 next actions.

## The checklist

The checklist is tiered on purpose. Treating "no caching layer" as equal to "API key in the frontend" is how checklists become noise.

### Tier 1: Blockers (never ship with a FAIL)

1. Secrets not exposed (frontend, bundle, git history)
2. Authentication on every non-public route
3. Authorization and IDOR checks on every route that takes an id
4. No injection (SQL, shell, `eval`)
5. No XSS (no raw user HTML)
6. Passwords hashed with bcrypt, argon2, or scrypt
7. Server-side input validation and upload limits
8. Rate limiting on login, signup, reset, and anything that costs money
9. CORS as an explicit allowlist when credentials are involved
10. Secure cookies and sessions, HTTPS, no leaked stack traces
11. Dependencies audited, lockfile committed

### Tier 2: Fix before real users

12. Error handling and timeouts around external calls
13. Logging and error tracking, with no secrets or PII in logs
14. Schema constraints, versioned migrations, backups with a tested restore
15. Tests exist and pass
16. Env-driven config, `.env.example`, fail loudly on missing vars
17. Health check and a rollback path
18. README with run, env, and deploy instructions

### Tier 3: Scale when needed

Only flagged if there is real traffic or data volume: indexes, N+1 queries, pagination, caching, background jobs, monitoring and alerts, load testing.

The full checklist with a concrete verification method for every item lives in [`ppac/SKILL.md`](ppac/SKILL.md).

## Principles

- **Verify, never assume.** "Looks fine" is not a pass. No evidence, no pass. If something cannot be checked, it is marked UNVERIFIED with the reason.
- **Tiered, not flat.** Blockers first. Scaling concerns only when they matter.
- **Smallest fix.** PPAC closes holes. It does not rewrite your app.
- **Secrets stay secret.** Anything found is redacted in the report, and leaked secrets are flagged for rotation.

## Install

**Claude Code**

```bash
git clone https://github.com/zanni098/ppac-skill.git
cp -r ppac-skill/ppac ~/.claude/skills/ppac
```

**Claude.ai**

Zip the `ppac` folder and upload it as a custom skill in Claude's skill settings.

**Anywhere else**

Paste the contents of `ppac/SKILL.md` into a chat and say: "Run this on my app."

## Usage

After you finish building something, type:

```
/PPAC
```

or just ask: "run PPAC on this app", "is this production ready?", "audit this before I ship".

## Example report

```
PPAC REPORT: invoice-tracker
Stack: Next.js, Postgres, Stripe   Profile: public, user accounts, payments

TIER 1: BLOCKERS
 1 Secrets ............ PASS  (no keys in src/, .env gitignored, history clean)
 2 Auth ............... PASS  (middleware.ts:12 guards /api/*)
 3 IDOR ............... FAIL -> FIXED (GET /api/invoices/[id] now checks owner, route.ts:21)
 8 Rate limiting ...... FAIL -> OPEN  (no limit on /api/login, proposed: upstash ratelimit)
 ...

TESTS: 6 written, 6 passing
VERDICT: DO NOT SHIP (1 open blocker)
TOP 3 NEXT ACTIONS:
 1. Add rate limiting to /api/login and /api/reset
 2. Enable database backups and test a restore
 3. Add error tracking
```

## What PPAC is not

- Not a penetration test or a security guarantee. It is a disciplined code review plus automated checks.
- Not able to see infrastructure it has no access to (cloud console settings, production database config). Those items are marked UNVERIFIED and left for you to confirm.
- Not a substitute for understanding your own code. It works best when you read the report.

## Contributing

Issues and pull requests are welcome, especially stack-specific verification commands (Django, Rails, Laravel, Go, mobile backends) and new checks that catch real failures.

## License

MIT. See [LICENSE](LICENSE).
