# Vendor Vuln Watch

A lightweight vulnerability monitor for a fixed watchlist of enterprise vendors — Tableau, NGINX, GitHub, FileCloud, ServiceNow, AWS, Jira, Confluence, Splunk, Microsoft Defender, and Tanium. Built to answer one question quickly: **has anything new and serious shown up for the software we run?**

Two ways to use it, in this repo:

| | What it is | When to use it |
|---|---|---|
| **`vendor-vuln-watch.html`** | A single-file, client-side dashboard you open in a browser | Manual, interactive checks — filter, sort, export |
| **`scripts/vendor_vuln_watch.py`** + **`.github/workflows/daily-vuln-watch.yml`** | A script that runs automatically on a schedule via GitHub Actions | Hands-off daily monitoring — opens a GitHub Issue when something's found |

No backend, no database, no accounts required beyond GitHub itself.

---

## What it checks

- **[NVD (National Vulnerability Database)](https://nvd.nist.gov/)** — queried per vendor with a tuned keyword search, filtered by publish date and severity
- **[CISA's Known Exploited Vulnerabilities (KEV) catalog](https://github.com/cisagov/kev-data)** — cross-referenced against every CVE found, flagging anything actively exploited in the wild (and separately, anything tied to known ransomware campaigns)
- **[Tenable](https://www.tenable.com/)** — a direct link per CVE to Tenable's public plugin lookup page, so you can check in one click whether your Nessus scanner already covers it

---

## 1. The web dashboard (`vendor-vuln-watch.html`)

### Opening it

Browsers won't let this file make live API calls if you just double-click it and open it as `file://...` — cross-origin requests get silently blocked. It needs to be served over `http(s)://`. Two options:

- **GitHub Pages** (permanent): `Settings → Pages → Source: Deploy from a branch → main / (root) → Save`. Once live, use the URL GitHub gives you.
- **htmlpreview.github.io** (instant, no setup):
  ```
  https://htmlpreview.github.io/?https://raw.githubusercontent.com/<your-username>/<repo>/main/vendor-vuln-watch.html
  ```

### Using it

1. Pick a lookback window (default: 3 days) and minimum severity (default: High+)
2. Check/uncheck vendors in the watchlist — click **edit** on any vendor to tune its NVD search keywords
3. Click **Pull Latest**
4. Sort the results table by clicking column headers; filter by severity or "KEV only"
5. Export as JSON, CSV, or Markdown
6. Use the **Quick Links** panel at the bottom for vendors that don't have a reliable public API (ServiceNow, Splunk, Tanium, Defender) — those need a manual check

### Notes

- An **NVD API key** (free, instant at [nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key)) raises your rate limit from 5 to 50 requests/30s. Without one, pulls are deliberately slow to stay under NVD's limit — expect ~1–2 minutes for the full watchlist.
- Config and last-pulled results are cached in your browser's `localStorage` — nothing leaves your machine except the API calls themselves.
- The KEV feed is pulled from CISA's own GitHub mirror (`cisagov/kev-data`), not `cisa.gov` directly — the official site doesn't send browser-CORS headers, the mirror does.

---

## 2. The daily automation (`scripts/vendor_vuln_watch.py`)

Runs the same NVD + KEV logic server-side on a GitHub Actions schedule (default: 13:00 UTC daily). No browser involved, so no CORS issues. When it finds anything at or above the severity threshold, it opens a **GitHub Issue** titled `[VULN WATCH] ...` — since you own the repo, GitHub automatically emails you about it through its normal notification system. No SMTP credentials, no API keys, no passwords stored anywhere; it authenticates with `GITHUB_TOKEN`, which GitHub Actions injects automatically.

### Setup

1. Make sure both files are in the repo at these exact paths:
   - `scripts/vendor_vuln_watch.py`
   - `.github/workflows/daily-vuln-watch.yml`
2. *(Optional)* Add an `NVD_API_KEY` repository secret (`Settings → Secrets and variables → Actions`) for faster, more reliable runs.
3. Confirm your GitHub notification email is one you check: `github.com/settings/emails`.
4. Test it manually: **Actions tab → "Daily Vendor Vuln Watch" → Run workflow**.

### Configuration (environment variables, set in the workflow file)

| Variable | Default | Meaning |
|---|---|---|
| `LOOKBACK_DAYS` | `3` | How many days back to search — should exceed the gap between scheduled runs with margin |
| `MIN_SEVERITY` | `HIGH` | Minimum severity to report: `ALL`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `SKIP_EMPTY` | `true` | If `true`, no issue is created on quiet days — keeps the Issues tab from filling up with empty reports |
| `NVD_API_KEY` | *(none)* | Optional; set as a repo secret, not hardcoded |

### Verifying it's actually working

A green checkmark on a run only means the script didn't crash — it doesn't confirm real data came back, since per-vendor failures are caught and logged rather than stopping the run. To check properly:

1. Click into a run → expand **"Run vendor vuln watch"** → read the log. You should see a per-vendor line like `NGINX ("nginx"): 16 matched.` and a final `Total: X unique CVEs found...` line.
2. Since `SKIP_EMPTY` hides quiet days, temporarily set it to `false` and re-run manually to force a visible test issue in the Issues tab.

---

## Keeping the two in sync

The web tool's vendor list is editable live in the browser UI and saved to `localStorage`. The Python script has its own hardcoded copy of the same list at the top of `vendor_vuln_watch.py`. **They don't share configuration** — if you tune a vendor's keywords in the web UI, update the script's `VENDORS` list to match if you want the daily automation to reflect the same change.

---

## Known limitations

- **ServiceNow, Splunk (advisory portal), Tanium, Microsoft Defender/MSRC** don't expose a public, CORS-friendly, keyword-searchable feed — coverage for these leans on NVD's keyword search picking up relevant CVEs by product name, plus the manual Quick Links panel as a backstop. Don't treat a "0 results" for these as a guaranteed all-clear without a manual glance at their bulletin pages occasionally.
- **AWS** rarely issues CVEs directly (most of its surface is service-side, not versioned software) — NVD hits under "Amazon Web Services" are mostly AWS's own open-source tooling (CLI, agents, SDKs), not the platform itself.
- Tenable plugin coverage is a **manual one-click check**, not automated — Tenable doesn't expose a public CORS-friendly API for this, and querying it in bulk in the background would also risk looking like scraping against their ToS.
