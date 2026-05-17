# warranty-getcoverspiff

CTL GetCover Spiff Dashboard — single-file SPA showing spiff-eligible GetCover warranty contracts grouped by invoice month and sales rep, with per-month CSV download.

**Live URL:** https://warranty.ctl.net/getcoverspiff/

## Eligibility rule

A warranty SKU qualifies for spiff when:

- `item_code` matches `^WRCB.{4,}GC$` (4+ chars between `WRCB` and `GC`), **and**
- `item_code != 'WRCB3025GC'` (explicitly excluded)

## How it gets served

Static GitHub Pages site at `https://jerm35.github.io/warranty-getcoverspiff/`. End users reach it via `https://warranty.ctl.net/getcoverspiff/`, where the [`warranty-router`](https://github.com/jerm35/warranty-router) Cloudflare Worker reverse-proxies `/getcoverspiff/*` to this repo.

Direct GitHub Pages access (`https://jerm35.github.io/warranty-getcoverspiff/`) won't render correctly — `/ctl-auth.js` is an absolute path that only resolves under `warranty.ctl.net`. Always test via `warranty.ctl.net/getcoverspiff/`.

## Files

- `index.html` — the app (HTML/CSS/JS inline)

## Dependencies

- `/ctl-auth.js` — shared CTL Google SSO library, served from `jerm35/warranty` via worker default-route
- `https://getcoverspiff-worker.jburnett-589.workers.dev` — dedicated CF Worker backend, `/gcs/*` API. Source: [`jerm35/getcoverspiff-worker`](https://github.com/jerm35/getcoverspiff-worker)
- `https://accounts.google.com/gsi/client` — Google Identity Services

## Access

- **CTL staff** — sign in with Google (any `@ctl.net` address is auto-approved on first SSO).
- **External users** — request access; an admin approves from the in-app admin panel.

## Deploy

```bash
CONTENT=$(base64 -i index.html)
SHA=$(gh api repos/jerm35/warranty-getcoverspiff/contents/index.html --jq '.sha')
jq -n --arg content "$CONTENT" --arg sha "$SHA" --arg msg "Update warranty-getcoverspiff" \
  '{message: $msg, content: $content, sha: $sha, branch: "main"}' \
  | gh api repos/jerm35/warranty-getcoverspiff/contents/index.html -X PUT --input - --jq '.commit.html_url'
```

GitHub Pages auto-deploys on push to `main`. Live in ~30s.

## Rollback

Remove `getcoverspiff` from `PREFIX_MAP` in `jerm35/warranty-router` and `npx wrangler@4 deploy`. The route 404s cleanly without affecting other apps.
