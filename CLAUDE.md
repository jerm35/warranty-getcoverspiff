# warranty-getcoverspiff — Frontend Repo

Single-file vanilla-JS SPA for the **GetCover Spiff** dashboard. Live at https://warranty.ctl.net/getcoverspiff/.

## What's here

- `index.html` — the entire app (HTML/CSS/JS inline, ~50KB). No build step.
- `docs/` — long-form [USER_GUIDE.md](docs/USER_GUIDE.md) and [ADMIN.md](docs/ADMIN.md) (plus `.docx`)
- `README.md` — short Ankane-style overview

## Companion components (not in this repo)

- **Backend Worker:** [`jerm35/getcoverspiff-worker`](https://github.com/jerm35/getcoverspiff-worker) — `/gcs/*` endpoints, BigQuery + auth
- **Routing:** [`jerm35/warranty-router`](https://github.com/jerm35/warranty-router) — proxies `warranty.ctl.net/getcoverspiff/*` to this repo's GH Pages
- **Shared SSO:** `/ctl-auth.js` served from `jerm35/warranty` (absolute path — only works behind `warranty.ctl.net`, NOT direct `jerm35.github.io`)

## Working in this file

- Edit `index.html` only — single source of truth
- `WORKER_URL` near the top (~line 470) points at the backend worker
- All worker endpoints are `/gcs/*`; localStorage keys are `gcs_*`
- Theme system uses `data-theme="light|light-dim|dark-dim|dark"` on `<html>`; accent hue via `--accent-h` CSS var (0–359)
- Auth flow: `checkExistingSession()` tries the stored worker token first, then falls back to the shared `ctl_token` (CTL SSO), then shows the login overlay
- Each spiff line in the table renders 3 sub-rows: CUSTOMER, SALES TEAM, DEVICE. The pipe-separated worker strings are parsed client-side by `renderSalesTeam()` and `renderDevices()`

## Deploy

```bash
# Edit index.html, then:
git add index.html
git commit -m "..."
git push          # GH Pages live in ~30 seconds
```

## Rollback

```bash
git revert HEAD   # or specific commit
git push
```

Worse-case takedown: remove the `getcoverspiff` entry from `PREFIX_MAP` in `warranty-router/src/index.js` and redeploy that worker. This repo can stay; `/getcoverspiff/` will 404 cleanly.

## Don't

- Don't add a build step or framework — the entire stack is "edit index.html, push." Keeping it dependency-free is the project's design intent.
- Don't break the `gcs_*` localStorage key prefix — would invalidate active sessions.
- Don't reference `/ctl-auth.js` differently — it's intentionally absolute so the shared SSO library cache works across all 28 sibling tools on `warranty.ctl.net`.

## Background

See [docs/ADMIN.md](docs/ADMIN.md) for architecture, deploy/rollback procedures, integrations, troubleshooting runbook, and roadmap.
