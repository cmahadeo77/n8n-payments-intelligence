# n8n Payments & Market Signals Intelligence Workflow — Setup Guide

## Overview

This workflow runs every day at **6:30 AM** (before markets open), scrapes X (Twitter)
for posts from key payments/fintech accounts and topic keywords, filters and scores the
results, sends them to Claude (claude-sonnet-4-6) for signal analysis, and delivers a
structured report to both **#payments-signals** in Slack and a **Notion database page**.

### Accounts monitored
`@PaymentsSource`, `@PYMNTS`, `@VisaNews`, `@Stripe`, `@Mastercard`,
`@federalreserve`, `@BIS_org`

### Keywords monitored
CBDC, stablecoin, interchange, fintech, Fed, interest rates, crypto regulation,
payment rails, real-time payments, FedNow, open banking

### Claude prompt (as configured in the workflow)
> "You are a payments and financial markets analyst. From the following X posts:
> 1. Flag any central bank or Fed signals
> 2. Identify fintech/payments product moves (Visa, Mastercard, Stripe, etc.)
> 3. Highlight stablecoin or crypto regulatory news
> 4. Note any market-moving macro signals
> Rate each signal: Buy signal / Watch / Noise."

---

## Workflow Architecture

```
Schedule Trigger (6:30AM Daily)
        |
        v
Apify - Start X Scraper Run   (POST /runs to apidojo~tweet-scraper)
        |
        v
Wait 45s for Scraper
        |
        v
Apify - Fetch Scraped Results  (GET /runs/{id}/dataset/items)
        |
        v
Clean & Filter Posts           (Code node: dedup, spam filter, score, categorize)
        |
        v
If No Posts Found
    |                           |
    | (true — no results)       | (false — posts found)
    v                           v
Slack - No Posts           Claude API - Analyze Posts
Notification               (POST https://api.anthropic.com/v1/messages)
                                |
                                v
                          Extract Claude Response
                          (Code node: parse, detect buy signals)
                               / \
                              /   \
                             v     v
             Slack - Post to         Notion - Create
             #payments-signals        Signal Page
```

---

## Step 1: Import the Workflow

1. Open your n8n instance.
2. Go to **Workflows** → **Import from File**.
3. Select `payments_intelligence_workflow.json`.
4. Click **Import**.

---

## Step 2: Configure All Credentials

You need four credentials. Create each one under **Settings → Credentials → New**.

---

### Credential 1: Apify (HTTP Header Auth)

**Credential type:** `Header Auth`
**Name:** `Apify Header Auth`

| Field | Value |
|-------|-------|
| Name  | `Authorization` |
| Value | `Bearer YOUR_APIFY_API_TOKEN` |

**How to get your Apify API token:**
1. Sign up or log in at https://console.apify.com
2. Go to **Settings → Integrations → API tokens**
3. Copy your **Personal API token**

**IMPORTANT:** After importing the workflow, open both Apify HTTP Request nodes and
replace the literal string `YOUR_APIFY_API_TOKEN` in the query parameter `token` with
your actual Apify API token. The two nodes are:
- `Apify - Start X Scraper Run`
- `Apify - Fetch Scraped Results`

---

### Credential 2: Claude API (HTTP Header Auth)

**Credential type:** `Header Auth`
**Name:** `Claude API Auth`

| Field | Value |
|-------|-------|
| Name  | `x-api-key` |
| Value | `YOUR_ANTHROPIC_API_KEY` |

**How to get your Anthropic API key:**
1. Go to https://console.anthropic.com
2. Navigate to **API Keys** → **Create Key**
3. Copy the key (it starts with `sk-ant-`)

After importing the workflow, open the `Claude API - Analyze Posts` node and replace
`YOUR_ANTHROPIC_API_KEY` in the header parameters with your actual key.

The node calls:
```
POST https://api.anthropic.com/v1/messages
Headers:
  x-api-key: YOUR_KEY
  anthropic-version: 2023-06-01
  content-type: application/json
Model: claude-sonnet-4-6
Max tokens: 4096
```

---

### Credential 3: Slack

**Credential type:** `Slack API`
**Name:** `Slack API`

**How to create a Slack app and get a token:**
1. Go to https://api.slack.com/apps → **Create New App** → **From Scratch**
2. Name it something like `n8n Payments Bot`
3. Under **OAuth & Permissions**, add these **Bot Token Scopes**:
   - `chat:write`
   - `chat:write.public` (lets the bot post to channels without being invited)
4. Click **Install to Workspace** and copy the **Bot User OAuth Token** (starts with `xoxb-`)
5. Paste that token into the n8n Slack credential.

**Channel setup:**
- Make sure `#payments-signals` exists in your Slack workspace.
- If using `chat:write` (not `chat:write.public`), invite the bot: `/invite @n8n Payments Bot`

---

### Credential 4: Notion

**Credential type:** `Notion API`
**Name:** `Notion API`

**How to create a Notion integration:**
1. Go to https://www.notion.so/my-integrations → **New Integration**
2. Name it (e.g., `n8n Payments Intelligence`), select your workspace
3. Under **Capabilities**, enable: **Insert content**, **Read content**, **Update content**
4. Copy the **Internal Integration Token** (starts with `secret_`)
5. Paste into the n8n Notion credential.

**Connect your database:**
1. Create (or open) a Notion database where signal reports will be stored.
2. Ensure it has these properties:

   | Property name | Type |
   |--------------|------|
   | `Name` | Title (default) |
   | `Date` | Date |
   | `Buy Signal` | Checkbox |
   | `Urgent` | Checkbox |
   | `Model` | Text (Rich Text) |
   | `Tags` | Multi-select |

3. Open the database → click `...` (top right) → **Add connections** → select your integration.
4. Copy the **Database ID** from the URL:
   `https://notion.so/YOUR_WORKSPACE/DATABASE_ID?v=...`
   The Database ID is the 32-character string before `?v=`.
5. In the `Notion - Create Signal Page` node, replace `YOUR_NOTION_DATABASE_ID`
   with your actual database ID.

---

## Step 3: Configure the Apify Actor

**Actor used:** `apidojo/tweet-scraper`
- Apify Store link: https://apify.com/apidojo/tweet-scraper
- Actor ID in the API URL: `apidojo~tweet-scraper`

**How to verify the actor is available:**
1. Go to https://apify.com/apidojo/tweet-scraper
2. Click **Try for free** to add it to your account (no separate purchase required for
   API-based runs on a paid Apify plan)

**Actor input already configured in the workflow:**
```json
{
  "searchTerms": [
    "from:PaymentsSource",
    "from:PYMNTS",
    "from:VisaNews",
    "from:Stripe",
    "from:Mastercard",
    "from:federalreserve",
    "from:BIS_org",
    "CBDC",
    "stablecoin",
    "interchange fee",
    "fintech regulation",
    "Federal Reserve payments",
    "interest rates decision",
    "crypto regulation payments",
    "payment rails",
    "real-time payments",
    "FedNow",
    "open banking"
  ],
  "maxItems": 150,
  "tweetLanguage": "en",
  "start": "<yesterday's date — auto-calculated>",
  "end": "<today's date — auto-calculated>",
  "addUserInfo": true
}
```

> **NOTE:** X (Twitter) scraping via Apify relies on unofficial APIs. If
> `apidojo~tweet-scraper` becomes unavailable, check the Apify Store for alternatives:
> https://apify.com/store?search=twitter+scraper
> Common alternatives: `quacker/twitter-scraper`, `microworlds/twitter-scraper`

---

## Step 4: Adjust the Wait Time (Optional)

The workflow waits **45 seconds** between starting the Apify run and fetching results.
For 150-item scrapes across many search terms, 45–60 seconds is usually sufficient.

- Open the `Wait 45s for Scraper` node
- Increase to `60` or `90` seconds if you see empty results on first fetch

---

## Step 5: Verify Node Connections After Import

After import, confirm these edges exist on the canvas:

| From | To | Branch |
|------|----|--------|
| Schedule Trigger (6:30AM Daily) | Apify - Start X Scraper Run | — |
| Apify - Start X Scraper Run | Wait 45s for Scraper | — |
| Wait 45s for Scraper | Apify - Fetch Scraped Results | — |
| Apify - Fetch Scraped Results | Clean & Filter Posts | — |
| Clean & Filter Posts | If No Posts Found | — |
| If No Posts Found | Slack - No Posts Notification | true (left/output 0) |
| If No Posts Found | Claude API - Analyze Posts | false (right/output 1) |
| Claude API - Analyze Posts | Extract Claude Response | — |
| Extract Claude Response | Slack - Post to #payments-signals | — |
| Extract Claude Response | Notion - Create Signal Page | — |

---

## Step 6: Test the Workflow

1. **Disable the schedule trigger** (toggle it off) to prevent accidental automatic runs.
2. Click the `Schedule Trigger` node → **Test step** to manually fire it.
3. Step through each node using **Execute node** to verify output at each stage.
4. Check:
   - `Apify - Fetch Scraped Results` returns an array of tweet objects
   - `Clean & Filter Posts` outputs a non-empty `formattedPosts` field
   - `Claude API - Analyze Posts` returns a response with `content[0].text`
   - `Extract Claude Response` populates `analysisText`, `hasBuySignal`, `slackBody`
   - Slack receives the message in `#payments-signals`
   - Notion creates a new page in your database

---

## Step 7: Activate the Workflow

1. Once testing passes, click **Activate** (toggle in the top-right of the editor).
2. The workflow will run automatically every day at **6:30 AM** in your n8n server's
   local timezone.

**To adjust the time or timezone:**
- Open `Schedule Trigger (6:30AM Daily)` node
- Current cron expression: `30 6 * * *` = 6:30 AM daily
- For 6:30 AM EST (UTC-5): use `30 11 * * *`
- For 6:30 AM PST (UTC-8): use `30 14 * * *`
- Alternatively, set your n8n server timezone to your local timezone under
  **Settings → General → Timezone**

---

## How the Signal Rating Works

The `Extract Claude Response` code node detects two flags automatically:

| Flag | Detection | Slack indicator |
|------|-----------|-----------------|
| `hasBuySignal` | Claude's response contains "Buy signal" | Green circle in header |
| `hasUrgent` | Claude's response contains "urgent" or "[URGENT]" | Red circle in header |
| Neither | Watch/Noise signals only | Yellow circle in header |

These flags also populate the `Buy Signal` and `Urgent` checkboxes in Notion.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Apify returns empty results | Increase wait to 60–90s; check Apify dashboard for run errors |
| Claude returns 401 | Verify your API key in the `Claude API Auth` credential |
| Slack message fails | Confirm bot is in `#payments-signals`; check `chat:write` scope |
| Notion page not created | Confirm database ID is correct; confirm integration is connected to the database |
| `noPosts: true` every run | X API restrictions may be blocking scraping; try fewer search terms or a different Apify actor |
| Claude response truncated in Slack | Intentional — Slack limits to 2900 chars; full report is always in Notion |
| Run ID missing on fetch step | Increase wait time; the Apify run may not have started returning an ID yet |

---

## Cost Estimates (approximate, daily)

| Service | Est. Daily Cost |
|---------|----------------|
| Apify (tweet-scraper, 150 items, 18 search terms) | ~$0.05–0.20 per run |
| Claude claude-sonnet-4-6 (~5K input tokens, ~1.5K output) | ~$0.02–0.06 per run |
| Slack | Free (standard plan) |
| Notion | Free (personal plan) |
| **Total** | **~$0.07–0.26/day (~$2–8/month)** |

---

## Files in This Package

- `payments_intelligence_workflow.json` — Import this into n8n
- `SETUP_GUIDE.md` — This file
