# n8n Payments Intelligence

> AI-powered payments signal monitor — stablecoin, CBDC, card network, and fintech

Built by [Arikina](https://www.arikina.com) | Chris Mahadeo

---

## What It Does

Runs on a weekday schedule, scanning 16+ curated RSS/API feeds across payments infrastructure, then routes each signal through Claude for classification and delivery.

```
RSS Feeds (stablecoin, CBDC, fintech) → n8n → Claude API
  → Signal Score: BUY / WATCH / NOISE
    → Notion database entry
    → Daily email brief (Gmail)
    → Slack alert for high-priority signals
```

## Signal Sources

- Circle (USDC), Tether, Paxos stablecoin updates
- Federal Reserve (FedNow, CBDC research)
- Visa, Mastercard, American Express network announcements
- CFPB, BIS, IMF regulatory signals
- Fintech press: TechCrunch Fintech, Payments Dive, PYMNTS
- Crypto infrastructure: Coindesk, The Block

## Claude Classification Schema

Each signal is scored on three axes:

| Dimension | Output |
|---|---|
| Signal type | REGULATORY / INFRASTRUCTURE / MARKET / PRODUCT |
| Priority | BUY / WATCH / NOISE |
| Impact area | STABLECOIN / CBDC / CARD_NETWORKS / CROSS_BORDER / CRYPTO |

## Setup

See [SETUP_GUIDE.md](SETUP_GUIDE.md) for n8n import instructions, credential configuration, and Notion schema.

### Requirements
- n8n (self-hosted or cloud)
- Anthropic API key (Claude)
- Notion API key + database
- Gmail OAuth (optional — for email brief delivery)

## Stack

`n8n` · `Claude API (Anthropic)` · `Notion API` · `Gmail API` · `RSS/Atom feeds`
