# Changelog

## v1.0 — 2026-04-21

- Production release
- Added `Process SSL Result` code node — evaluates ssl-checker.io API response and sets `needsAlert`, `alertLevel`, `daysLeft` fields (previously missing, causing silent filter failures)
- Renamed `Filter Alerts` → `Needs Alert?` for clarity
- Renamed `Loop Over Items` → `Loop Over Domains`
- Renamed `Loop Over Items1` → `Continue Domain Loop`
- Removed disabled `Send Slack Alert1` node (dead code)
- Simplified `Needs Alert?` to single condition: `needsAlert == true` (removed redundant cert_valid check — handled in Process SSL Result)
- Added 3 section-level sticky notes (Data Source, SSL Check Loop, Alert & Continue)
- Renamed workflow to `OneDrive - SSL Certificate Monitor V1.0`

## v0.1 — 2026-03-28

- Initial build
- Missing processing node between SSL API call and filter (silent failure)
- Disabled duplicate Slack node present as dead code
- Generic loop node names
