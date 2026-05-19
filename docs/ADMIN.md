# GetCover Spiff — Admin Reference

> System architecture, deployment procedures, and operational runbook for the GetCover Spiff dashboard.

**Version:** 1.0 | **Status:** Production | **Last Updated:** 2026-05-18

For end-user documentation see [USER_GUIDE.md](USER_GUIDE.md).

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Repositories](#repositories)
- [Setup and Deployment](#setup-and-deployment)
- [Configuration](#configuration)
- [Integrations and APIs](#integrations-and-apis)
- [Data Sources](#data-sources)
- [Security and Access Control](#security-and-access-control)
- [Maintenance and Monitoring](#maintenance-and-monitoring)
- [Admin Troubleshooting](#admin-troubleshooting)
- [Known Issues and Limitations](#known-issues-and-limitations)
- [Roadmap](#roadmap)

---

## Architecture

The system is four layers behind a single user-facing domain:

```
user → warranty.ctl.net/getcoverspiff/
     → warranty-router (Cloudflare Worker, PREFIX_MAP routing)
     → jerm35.github.io/warranty-getcoverspiff (GitHub Pages, single index.html)
     → getcoverspiff-worker.jburnett-589.workers.dev (Cloudflare Worker)
     → BigQuery (ctl-repair-data.netsuite.* + fixably.netsuite_devices)
```

**Components:**

| Component | Role | Notes |
|---|---|---|
| `warranty-router` (CF Worker) | Reverse-proxies `warranty.ctl.net/*` to per-app GitHub Pages repos | Shared across 28+ sibling tools; PREFIX_MAP in `src/index.js` |
| `warranty-getcoverspiff` (GH Pages) | Static single-page SPA — auth UI, dashboard table, CSV download trigger | Single `index.html`, no build step |
| `getcoverspiff-worker` (CF Worker) | Backend: session auth, BigQuery proxy, CSV generation, admin user management | Endpoints under `/gcs/*`; KV-backed sessions/users |
| `sales-etl` (Cloud Run Job) | Daily NetSuite → BigQuery ETL covering invoices, lines, customers, items, fulfillments, sales team | Job name `netsuite-sales-etl`, region `us-central1` |
| BigQuery datasets | Source-of-truth data store for invoices, items, sales team, and per-device serials | Project `ctl-repair-data` |

**Data flow per request:**

```
[User clicks month] → fetch /gcs/details?month=YYYY-MM
                    → worker validates session token (KV lookup)
                    → worker mints BigQuery access token from service-account JWT
                    → worker runs BigQuery SQL (spiff_lines + device_lines + sales_team CTEs)
                    → worker returns JSON
                    → frontend renders 5-column table + 3 sub-rows per spiff line
```

```
[User clicks CSV] → fetch /gcs/csv?month=YYYY-MM
                  → worker runs csvSerialQuery (joins fixably.netsuite_devices for per-serial expansion)
                  → worker streams text/csv response with Content-Disposition: attachment
```

---

## Tech Stack

| Layer | Technology | Hosting |
|---|---|---|
| Frontend | HTML/CSS/Vanilla JS (no build) | GitHub Pages (`jerm35.github.io/warranty-getcoverspiff/`) |
| Backend | Cloudflare Workers (ES modules, `nodejs_compat`) | Cloudflare account `jburnett-589` |
| State | Cloudflare KV (4 namespaces: USERS, SESSIONS, PENDING, PREFS) | Cloudflare |
| Auth | Google Identity Services (SSO) + bcrypt-equivalent SHA-256+salt for passwords | Worker-side |
| Data store | BigQuery (`ctl-repair-data` project, `netsuite` + `fixably` datasets) | GCP |
| ETL | Python 3.11 + `requests-oauthlib`, deployed as Cloud Run Job | GCP |
| Routing | Cloudflare Worker (`warranty-router`) reverse-proxying GH Pages by path prefix | Cloudflare |

---

## Repositories

| Repo | Purpose | Tracking |
|---|---|---|
| `jerm35/warranty-getcoverspiff` | Frontend (single `index.html`) | Public; GH Pages auto-deploys on push to `main` |
| `jerm35/getcoverspiff-worker` | Backend Cloudflare Worker | Private |
| `jerm35/warranty-router` | Shared path-prefix router (`/getcoverspiff/` entry added in v0.29.0) | Existing; one-line addition for this project |
| `jerm35/warranty` | Hosts shared `/ctl-auth.js` Google SSO library | Existing; no change needed |
| **Not yet tracked:** `/path/to/Claude/NetSuite/sales-etl/main.py` | Daily NetSuite ETL | Local file only — see [Known Issues](#known-issues-and-limitations) |

---

## Setup and Deployment

### Prerequisites

- Node 20+, `npx` available
- Cloudflare account access (`jburnett-589`) and `wrangler login` completed
- `gcloud` authenticated as `jburnett@ctl.net`, default project `ctl-repair-data`
- GitHub CLI (`gh`) authenticated as `jerm35`

### Initial Deployment (one-time)

These steps were completed when the project was first built (2026-05-16). Reference for cloning the pattern to a new sibling tool.

**1. Create the 4 KV namespaces and capture their IDs:**

```bash
cd /path/to/getcoverspiff-worker
for ns in gcs_users gcs_sessions gcs_pending gcs_prefs; do
  npx wrangler@4 kv namespace create "$ns"
done
# Paste each returned ID into wrangler.toml under the matching binding
```

**2. Set the worker secrets:**

```bash
cd /path/to/getcoverspiff-worker

# BigQuery service account — create a new key for bq-reader-sales, pipe it in, then delete the local file
KEYFILE=$(mktemp -t bq-sa.XXXXXX.json)
gcloud iam service-accounts keys create "$KEYFILE" \
  --iam-account=bq-reader-sales@ctl-repair-data.iam.gserviceaccount.com \
  --project=ctl-repair-data
cat "$KEYFILE" | npx wrangler@4 secret put GCP_SERVICE_ACCOUNT_KEY
rm -f "$KEYFILE"

# Public Google OAuth client ID — same value as the rest of CTL's warranty tools
echo "264235504831-ahd6lj613ff75tcpp51jqdifodbgeinm.apps.googleusercontent.com" \
  | npx wrangler@4 secret put GOOGLE_CLIENT_ID

# First admin email — auto-promoted to admin on first cold start
echo "jburnett@ctl.net" | npx wrangler@4 secret put INITIAL_ADMIN_EMAIL
```

**3. Deploy the worker:**

```bash
cd /path/to/getcoverspiff-worker
npx wrangler@4 deploy
# Live at https://getcoverspiff-worker.jburnett-589.workers.dev
```

**4. Push the frontend repo and enable GitHub Pages:**

```bash
cd /path/to/warranty-getcoverspiff
gh repo create jerm35/warranty-getcoverspiff --public --source=. \
  --description "CTL GetCover Spiff Dashboard" --push
gh api -X POST 'repos/jerm35/warranty-getcoverspiff/pages' --input - <<'EOF'
{"source": {"branch": "main", "path": "/"}}
EOF
```

**5. Add the prefix to `warranty-router` and deploy:**

Edit `/path/to/warranty-router/src/index.js`, add `"getcoverspiff": "warranty-getcoverspiff",` to `PREFIX_MAP`, bump `VERSION`, then:

```bash
cd /path/to/warranty-router
npx wrangler@4 deploy
git add src/index.js && git commit -m "Add getcoverspiff route" && git push
```

**6. Extend `sales-etl` for sales team data (one-time, only if not already in BigQuery):**

```bash
# Run locally to backfill 200 days of history
cd /path/to/Claude/NetSuite/sales-etl
python3 main.py --incremental --since-days 200

# Deploy to Cloud Run so the daily job picks up the new extractor
bash deploy/deploy.sh
# The deploy.sh `gcloud run jobs update` step may fail on already-set secrets.
# Workaround: just update the image, leave env vars alone:
gcloud run jobs update netsuite-sales-etl --region=us-central1 \
  --project=ctl-repair-data \
  --image=us-central1-docker.pkg.dev/ctl-repair-data/netsuite-sales-etl/etl:latest
```

### Routine Re-deployment

**Frontend** (auto-deploys to GH Pages):

```bash
cd /path/to/warranty-getcoverspiff
# edit index.html
git add index.html
git commit -m "..."
git push  # GH Pages live in ~30 seconds
```

**Worker:**

```bash
cd /path/to/getcoverspiff-worker
npx wrangler@4 deploy
git add -A
git commit -m "..."
git push
```

**Router** (only when adding/removing tools from PREFIX_MAP):

```bash
cd /path/to/warranty-router
# edit src/index.js PREFIX_MAP and bump VERSION
npx wrangler@4 deploy
git add -A
git commit -m "..."
git push
```

### Rollback

**Frontend rollback** (revert to a prior commit):

```bash
cd /path/to/warranty-getcoverspiff
git revert HEAD          # or git revert <bad-commit-sha>
git push                 # GH Pages updates in ~30 seconds
```

**Worker rollback** (Cloudflare keeps version history; roll back via dashboard or CLI):

```bash
cd /path/to/getcoverspiff-worker
git checkout <last-good-commit> -- src/
npx wrangler@4 deploy
git checkout main -- src/   # restore working tree
```

**Full takedown of just this tool:**

```bash
cd /path/to/warranty-router
# Remove the "getcoverspiff" line from PREFIX_MAP in src/index.js
npx wrangler@4 deploy
# /getcoverspiff/ now returns 404 without affecting the other 27 tools
```

---

## Configuration

### Worker environment variables (`getcoverspiff-worker/wrangler.toml` `[vars]` section)

| Variable | Value | Description |
|---|---|---|
| `BQ_PROJECT` | `ctl-repair-data` | GCP project for BigQuery queries |
| `BQ_DATASET` | `netsuite` | Primary BQ dataset (also queries `fixably` dataset for serials) |
| `ALLOWED_CTL_DOMAIN` | `ctl.net` | Domain that gets auto-approved on Google SSO and qualifies for NetSuite deep links |
| `NETSUITE_ACCOUNT_ID` | `11277173` | Used to construct NetSuite deep-link URLs |

### Worker secrets (set via `npx wrangler@4 secret put`)

| Secret | Description | Rotation |
|---|---|---|
| `GCP_SERVICE_ACCOUNT_KEY` | Full JSON key for `bq-reader-sales@ctl-repair-data.iam.gserviceaccount.com` | Rotate annually or on staff departure; regenerate via `gcloud iam service-accounts keys create` |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID (public value, but stored as a secret for parity with other workers) | Match the value in `/ctl-auth.js`; rotate only if the OAuth client is rotated |
| `INITIAL_ADMIN_EMAIL` | Email auto-promoted to admin on first cold start | Change only if seeding a different first admin |

### Worker KV namespaces (bindings in `wrangler.toml`)

| Binding | Purpose | Key prefix |
|---|---|---|
| `USERS` | Approved user records | `u:<email>` |
| `SESSIONS` | Active session tokens (30-day TTL) | `s:<token>` |
| `PENDING` | Pending self-signup requests awaiting admin approval | `p:<email>` |
| `PREFS` | Per-user theme/accent/font-size preferences | `pf:<email>` |

### sales-etl environment (`/path/to/Claude/NetSuite/sales-etl/.env`)

| Variable | Description |
|---|---|
| `NETSUITE_ACCOUNT_ID` | `11277173` |
| `NETSUITE_CONSUMER_KEY` / `_SECRET` | NetSuite Integration record |
| `NETSUITE_TOKEN_ID` / `_SECRET` | NetSuite Token-Based Auth token |
| `GCP_PROJECT_ID` | `ctl-repair-data` |
| `BQ_DATASET` | `netsuite` |

> WARNING: Never commit `.env`. The Cloud Run Job's env vars are set via the deploy script and stored in GCP. Pull a fresh service-account key only when needed and delete the local file immediately.

---

## Integrations and APIs

### BigQuery

- **Purpose:** Primary data source for all dashboard queries and CSV exports
- **Authentication:** JWT signed with `GCP_SERVICE_ACCOUNT_KEY`, exchanged at `https://oauth2.googleapis.com/token` for an access token scoped to `bigquery.readonly`
- **Datasets queried:** `ctl-repair-data.netsuite.*` and `ctl-repair-data.fixably.netsuite_devices`
- **Client implementation:** [`getcoverspiff-worker/src/bigquery.js`](../getcoverspiff-worker/src/bigquery.js) — minimal pure-fetch client, WebCrypto JWT signing

### Google Identity Services (SSO)

- **Purpose:** `@ctl.net` staff sign-in
- **Client ID:** `264235504831-ahd6lj613ff75tcpp51jqdifodbgeinm.apps.googleusercontent.com` (shared across CTL warranty tools)
- **Library:** `/ctl-auth.js`, served by the default fallback in `warranty-router` (sourced from the `jerm35/warranty` repo)
- **Token verification:** Worker calls `https://oauth2.googleapis.com/tokeninfo?id_token=...` and validates `aud == GOOGLE_CLIENT_ID` and `email_verified`. Restricts to `ALLOWED_CTL_DOMAIN` (`ctl.net`).

### NetSuite (read-only via deep links)

- **Purpose:** Allow CTL staff to jump from a spiff line straight to the underlying SO or invoice
- **URL pattern:** `https://11277173.app.netsuite.com/app/accounting/transactions/{salesord|custinvc}.nl?id=<internal_id>`
- **Authorization gate:** The worker constructs the URL only when the session email ends with `@ctl.net`. External users see plain text.
- **Internal IDs:** Sourced from `netsuite.invoices.invoice_id` and `netsuite.invoices.sales_order_id` (both INTEGER fields).

### NetSuite (write — via sales-etl)

- **Purpose:** Pull source data into BigQuery
- **Authentication:** OAuth1 (HMAC-SHA256) using consumer + token credentials
- **Client implementation:** [`/path/to/Claude/NetSuite/sales-etl/netsuite_client.py`](../../NetSuite/sales-etl/netsuite_client.py)
- **SuiteQL endpoint:** `https://<account>.suitetalk.api.netsuite.com/services/rest/query/v1/suiteql`

---

## Data Sources

### `ctl-repair-data.netsuite.invoices`

Header rows for each invoice. Used for invoice/SO numbers, dates, customer, ship-to city/state, and the primary `sales_rep`.

### `ctl-repair-data.netsuite.invoice_lines`

Per-line revenue detail. Quantity and amount come back NEGATIVE from NetSuite (credit side of JE); always `ABS()` for display. Used for warranty quantity, device unit price (`rate`), and Chromebook identification.

### `ctl-repair-data.netsuite.items`

Item master with `item_code` and `display_name`. Used for eligibility regex (`^WRCB.{4,}GC$` on `item_code`) and Chromebook filter (`display_name LIKE '%CHROMEBOOK%'`).

### `ctl-repair-data.netsuite.sales_team` (added 2026-05-18)

Per-transaction sales team breakouts pulled from NetSuite `transactionSalesTeam`. One row per (transaction, employee) with `role_name`, `contribution_pct` (stored as percent 0–100, normalized from NetSuite's decimal fraction), and `is_primary`. Holds rows for both Sales Orders and Invoices; downstream queries prefer the invoice's team and fall back to the SO's.

### `ctl-repair-data.fixably.netsuite_devices`

Per-serial device + warranty mapping (loaded by a separate ETL, not `sales-etl`). Keys: `sales_order_id` + `warranty_sku`. Used exclusively by the per-month CSV download to explode warranty lines into one row per device serial number.

---

## Security and Access Control

### Authentication

| Path | Mechanism |
|---|---|
| `@ctl.net` users | Google SSO via `ctl-auth.js`. Worker calls Google's `tokeninfo` endpoint to validate the ID token, then mints a 30-day session token stored in KV. |
| External users | Email/password (SHA-256 + per-user random salt). Self-signup via `Request Access` adds a row to the `PENDING` KV; an admin must approve before the user can sign in. |
| Admins | Same as above. The `isAdmin` flag is set on the user record in KV — no separate admin password. The first admin (`INITIAL_ADMIN_EMAIL`) is seeded automatically on cold start; further admins are promoted via the in-app Users tab. |

### Roles and Permissions

| Role | Who Has It | Capabilities |
|---|---|---|
| Admin | `isAdmin=true` in the `USERS` KV record | View dashboard data, approve/deny pending users, reset passwords, remove users, promote/demote admins, view all users |
| Approved user (CTL) | `status=approved` AND email ends with `@ctl.net` | View dashboard data; NetSuite deep links rendered as clickable |
| Approved user (external) | `status=approved` AND email does NOT end with `@ctl.net` | View dashboard data; NetSuite deep links rendered as plain text |
| Pending | Submitted `Request Access` but not yet approved | No access; sign-in returns "Invalid email or password" |

### Secrets and Credentials

| Secret | Location | Rotation Policy |
|---|---|---|
| `GCP_SERVICE_ACCOUNT_KEY` | Cloudflare Worker secret (encrypted at rest) | Rotate annually or upon staff departure. Use `gcloud iam service-accounts keys list` to audit existing keys. |
| `GOOGLE_CLIENT_ID` | Cloudflare Worker secret + embedded in public `ctl-auth.js` | Rotate only if the OAuth client itself is rotated; coordinate with all CTL warranty tools |
| `INITIAL_ADMIN_EMAIL` | Cloudflare Worker secret | Change if seeding a different first admin; existing admin records are not affected |
| NetSuite OAuth credentials | `sales-etl/.env` (local) and Cloud Run Job env vars (GCP) | Rotate per CTL's standard NetSuite integration policy |

### Data Handling

- **PII stored:** Customer name and ship-to city/state are pulled from BigQuery on every request (not persisted in the worker). User emails are stored in KV.
- **Data retention:** Worker KV records persist until manually removed. Session tokens auto-expire after 30 days.
- **Encryption at rest:** Cloudflare KV is encrypted at rest. BigQuery enforces encryption at rest by default.
- **Encryption in transit:** All endpoints are HTTPS-only. Cloudflare enforces TLS 1.2+ at the edge.

---

## Maintenance and Monitoring

### Logs

| Log type | Location | How to access |
|---|---|---|
| Worker request logs | Cloudflare dashboard → Workers & Pages → `getcoverspiff-worker` → Logs | Real-time tail via dashboard, or `npx wrangler@4 tail` |
| Router request logs | Cloudflare dashboard → Workers & Pages → `warranty-router` → Logs | Same as above |
| ETL job logs | Cloud Run → Jobs → `netsuite-sales-etl` → Logs | `gcloud run jobs executions list --job=netsuite-sales-etl --region=us-central1` then `gcloud run jobs executions describe <execution-id>` |
| BigQuery query history | BigQuery console → Query history | Filter by service account `bq-reader-sales` |

### Health Check

```bash
# Worker — should return {"ok": true, "version": "0.1.0"}
curl https://getcoverspiff-worker.jburnett-589.workers.dev/gcs/health

# Router — should return version + list of all 28+ prefixes including "getcoverspiff"
curl https://warranty.ctl.net/__router/health

# Frontend HTTP 200
curl -sI https://warranty.ctl.net/getcoverspiff/ | head -1
```

### Routine Maintenance

| Task | Frequency | How |
|---|---|---|
| Review pending access requests | Daily / on Slack ping | Open dashboard → Admin panel → Pending tab |
| Rotate BigQuery service-account key | Annually or on staff departure | `gcloud iam service-accounts keys create ... && npx wrangler@4 secret put GCP_SERVICE_ACCOUNT_KEY` |
| Confirm daily ETL ran clean | Daily | `gcloud run jobs executions list --job=netsuite-sales-etl --region=us-central1 --limit=2` |
| Update worker `wrangler` version | Quarterly | Pin to latest in `npx wrangler@4` calls; review changelog |
| Audit admin user list | Quarterly | Open dashboard → Admin panel → Users tab; demote anyone who should not be admin |

### Verifying the ETL Backfilled Correctly

```bash
# Should return today's loaded_at timestamp
bq --project_id=ctl-repair-data query --nouse_legacy_sql --format=csv \
  "SELECT MAX(loaded_at) AS last_loaded FROM \`ctl-repair-data.netsuite.sales_team\`"

# Sales team rows for any 2026 invoice — should be non-zero
bq --project_id=ctl-repair-data query --nouse_legacy_sql --format=csv \
  "SELECT COUNT(*) FROM \`ctl-repair-data.netsuite.sales_team\` WHERE transaction_type = 'CustInvc'"
```

---

## Admin Troubleshooting

### Dashboard returns "Failed to load data"

**Symptom:** User opens dashboard, sees the red error message after auth.
**Likely cause:** BigQuery query failure, expired SA key, or KV permissions issue.
**Fix:**

```bash
# Tail worker logs to see the underlying error
cd /path/to/getcoverspiff-worker
npx wrangler@4 tail

# Then have the user retry; the log will print the actual BigQuery 4xx/5xx
```

If the error is `403` or `401` on the BigQuery call, rotate the SA key.

### Sales Team row shows no reps

**Symptom:** A spiff line's Sales Team sub-row is missing.
**Likely cause:** The `netsuite.sales_team` table has no rows for that SO or invoice — either the ETL has not run since the SO was created, or the SO genuinely has no sales team configured in NetSuite.
**Fix:**

```bash
# 1. Confirm the data exists
bq --project_id=ctl-repair-data query --nouse_legacy_sql \
  "SELECT * FROM \`ctl-repair-data.netsuite.sales_team\` WHERE transaction_id = <so_id_or_invoice_id>"

# 2. If empty, force a backfill that includes that SO's modification date
cd /path/to/Claude/NetSuite/sales-etl
python3 main.py --incremental --since-days 30  # widen as needed

# 3. If still empty, the SO has no sales team in NetSuite — verify in the UI
```

### Daily ETL did not run

**Symptom:** `MAX(loaded_at)` from `sales_team` is older than 24h.
**Likely cause:** Cloud Scheduler trigger failed, or the Cloud Run Job is throwing.
**Fix:**

```bash
# Check recent executions
gcloud run jobs executions list --job=netsuite-sales-etl --region=us-central1 --limit=3

# Inspect the most recent
gcloud run jobs executions describe <execution-id> --region=us-central1

# Re-run manually
gcloud run jobs execute netsuite-sales-etl --region=us-central1 --project=ctl-repair-data
```

### User cannot sign in via Google SSO

**Symptom:** User clicks Google button, gets an error or nothing happens.
**Likely cause:** `GOOGLE_CLIENT_ID` mismatch between worker and `ctl-auth.js`, or browser is signed into a non-CTL Google account.
**Fix:** Have user verify they are signed into a `@ctl.net` Google account. If still failing:

```bash
# Confirm the secret value
cd /path/to/getcoverspiff-worker
npx wrangler@4 secret list   # shows only names, not values

# If a fresh value is needed:
echo "264235504831-ahd6lj613ff75tcpp51jqdifodbgeinm.apps.googleusercontent.com" \
  | npx wrangler@4 secret put GOOGLE_CLIENT_ID
```

### CSV download is missing serial numbers

**Symptom:** CSV opens, but Serial Number column is blank on many rows.
**Likely cause:** `fixably.netsuite_devices` has not been refreshed by its own ETL, OR the warranty was billed but the device fulfillment record is missing in NetSuite.
**Fix:** Check `fixably.netsuite_devices` for the affected SO. If empty, escalate to the team that owns the fixably ETL. If only some serials are missing, verify the fulfillment exists in NetSuite.

### Need to grant a new admin

```bash
# Option 1 (recommended): use the in-app Admin panel — Users tab → "Make Admin" next to the user
# Option 2 (if you need to do it before they sign in once): directly via wrangler
cd /path/to/getcoverspiff-worker
echo '{"email":"new.admin@ctl.net","passwordHash":"","salt":"","status":"approved","isAdmin":true,"createdAt":"'$(date -u +%FT%TZ)'","ssoOnly":true}' \
  | npx wrangler@4 kv key put --binding=USERS "u:new.admin@ctl.net" --remote
```

---

## Known Issues and Limitations

| Issue | Impact | Workaround | Planned Fix |
|---|---|---|---|
| `sales-etl/main.py` is not under version control | Medium — changes could be lost; no review trail | Back up changes manually; eventually create a dedicated repo | Move to `jerm35/sales-etl` (TBD) |
| `deploy/deploy.sh` fails on `gcloud run jobs update` when env vars are already set as secrets | Low — must fall back to image-only update | Use `gcloud run jobs update ... --image=...` instead of running deploy.sh's job-update step | Patch deploy.sh to detect existing secret bindings |
| Shipping address shows only city/state, no street | Low — sufficient for current spiff workflow | Click into NetSuite for full address | Extend `sales-etl` to pull `addr1`/`zip`/`country`/`addressee` from `transactionShippingAddress`; the JOIN already exists at `main.py:268` |
| Device filter relies on display_name LIKE `'%CHROMEBOOK%'` | Low — could miss a Chromebook with bad metadata, or over-match if someone names a non-Chromebook "Chromebook" | Update the SKU naming convention in NetSuite; current data is clean | Tighten to combine `display_name LIKE` with `item_type='InvtPart'` if false positives appear |
| No live websocket — data is one daily ETL behind | Low — accepted by users for monthly reporting | Click ↻ Refresh in top bar | None planned |
| `WRCB3025GC` is excluded by hardcoded string match | None — intentional | N/A | If more SKUs need exclusion, move to a configurable list in `queries.js` |

---

## Roadmap

| Item | Priority | Target |
|---|---|---|
| Extend `sales-etl` for full shipping street/zip | Low | When end users request more address granularity |
| Move `sales-etl/main.py` under version control | Medium | Next sales-etl change |
| Add unit-test coverage for the BigQuery SQL templates | Low | If a third feature requires SQL changes |
| Surface line dollar totals (qty × unit price) in the UI | Low | If reconciliation requires it |
| Bring up a sandbox worker on a `*.workers.dev` URL for QA | Low | If breaking changes become more frequent |

---

*Last verified: 2026-05-18 | Maintained by Jeremy Burnett (jburnett@ctl.net)*
