# Status - n8n-payments-intelligence

_Last reviewed: 2026-10-20_

Weekday n8n monitor scanning 16+ payments/stablecoin/CBDC feeds; Claude classifies each signal (BUY/WATCH/NOISE) and routes to Notion, email, and Slack.

## Components

| Component | State | Notes |
|---|---|---|
| payments_intelligence_workflow.json | Built | Feeds -> Claude classification -> Notion / Gmail / Slack |
| SETUP_GUIDE.md | Done | Import, credentials, Notion schema |
| Sample output | Missing | No example signal records in repo |

## Known gaps

- No sample classified signals or brief in the repo.

## Next up

See **M1 - Showcase** in [ROADMAP.md](ROADMAP.md).
