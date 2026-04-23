# 🔒 SSL Certificate Monitor

> **Daily SSL certificate health checks for all domains in an OneDrive Excel list — Slack alerts for anything expiring or invalid**

[![Status](https://img.shields.io/badge/Status-Active-green)]()
[![Category](https://img.shields.io/badge/Category-SecOps-red)]()
[![n8n](https://img.shields.io/badge/n8n-v1.121.3-orange)]()
[![License](https://img.shields.io/badge/License-MIT-lightgrey)]()

---

## 📋 Table of Contents
- [Overview](#-overview)
- [Workflow Architecture](#-workflow-architecture)
- [Alert Levels](#-alert-levels)
- [Node Table](#-node-table)
- [Credentials Required](#-credentials-required)
- [Quick Start](#-quick-start)
- [Customisation](#-customisation)
- [Error Handling](#-error-handling)
- [Technologies Used](#️-technologies-used)
- [Known Limitations](#-known-limitations)
- [Changelog](#-changelog)
- [Contact](#-contact)

---

## 🎯 Overview

Runs every morning at 9 AM. Reads a domain list from an Excel file stored in a **OneDrive Security folder**, checks each domain's SSL certificate via **ssl-checker.io**, evaluates the result against four alert thresholds, and posts a **Slack alert** for any domain that is invalid or expiring within 30 days. All results are logged to n8n execution history.

### Why This Matters
- **Expired SSL certs kill trust** — browsers block users, Google penalises rankings, and monitoring teams get woken up at 2 AM
- **Centralised domain list** — one Excel file in OneDrive means the list is maintained by the team, not locked inside the automation tool
- **Threshold-based alerting** — not every cert needs a page. NOTICE fires silently. WARNING and CRITICAL go to Slack. Alert fatigue is a real cost.
- **No infrastructure required** — ssl-checker.io handles the check; this workflow just manages the loop and routing

### Key Features
- ✅ Reads domain list from OneDrive Excel — no hardcoded domains
- ✅ Processes domains one at a time via batching loop
- ✅ Four-tier alert classification (OK / NOTICE / WARNING / CRITICAL)
- ✅ Only CRITICAL and WARNING generate Slack alerts
- ✅ Graceful error handling — one failed domain doesn't stop the run

---

## 🏗️ Workflow Architecture

```
Daily SSL Check (09:00)
         │
         ▼
Get items in Security folder (OneDrive)
         │
         ▼
Download all files
         │
         ▼
Read Excel from OneDrive
         │
         ▼
Extract Excel Data
         │
         ▼
Loop Over Domains ◄─────────────────────────┐
         │ (one domain per batch)             │
         ▼                                   │
Check SSL Certificate (ssl-checker.io)       │
         │                                   │
         ▼                                   │
Process SSL Result                           │
(sets needsAlert / alertLevel / daysLeft)    │
         │                                   │
         ▼                                   │
    Needs Alert?                             │
         │                                   │
    YES  │  NO                               │
         │   │                               │
         ▼   ▼                               │
  Send    Log Results ──► Continue Domain Loop┘
  Slack         │
  Alert ────────┘

Alert levels: CRITICAL (≤7d or invalid) | WARNING (≤30d) | NOTICE (≤60d) | OK (>60d)
Slack fires on: CRITICAL and WARNING only
```

---

## 🚨 Alert Levels

| Level | Condition | Slack Alert? |
|-------|-----------|-------------|
| 🚨 CRITICAL | Certificate invalid OR ≤ 7 days remaining | ✅ Yes |
| ⚠️ WARNING | 8–30 days remaining | ✅ Yes |
| ℹ️ NOTICE | 31–60 days remaining | ❌ Logged only |
| ✅ OK | > 60 days remaining | ❌ Logged only |

---

## 📊 Node Table

| Node | Type | Purpose |
|------|------|---------|
| Daily SSL Check | Schedule Trigger | Fires at 09:00 daily |
| Get items in Security folder | OneDrive | Lists files in the OneDrive Security folder |
| Download all files | OneDrive | Downloads each file from the folder listing |
| Read Excel from OneDrive | OneDrive | Downloads the specific Excel file by ID |
| Extract Excel Data | Extract From File | Parses Excel binary into row data |
| Loop Over Domains | Split In Batches | Iterates one domain at a time |
| Check SSL Certificate | HTTP Request | Calls ssl-checker.io API for certificate data |
| Process SSL Result | Code | Evaluates API response, sets alertLevel / needsAlert / daysLeft |
| Needs Alert? | If | Routes CRITICAL/WARNING to Slack, others to log only |
| Send Slack Alert | Slack | Posts alert to n8n-testing-space channel |
| Log Results | Code | Logs result to execution console |
| Continue Domain Loop | Split In Batches | Signals loop continuation to next domain |

---

## 🔑 Credentials Required

| Credential | Type | Used By |
|------------|------|---------|
| Microsoft OneDrive OAuth2 | n8n OneDrive credential | Get items, Download files, Read Excel |
| Slack OAuth2 | n8n Slack credential | Send Slack Alert |

---

## 🚀 Quick Start

### Prerequisites
- n8n instance (self-hosted or cloud)
- Microsoft 365 account with OneDrive access
- Slack workspace with bot permissions

### Setup Steps

**1. Prepare your domain list**

Create an Excel file with a `Domain` column:
```
Domain
example.com
api.example.com
shop.example.com
```
- No `https://`, no trailing slashes
- Column A, header must be exactly `Domain`

**2. Upload to OneDrive**

Upload the file to a folder named `Security` (or any name — you'll reference it by folder ID).

**3. Get your OneDrive folder ID**

Open the folder in OneDrive → copy the ID from the URL → paste into the `Get items in Security folder` node.

**4. Import the workflow**
```bash
# Via n8n CLI
n8n import:workflow --input=ssl-cert-monitor.json
```

**5. Configure credentials**

In n8n → Credentials, create:
- **Microsoft OneDrive OAuth2** with your M365 account
- **Slack OAuth2** with your bot token

**6. Set your Slack channel**

In `Send Slack Alert`, update `channelId` to your target security channel.

**7. Test on a single domain**

Pin a single-row Excel file, run manually, verify Slack output, then swap in the full list.

---

## ⚙️ Customisation

**Alert thresholds** — Edit the `if/else` block in `Process SSL Result`:
```javascript
if (daysLeft <= 7)  → CRITICAL
if (daysLeft <= 30) → WARNING
if (daysLeft <= 60) → NOTICE
else                → OK
```

**Schedule** — Change `triggerAtHour` in `Daily SSL Check`. Add `triggerAtMinute` for sub-hour precision.

**Slack message format** — Edit the `text` field in `Send Slack Alert`. Current fields: domain, alertLevel, alertMessage, daysLeft, certValid, checkDate.

**Rate limiting** — For large domain lists (>50), add a **Wait** node (1–2 seconds) inside the loop before `Check SSL Certificate` to stay within ssl-checker.io free tier limits.

---

## 🛡️ Error Handling

- `onError: continueRegularOutput` on OneDrive and HTTP Request nodes — one failed domain won't stop the run
- `Process SSL Result` handles missing or malformed API responses gracefully
- No workflow-level failure notification — add an Error Trigger node connected to Slack for production use

---

## 🛠️ Technologies Used

| Tool | Version | Purpose |
|------|---------|---------|
| n8n | v1.121.3 | Workflow orchestration and loop management |
| Microsoft OneDrive | Graph API v1.0 | Domain list storage and retrieval |
| ssl-checker.io | v1 | Certificate status API |
| Slack | API v2 | Alert delivery |

### Why These Tools?

**OneDrive** — The domain list lives where the team already manages files. No separate database or config file needed. The team can update it without touching n8n.

**ssl-checker.io** — Free, no authentication required per-check, simple JSON response. For enterprise use with hundreds of domains, consider replacing with a paid service or the `tls` module via a Code node.

**Slack** — Where alerts are already going. One channel, one place to look.

---

## ⚠️ Known Limitations

- **Logging is not queryable** — `console.log` output only appears in n8n execution history. A Google Sheets or Notion sub-workflow would make results searchable and trend-reportable.
- **No end-of-run summary** — alerts fire per domain. There is no consolidated "checked 47 domains, 2 warnings" message after the loop completes.
- **ssl-checker.io rate limits** — free tier has limits. Add a Wait node for lists >50 domains.
- **No email notification** — Slack only. Add a Send Email node if required.

---

## 📝 Changelog

See [changelog.md](changelog.md) for full version history.

**Latest:** v1.0 — 2026-04-21 — Added missing `Process SSL Result` node, renamed generic nodes, removed dead code, added section sticky notes.

---

## 📫 Contact

**Tony O'Connor**
- GitHub: [@digitaloconnor](https://github.com/digitaloconnor)
- n8n Instance: [automation.fioslabs.org](https://automation.fioslabs.org)

---

## 🔖 Tags

`#n8n` `#automation` `#ssl` `#certificates` `#secops` `#monitoring` `#onedrive` `#slack` `#security-automation` `#certificate-monitoring`

---

**⚡ Status**: Active | **🗂️ Category**: SecOps | **📅 Last Updated**: 2026-04-21
