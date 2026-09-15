# GDPR + Cookie Consent Retrofit — Agent Prompt

Paste this into Claude Code at the root of the repo you want to retrofit. It's written to run as an audit-then-implement task, not a blind "add a cookie banner" task — most cookie banners fail compliance because trackers fire before consent, and this prompt exists to stop that specific failure mode.

---

## Context for the agent

This is a TypeScript app (tRPC / Drizzle / Postgres, pnpm monorepo — adjust if this repo differs). It needs to comply with GDPR and the ePrivacy Directive for EU users. Two legal facts that drive every implementation decision below:

1. **Strictly necessary** cookies/storage (session auth, CSRF, load balancing) need no consent, but must be disclosed.
2. **Everything else** (analytics, ads, embeds, third-party pixels/scripts) needs **prior, opt-in, granular consent** — the script must not execute until the specific category is accepted. A banner that displays *after* GA/Intercom/etc. has already loaded does not satisfy this.

Valid consent = freely given, specific, informed, unambiguous, opt-in (no pre-ticked boxes), reject-as-easy-as-accept, granular per category, logged with timestamp + policy version, and withdrawable anytime via a persistent control.

## Task 1 — Audit (do this first, report before changing anything)

Scan the codebase and list:
- Every cookie, localStorage/sessionStorage write, and third-party script/pixel/embed currently loaded (analytics, ads, chat widgets, video embeds, payment SDKs, error tracking, etc.)
- For each: what it does, whether it's strictly-necessary or needs consent, and where in the code it currently loads relative to any existing consent mechanism
- Every third-party processor implied by the above (hosting, analytics provider, email provider, payment provider) — flag any without a documented DPA
- Whether a privacy policy / cookie policy page currently exists and whether it's accurate to what's actually running
- Existing data subject request handling (account deletion, data export) if any

Present this as a table before writing implementation code.

## Task 2 — Implement consent infrastructure

**Consent state & gating**
- Build a `ConsentProvider` (React context, or equivalent for this stack) with categories: `necessary` (always true, not shown as a toggle), `functional`, `analytics`, `marketing` — trim categories to what the audit actually found, don't invent unused ones.
- Every non-necessary script must be gated behind its category's consent state — no script tag or SDK init call for a gated category should execute until that category is explicitly true. Prefer lazy dynamic import / conditional `next/script` (or framework equivalent) over "load then hide."
- Persist consent client-side (cookie or localStorage) AND server-side per user record if the app has auth — include: choice per category, ISO timestamp, and a policy version string.
- Add a Drizzle table (or reuse existing user metadata) for consent logging if server-side persistence applies:
  ```
  consent_log: id, user_id (nullable, for anon visitors), category, granted (bool), policy_version, created_at
  ```

**UI**
- Banner on first visit: Accept All / Reject All / Manage Preferences, all equally prominent — no dark patterns, no disabled reject button, no pre-checked toggles.
- "Manage Preferences" opens a per-category picker with plain-language descriptions (what it is, why it's there, third party involved).
- Persistent "Cookie settings" link in the footer that reopens the picker at any time, on every page.
- Don't block page content behind the banner (no full-page cookie wall) unless the app has a genuine reason to require tracking — flag this to me if it comes up rather than assuming it.

**Policy pages**
- Generate/update a Privacy Policy page and Cookie Policy page reflecting exactly what Task 1's audit found — real vendors, real categories, real retention periods where known. Don't template in generic boilerplate that doesn't match what's actually running; ask me for any specifics you can't infer from the codebase (data retention periods, business/contact details, EU rep if applicable).

**Data subject rights**
- Add a documented process (can be manual for MVP: e.g. a support email flow, or a simple account-settings "Request my data" button that emails me) for access, export, deletion, correction. Don't over-build a self-service portal unless one already exists — flag it as a manual process explicitly in the code comments/docs if that's the MVP choice.

## Task 3 — Output

- Summary of what was gated and how
- The audit table from Task 1
- Any DPA gaps found (processors without one on file) — list these, don't try to fix them, that's a legal/admin action for me
- Anything you couldn't determine from the codebase and need from me (business details, retention periods, whether an EU representative is needed)

## Constraints

- Smallest shippable slice: functioning consent gate + banner + policy pages that reflect reality. Don't build a full consent-management SaaS layer, admin analytics dashboard for consent rates, or multi-language i18n unless I ask.
- Don't remove or disable any strictly-necessary functionality while gating trackers — double-check auth/session cookies aren't accidentally caught by the analytics/marketing gate.
- This is engineering implementation, not legal sign-off — flag anything ambiguous (e.g. "is this embed necessary or marketing?") rather than guessing silently.
