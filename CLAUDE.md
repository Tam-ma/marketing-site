# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Marketing / "Coming Soon" site for **Tamma** ("the autonomous development platform that maintains itself"), deployed on Cloudflare. Vanilla HTML/CSS frontend in `public/`, TypeScript edge code for an email-signup API backed by Cloudflare KV. No frontend framework, no build step for the HTML — pages are hand-authored and served as static assets.

This directory was **subtree-split out of the `meywd/tamma` monorepo**. Paths like `../LICENSE` and `../docs/stories/1-12-initial-marketing-website.md` (referenced in `package.json` and `README.md`) point at the old monorepo root and do **not** resolve here. Don't trust relative `../` references.

## Commands

```bash
npm run build        # tsx scripts/generate-sitemap.ts && tsc  — regenerate sitemap, then TYPECHECK
npm run dev          # wrangler dev — local server (Worker entry src/index.ts)
npm run deploy:prod  # wrangler deploy --env production  — deploys the WORKER
npm run preview      # wrangler deploy --env preview
npm run logs         # wrangler tail — live production logs
npm run kv:list      # list signup KV keys (binding=SIGNUPS)
```

- **There is no test runner and no linter.** `npm run validate*` are `echo` placeholders, not real checks. `tsc` (via `npm run build`) is the only automated gate — `outDir` is `dist/` but nothing serves `dist/`, so in practice `tsc` is a typecheck. tsconfig is strict (`noUnusedLocals`, `noUnusedParameters`, `noImplicitReturns`, full `strict`); unused vars/params will fail the build.
- To typecheck without regenerating the sitemap, run `npx tsc` directly.

## Deployment — two models, and they disagree

Be deliberate about which path you're touching:

1. **Worker model** (`wrangler.toml` → `main = "src/index.ts"`): used by `npm run deploy*`, `npm run dev`, and `wrangler tail`. The Worker serves `public/` via `[assets]`, redirects `/github/callback` → `https://api.tamma.dev/...`, and handles `POST /signup` inline.
2. **Pages model** (`.github/workflows/deploy.yml`): on push to `main`, CI runs `wrangler pages deploy public --project-name=tamma-marketing-site`. This deploys **only the `public/` directory as static files** — it does *not* bundle `src/index.ts`, and because only `public` is passed it does not pick up the root `functions/` directory either.

Consequence: **`functions/signup.ts` is not reached by the current CI deploy, and `src/index.ts` is not run by it.** If you change signup behaviour, confirm with the user which target is actually live before assuming your change ships. The KV namespace IDs in `wrangler.toml` are placeholders except production (`1a777a77...`).

## Signup logic is duplicated and has drifted

The same flow — email regex validation, normalize (trim+lowercase), 5-signups-per-hour-per-IP rate limit via KV TTL, store under `email:<addr>`, silently succeed on duplicates to avoid enumeration — exists in **both**:

- `src/index.ts` — `handleSignup()`, the Worker path.
- `functions/signup.ts` — `onRequestPost` / `onRequestOptions`, the Pages Function path.

They are not shared code and have already diverged (the Pages version increments the rate-limit counter before storage and on duplicate hits to harden against enumeration; the Worker version does not). **If you fix or change signup behaviour, apply it to both files** unless the user confirms one is dead.

KV key conventions (binding `SIGNUPS`): `email:<normalized-email>` (signup record, JSON), `ratelimit:<ip>` (counter, 1h TTL).

## Frontend (`public/`)

- `index.html` is the live landing page (Midnight Ocean redesign, applied in commit `1d23738`). **Its email form is cosmetic** — `onsubmit` only flips the button text to "Subscribed!" and clears the input; it does **not** POST to `/signup`. Wiring it to the backend is unfinished work, not a bug to silently "fix" without asking.
- Sibling `redesign-*.html`, `*-badges.html`, `*-logos.html`, `golden-stamps.html`, etc. are design prototypes/explorations, not routed pages. Don't assume edits there affect the live site.
- The explainer video is hosted on Bunny.net Stream and embedded via iframe using the `padding-bottom: 56.25%` aspect-ratio trick. Per README: do **not** put `display: grid` on the iframe's parent — it breaks iframe sizing.

## Scripts

- `scripts/generate-sitemap.ts` — rewrites `public/sitemap.xml` with today's date (single homepage URL). Runs as part of `npm run build`.
- `scripts/optimize-og-image.ts` — one-off `sharp` utility to compress `public/assets/og-image.png` below 300KB; backs up the original first. Not part of any automated flow.

## Notes

- The numerous `*_SUMMARY.md` / `*_GUIDE.md` / `QUICK*.md` files at the root are historical design/implementation write-ups. Treat them as background, not as current source of truth — verify against the actual files (e.g. README describes a working signup form and analytics that the live `index.html` doesn't currently have).
