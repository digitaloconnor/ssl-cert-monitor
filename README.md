# SSL Certificate Monitor

Daily n8n automation that reads a list of domains from an OneDrive Excel file, checks each SSL certificate via ssl-checker.io, and sends a Slack alert for any certificate that is invalid or expiring within 30 days.

---

## What It Does

Runs at 9 AM every day. Pulls a domain list from an Excel file stored in an OneDrive Security folder, iterates over each domain, calls the ssl-checker.io API to retrieve certificate status, evaluates the result against four alert thresholds, and posts a Slack message for any CRITICAL or WARNING finding. All results are logged to n8n execution history.

---

## Workflow Diagram

```
Daily SSL Check
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
Loop Over Domains
      │ (one domain per iteration)
      ▼
Check SSL Certificate (ssl-checker.io)
      │
      ▼
Process SSL Result
(sets needsAlert, alertLevel, daysLeft)
      │
      ▼
Needs Alert?
      ├── YES → Send Slack Alert → Log Results
      └── NO  → Log Results
                     │
                     ▼
              Continue Domain Loop → (back to Loop Over Domains)
```

---

## Alert Levels

| Level | Condition | Action |
|-------|-----------|--------|
| CRITICAL | Certificate invalid OR ≤ 7 days remaining | Slack alert |
| WARNING | 8–30 days remaining | Slack alert |
| NOTICE | 31–60 days remaining | No alert (logged only) |
| OK | > 60 days remaining | No alert (logged only) |

---

## Node Table

| Node | Type | Purpose |
|------|------|---------|
| Daily SSL Check | Schedule Trigger | Fires at 09:00 daily |
| Get items in Security folder | OneDrive | Lists files in the OneDrive Security folder |
| Download all files | OneDrive | Downloads each file from the folder |
| Read Excel from OneDrive | OneDrive | Downloads the specific Excel file by ID |
| Extract Excel Data | Extract From File | Parses Excel binary into row data |
| Loop Over Domains | Split In Batches | Iterates one domain at a time |
| Check SSL Certificate | HTTP Request | Calls ssl-checker.io API for the domain |
| Process SSL Result | Code | Evaluates API response, sets alertLevel/needsAlert/daysLeft |
| Needs Alert? | If | Routes CRITICAL/WARNING to Slack, others to log only |
| Send Slack Alert | Slack | Posts alert message to n8n-testing-space channel |
| Log Results | Code | Logs result to execution console (console.log) |
| Continue Domain Loop | Split In Batches | Signals loop continuation to next domain |

---

## Credentials Required

| Credential | Type | Used By |
|------------|------|---------|
| Microsoft OneDrive OAuth2 | n8n OneDrive credential | Get items in Security folder, Download all files, Read Excel from OneDrive |
| Slack OAuth2 | n8n Slack credential | Send Slack Alert |

---

## Setup

1. Create an Excel file with a `Domain` column (column A header must be `Domain`)
2. Add domains as plain values — no `https://`, no trailing slashes (e.g. `example.com`)
3. Upload the Excel file to your OneDrive Security folder
4. Get the OneDrive folder ID from the folder URL and update `Get items in Security folder`
5. Copy `.env.example` to `.env` and fill in credentials
6. Set up Microsoft OneDrive OAuth2 and Slack OAuth2 credentials in n8n
7. In `Send Slack Alert`, set your target Slack channel
8. Import and test manually on one domain before activating

---

## Customisation

**Alert thresholds** — Edit the `if/else` block in `Process SSL Result`. Current thresholds: CRITICAL ≤7 days, WARNING ≤30, NOTICE ≤60.

**Multiple Excel files** — The workflow downloads all files in the Security folder. If you have multiple Excel files, each will be processed. Ensure all use the same `Domain` column format.

**Slack channel** — Update the `channelId` in `Send Slack Alert`. Currently wired to `n8n-testing-space`.

---

## Error Handling

- `onError: continueRegularOutput` on OneDrive and HTTP Request nodes — a single failed domain won't stop the loop
- `Process SSL Result` handles missing/malformed API responses gracefully
- No escalation path if the entire workflow fails — consider adding an Error Trigger node connected to a Slack alert

---

## Known Limitations

- **Logging is not queryable** — `Log Results` uses `console.log`, which only appears in n8n execution history. A Google Sheets or Notion logging sub-workflow would make results searchable and reportable.
- **No daily summary** — alerts fire per domain. There's no end-of-run summary message. Consider adding a summary node after the loop completes.
- **ssl-checker.io rate limits** — free tier has limits. For large domain lists (>50), add a Wait node inside the loop.

---

## Related Workflows

<!-- add links -->
