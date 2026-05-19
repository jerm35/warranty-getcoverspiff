# GetCover Spiff — User Guide

> A monthly dashboard for sales reps and managers to see GetCover warranty sales eligible for spiff payouts, with downloadable per-device detail.

**Version:** 1.0 | **Status:** Production | **Last Updated:** 2026-05-18

---

## Table of Contents

- [What This Tool Does](#what-this-tool-does)
- [Getting Started](#getting-started)
- [Reading the Dashboard](#reading-the-dashboard)
- [Downloading the Monthly CSV](#downloading-the-monthly-csv)
- [Tips and Best Practices](#tips-and-best-practices)
- [FAQ](#faq)
- [Getting Help](#getting-help)

---

## What This Tool Does

GetCover Spiff shows every spiff-eligible warranty CTL has sold, grouped by month and by sales rep. Open a month, see who sold what, click into NetSuite for the underlying order, and download a CSV with every device serial number for commission processing.

**You can use GetCover Spiff to:**

- See monthly spiff-eligible warranty sales by sales rep
- Drill into the customer, ship-to, sales-team split, and device sold on each spiff line
- Click any Sales Order # or Invoice # to jump straight to NetSuite (CTL staff only)
- Download a monthly CSV with one row per device serial for spiff payout reconciliation

**What it does not do:**

- Calculate the spiff payout dollar amount — only shows the warranty quantity and the sales team breakout. Apply your commission rates separately.
- Include the warranty SKU `WRCB3025GC` — that one is explicitly not spiff-eligible.
- Show data prior to January 2026 — spiff reporting starts at the 2026 fiscal year.
- Show warranty sales that are not GetCover (must match the pattern `WRCB****GC`, e.g., `WRCB4001GC`, `WRCB4003GC`).

---

## Getting Started

### What You Will Need

- [ ] A modern web browser (Chrome, Edge, Firefox, Safari — current version)
- [ ] One of the following for sign-in:
  - A `@ctl.net` Google account (recommended — auto-approved), OR
  - An email and password registered through the in-app "Request Access" flow (requires admin approval)

### Accessing GetCover Spiff

1. Open `https://warranty.ctl.net/getcoverspiff/` in your browser.
2. Choose your sign-in path:
   - **CTL staff:** Click the **CTL Staff** tab → click the Google sign-in button → choose your `@ctl.net` account. You are auto-approved on first sign-in.
   - **External users:** Click **Request Access** → enter your email and a password (6+ characters) → submit. A CTL administrator will review and approve your request. Once approved, sign in via the **Sign In** tab.

You should see: a top bar with "CTL GetCover Spiff" on the left, three summary cards (Total Warranty Units, Total Invoices, Months), and a stack of collapsible month cards listed from most recent to oldest.

> NOTE: External users who have requested access but not yet been approved will see "Access request submitted! A CTL administrator will review it." Reach out to jburnett@ctl.net to expedite.

---

## Reading the Dashboard

### Summary Cards

The three cards at the top of the page roll up everything visible below:

- **Total Warranty Units** — sum of all spiff-eligible warranty quantities across all months
- **Total Invoices** — count of distinct invoices with at least one spiff-eligible warranty
- **Months** — number of months that have any spiff-eligible sales

### Month Cards

Each month appears as a collapsible card showing:

- The month name (e.g., "May 2026")
- A purple pill with `N invoices · M reps`
- A green count of total units sold that month
- A small download icon for the per-month CSV

Click the month header to expand. Click again to collapse.

### Inside an Expanded Month

Each month groups by sales rep. Each rep block has a header showing the rep's name and their subtotal units for the month, followed by a table.

**The main table has five columns:**

| Column | What it shows |
|---|---|
| Sales Order # | NetSuite SO number (e.g., `SO305113`). Dotted-underlined link if you are a CTL staff user — clicking opens the SO in NetSuite. |
| Invoice # | NetSuite invoice number (e.g., `INV415422`). Same link behavior as the SO. |
| Invoice Date | When the invoice was issued (M/D/YYYY) |
| Warranty Code | The spiff-eligible warranty SKU (e.g., `WRCB4001GC`) |
| Qty | Quantity of that warranty on the invoice |

### The Three Sub-Rows

Every spiff line is followed by three smaller sub-rows that provide the full context behind that warranty sale:

**1. CUSTOMER row** — Shows the customer name on the invoice and the ship-to city and state (e.g., "Acme School District · Ship to Portland, OR").

**2. SALES TEAM row** — One pill per rep on the invoice's sales team. Each pill shows the rep's name, their sales role (e.g., "Sales Rep"), their contribution percentage, and a "Primary" tag on the primary rep. The primary rep's pill has a colored border to make them easy to spot.

> **Example — single rep at 100%:**
> `[ Maureen Cooney  Sales Rep  100.0%  PRIMARY ]`
>
> **Example — split between two reps:**
> `[ Maureen Cooney  Sales Rep  70.0%  PRIMARY ]  [ John Smith  Sales Rep  30.0% ]`

**3. DEVICE row** — The Chromebook(s) sold on the same invoice, with the unit sell price. Format: item code, full display name, quantity, and unit price (e.g., `CBUS1400015  CTL Chromebook PLUS PX141GXT - 16/256 Clamshell Touchscreen  x65  $579.00`). If multiple Chromebook models were on the same invoice, each appears as its own entry.

### Theme and Appearance

- The **sun icon** in the top bar cycles through four themes (Light, Light Dim, Dark Dim, Dark).
- The **circle icon** opens the appearance picker for accent color, theme, and font size. Choices persist across sessions.

---

## Downloading the Monthly CSV

1. Locate the month you want to export.
2. Click the **download icon** on the right side of that month's header (next to the unit count).
3. Your browser downloads a file named `getcoverspiff-YYYY-MM.csv` (e.g., `getcoverspiff-2026-05.csv`).
4. Open the file in Excel, Google Sheets, or your spreadsheet tool of choice.

### What's in the CSV

**One row per device serial number.** A warranty line with `Qty 65` expands to 65 rows, one for each physical device that warranty applies to. The CSV has 15 columns:

| # | Column | Notes |
|---|---|---|
| 1 | Sales Rep | Primary rep on the invoice |
| 2 | Sales Order # | |
| 3 | Invoice # | |
| 4 | Invoice Date | YYYY-MM-DD |
| 5 | Warranty Code | e.g., `WRCB4001GC` |
| 6 | Customer | |
| 7 | Ship City | |
| 8 | Ship State | |
| 9 | Sales Team | Pipe-separated reps, e.g., `Maureen Cooney (Sales Rep) — 100.0% *` (`*` marks primary) |
| 10 | Serial Number | Per-device serial — different for every row |
| 11 | Device SKU | e.g., `CBUS1400015` |
| 12 | Device Name | Full product display name |
| 13 | Device Unit Price | Unit sell price on the invoice (decimal, no `$`) |
| 14 | MAC Address | If recorded |
| 15 | Asset Tag | If recorded |

> NOTE: If a spiff line has not been fulfilled yet (the warranty was billed but no device serials are recorded), it still appears once in the CSV with the Serial Number and device columns blank — so you do not silently miss anything.

### Spreadsheet Tips

- Use **pivot tables** to roll up by Sales Rep or by Device SKU for payout summaries.
- Use **filters** on the Sales Team column to find split orders (look for rows where the cell contains `|`).
- The Device Unit Price is plain decimal (e.g., `579.00`) so it sums correctly without text-to-number conversion.

---

## Tips and Best Practices

- **The data is one day behind.** Behind the scenes, NetSuite data syncs to the dashboard once per day. If you just billed an invoice, expect to see it tomorrow.
- **Refresh after long sessions.** The data on screen is loaded when the page first opens. Hit the **↻ Refresh** button in the top bar to pull a fresh copy.
- **Sales team split is the truth.** Always check the Sales Team sub-row, not just the "Sales Rep" column in the table. The table's Sales Rep is the *primary* rep on the invoice header; the Sales Team sub-row is the actual breakdown.
- **NetSuite links open in a new tab.** Hold ⌘ (Mac) or Ctrl (Windows) + click to force a new tab if you want to keep the dashboard open.
- **CSV per month, not per quarter.** Each download is one month. To work across multiple months, download each separately and combine in your spreadsheet.
- **The Devices row is Chromebooks only.** Software licenses, accessories, and services are not shown on the Device sub-row — only items whose name contains "Chromebook."

---

## FAQ

**Q: I am not seeing data for a recent invoice. Why?**
A: NetSuite data syncs once per day. If the invoice was billed today, wait until tomorrow morning. If it has been more than 48 hours, contact jburnett@ctl.net.

**Q: I think a warranty is missing. How do I check?**
A: Confirm the warranty SKU matches the pattern `WRCB****GC` (four or more characters between WRCB and GC), is not `WRCB3025GC` (excluded), and the invoice date is on or after January 1, 2026. Anything outside those rules is filtered out by design.

**Q: The Sales Team row only shows one rep at 100% on every invoice. Is that right?**
A: For most CTL orders, yes — a single primary rep at 100% is the default in NetSuite. Splits show up only when the sales team was explicitly configured on the SO or invoice. If you expect a split and do not see one, check the Sales Team sub-list in NetSuite on the SO — that is the source of truth.

**Q: Can I see the order subtotal or full address?**
A: Not in the current version. The dashboard shows ship-to city and state, not street or zip. The Device sub-row shows unit price; total per line is qty × unit price. If you need more, download the CSV and use a spreadsheet, or open the invoice in NetSuite via the link.

**Q: I am external — how do I request access?**
A: On the sign-in screen, click the **Request Access** tab, enter your email and a password, and submit. A CTL admin (typically jburnett@ctl.net) will be notified and can approve from inside the dashboard. You will see "Access request submitted!" when it goes through.

**Q: I forgot my password.**
A: Contact jburnett@ctl.net. An admin can reset your password from the Admin panel and send you a temporary password.

**Q: I am CTL staff but the Google sign-in is not working.**
A: Make sure you are signed into your `@ctl.net` Google account in the browser. If you have multiple Google accounts, the picker should appear when you click the Google button. If it still fails, sign out of all Google accounts and try again.

**Q: What if I see "Failed to load data"?**
A: First try **↻ Refresh** in the top bar. If the error persists, screenshot it and email jburnett@ctl.net with the time and the month you were trying to view.

---

## Getting Help

If something is not working or your question is not answered here:

| Channel | When to use | Contact |
|---|---|---|
| Email | Access requests, bugs, data questions | jburnett@ctl.net |
| In-app admin | Password resets, account approval | An existing admin (typically jburnett@ctl.net) |

**When reporting a problem, include:**

1. What you were trying to do (e.g., "download the CSV for May 2026")
2. What you expected to happen
3. What actually happened (screenshot the error if possible — include the URL and the time)
