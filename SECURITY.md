# Security

## Reporting Vulnerabilities

Report issues via [GitHub Issues](https://github.com/digitaloconnor/ssl-cert-monitor/issues) or [LinkedIn](https://linkedin.com/in/digitaloconnor).

Do not include API keys, tokens, credential values, or domain lists in any report.

## Data Handled

| Data | Source | Sent Externally? |
|------|--------|-----------------|
| Domain names | OneDrive Excel file | Sent to ssl-checker.io for certificate lookup |
| SSL certificate metadata (expiry date, validity) | ssl-checker.io | Stored in n8n execution history only |
| Alert messages (domain + cert status) | Internal | Posted to Slack channel |

**What this workflow does NOT touch:**
- No email content
- No personal data
- No credentials or passwords
- No internal system access beyond OneDrive file read

**ssl-checker.io:** Domain names are sent to a third-party API (ssl-checker.io). Review their privacy policy before using with sensitive internal domain names.

## Security Controls

- Microsoft OneDrive and Slack credentials stored in n8n credential store — not in workflow JSON
- OneDrive folder ID hardcoded in node (not a secret — it's a resource identifier, not a credential)
- No inbound webhooks — schedule-triggered only
- Execution logs contain domain names and cert status — ensure n8n instance access is appropriately restricted
- `.env` excluded from version control via `.gitignore`

## Network Exposure

**Inbound:** None. Schedule-triggered only.

**Outbound:**
- `graph.microsoft.com` — OneDrive file access
- `ssl-checker.io` — certificate status API
- `slack.com` — alert delivery
