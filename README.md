# 🤖 AI-Powered Slack Support Bot

An intelligent, fully automated customer support system built with **n8n**, **Groq AI**, **Slack**, and **Airtable**. When a user sends a support message in Slack, the bot classifies it using AI, creates a tracked ticket in Airtable, sends a personalised auto-reply, escalates unresolved tickets after 24 hours, and delivers a weekly summary report — all with zero human intervention.

> Built as a portfolio project demonstrating real-world workflow automation, AI integration, multi-platform API orchestration, and self-hosted infrastructure using Docker.

---

## 📸 Demo

> _Send a message in Slack → AI classifies it → Ticket created in Airtable → Auto-reply sent_

![Demo GIF](./assets/demo.gif)

<!-- Record a screen capture showing the full flow and save as assets/demo.gif -->

---

## 🧠 Why I Built This

Modern support teams waste hours manually triaging tickets. This project automates the entire first-response layer:
- No human needed to read, classify, or respond to incoming messages
- Every ticket is tracked with priority and category from the moment it arrives
- Unresolved tickets automatically escalate so nothing falls through the cracks
- Weekly reports give visibility into support volume and trends

**Technologies were chosen for the following reasons:**
- **n8n** — open source, self-hostable, no per-execution pricing, visual workflow builder
- **Groq** — fastest free AI inference API available (LPU-based), no credit card required
- **Slack** — professional messaging platform with a robust bot/event API
- **Airtable** — visual database perfect for ticket tracking, free tier sufficient for portfolio
- **Docker** — portable, reproducible deployment that runs identically on any machine

---

## ⚙️ System Overview

This project contains **3 separate automated workflows:**

### Workflow 1 — Support Bot (Main)
Handles every incoming support message in real time.

### Workflow 2 — Escalation Engine
Runs every hour and catches tickets that haven't been resolved in 24+ hours.

### Workflow 3 — Weekly Summary Report
Runs every Monday at 9am and sends a full stats digest to the support channel.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        WORKFLOW 1 — MAIN                        │
│                                                                 │
│  Slack #support  ──►  n8n Webhook  ──►  Code Node (extract)    │
│                                              │                  │
│                                    HTTP Request (Groq API)      │
│                                              │                  │
│                                    Code Node (parse JSON)       │
│                                              │                  │
│                              ┌───────────────┴──────────────┐  │
│                              ▼                              ▼  │
│                    Airtable (create ticket)    Slack (auto-reply)│
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    WORKFLOW 2 — ESCALATION                      │
│                                                                 │
│  Schedule (every hour)                                          │
│       │                                                         │
│  Airtable Search (Open tickets older than 24h)                  │
│       │                                                         │
│  IF node (any found?)                                           │
│       │                                                         │
│  ┌────┴────────────────────────────────┐                        │
│  ▼                                    ▼                        │
│  Code (aggregate) → Slack alert    Airtable (mark Resolved)    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  WORKFLOW 3 — WEEKLY SUMMARY                    │
│                                                                 │
│  Schedule (Monday 9am)                                          │
│       │                                                         │
│  Airtable Search (last 7 days tickets)                          │
│       │                                                         │
│  Code Node (count by category, priority, status)                │
│       │                                                         │
│  Slack (send formatted weekly report)                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Full Tech Stack

| Technology | What it does | Why I chose it |
|---|---|---|
| **n8n** (self-hosted) | Workflow automation engine — connects all services | Open source, free, self-hostable, no execution limits |
| **Docker** | Runs n8n in a container locally | Portable, reproducible, industry standard |
| **ngrok** | Creates a public HTTPS tunnel to local n8n | Required for Slack webhooks to reach localhost |
| **Slack API** (Bot OAuth v2) | User-facing messaging interface | Most widely used team communication tool, free bot tier |
| **Slack Event Subscriptions** | Pushes message events to n8n webhook | Allows real-time message processing without polling |
| **Groq API** (LLaMA 3.1 8B Instant) | AI classification of support messages | Fastest free inference API, no credit card, 14,400 req/day |
| **Airtable** (REST API) | Stores and tracks all support tickets | Visual database, great for demos, free tier = 1000 records |
| **JavaScript** | Custom logic in n8n Code nodes | Used for JSON parsing, aggregation, data transformation |

---

## 🤖 AI Classification — How It Works

Every message is sent to **Groq's LLaMA 3.1 8B Instant** model with this system prompt:

```
You are a support classifier. Return ONLY valid JSON with keys:
- category (billing/bug/general)
- summary (one sentence)
- priority (high/medium/low)
No markdown, no backticks, just raw JSON.
```

The model returns structured JSON like:
```json
{
  "category": "billing",
  "summary": "User is unable to complete a payment transaction.",
  "priority": "high"
}
```

**Why LLaMA 3.1 8B on Groq?**
- Groq's LPU (Language Processing Unit) delivers sub-200ms inference
- Free tier: 14,400 requests/day — more than enough for a support bot
- No credit card required
- LLaMA 3.1 8B is accurate enough for classification tasks at this scale

---

## 🗃️ Airtable Schema

**Base:** SupportBot
**Table:** Tickets

| Field | Type | Populated by |
|---|---|---|
| SlackUserID | Single line text | Slack event payload (`event.user`) |
| Message | Long text | Slack event payload (`event.text`) |
| Category | Single select (billing/bug/general) | Groq AI classification |
| Summary | Long text | Groq AI generated |
| Priority | Single select (high/medium/low) | Groq AI classification |
| Status | Single select (Open/Resolved) | Set to Open on create; Resolved on escalation |
| CreatedAt | Date & Time | Slack event timestamp (converted from Unix) |
| ChannelID | Single line text | Slack event payload (`event.channel`) |

---

## 📋 Workflow Details

### Workflow 1 — Support Bot (Main)

**Trigger:** Slack Event API sends a POST to n8n webhook when a message is posted in #support

**Node breakdown:**

| Node | Type | What it does |
|---|---|---|
| Webhook | Webhook | Receives Slack events, handles URL verification challenge |
| Code (extract) | JavaScript | Extracts message text, user ID, channel ID. Ignores bot messages to prevent infinite loops |
| HTTP Request | POST to Groq API | Sends message to LLaMA 3.1 8B for classification |
| Code (parse) | JavaScript | Parses Groq JSON response, combines with Slack metadata |
| Airtable | Create record | Creates ticket with all fields mapped |
| Slack | Send message | Sends personalised auto-reply with ticket ID and SLA time |

**Auto-reply logic:**
- High priority → "Our team will respond within 2 hours"
- Medium/Low priority → "We will respond within 24 hours"

**Bot loop prevention:**
```javascript
if (event.bot_id || event.subtype) {
  return [{ json: { skip: true } }];
}
```

**Slack challenge handler:**
```javascript
if (body.type === 'url_verification') {
  return [{ json: { challenge: body.challenge } }];
}
```

---

### Workflow 2 — Escalation Engine

**Trigger:** Schedule node — runs every hour

**Node breakdown:**

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | Every 1 hour | Fires the workflow hourly |
| Airtable Search | Search records | Finds tickets where Status=Open AND CreatedAt > 24h ago |
| IF node | Condition | Checks if any overdue tickets exist (`length > 0`) |
| Code (aggregate) | JavaScript | Combines all tickets into one formatted message |
| Slack | Send message | Sends one escalation alert listing all overdue tickets |
| Airtable Update | Update record | Marks each overdue ticket as Resolved |

**Airtable filter formula:**
```
AND(Status = 'Open', IS_BEFORE(CreatedAt, DATEADD(NOW(), -24, 'hours')))
```

**Why two branches from IF?**
The Code/Slack branch aggregates all tickets into one Slack message so you don't get spammed. The Airtable Update branch runs once per ticket to update each record individually. Both connect from the IF true output.

---

### Workflow 3 — Weekly Summary Report

**Trigger:** Schedule node — every Monday at 9:00am

**Node breakdown:**

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | Weekly (Monday 9am) | Fires the workflow weekly |
| Airtable Search | Search records | Fetches all tickets from the last 7 days |
| Code (aggregate) | JavaScript | Counts tickets by category, priority, and status |
| Slack | Send message | Posts formatted weekly report to #support channel |

**Airtable filter formula:**
```
IS_AFTER(CreatedAt, DATEADD(NOW(), -7, 'days'))
```

**Stats calculated:**
- Total tickets
- By category: billing / bug / general
- By priority: high / medium / low
- By status: resolved / open

---

## 🚀 How to Run Locally

### Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [ngrok](https://ngrok.com) free account
- Slack workspace + bot app created at api.slack.com/apps
- Groq API key (free at console.groq.com)
- Airtable account + SupportBot base set up

### Step 1 — Start n8n
```bash
# Windows
docker run -it --rm --name n8n -p 5678:5678 -v %USERPROFILE%\.n8n:/home/node/.n8n -e WEBHOOK_URL=https://YOUR_NGROK_URL -e N8N_ENCRYPTION_KEY=your32charstring n8nio/n8n

# Mac/Linux
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  -e WEBHOOK_URL=https://YOUR_NGROK_URL \
  -e N8N_ENCRYPTION_KEY=your32charstring \
  n8nio/n8n
```

### Step 2 — Start ngrok
```bash
# Use static domain (recommended — URL never changes)
ngrok http --domain=your-static-domain.ngrok-free.app 5678

# Or regular tunnel (URL changes every restart)
ngrok http 5678
```

### Step 3 — Import workflows
1. Open n8n at `http://localhost:5678`
2. Workflows → Import from file
3. Import all 3 JSON files from the `workflows/` folder
4. Add credentials:
   - Slack Bot Token (`xoxb-...`)
   - Groq API Key (`gsk_...`)
   - Airtable Personal Access Token (`pat...`)
5. Publish each workflow

### Step 4 — Connect Slack
1. Go to api.slack.com/apps → your app
2. Event Subscriptions → enable → set Request URL:
   ```
   https://YOUR_NGROK_URL/webhook/support-bot
   ```
3. Subscribe to bot event: `message.channels`
4. Save → reinstall app
5. In Slack: `/invite @YourBotName` in your #support channel

### Slack Bot Scopes Required
```
channels:history
channels:read
chat:write
chat:write.public
reactions:write
```

---

## 📁 Project Structure

```
n8n-slack-support-bot/
├── workflows/
│   ├── support-bot-main.json       # Main support bot workflow
│   ├── escalation-workflow.json    # 24h escalation workflow
│   └── weekly-summary.json         # Weekly report workflow
├── assets/
│   └── demo.gif                    # Screen recording demo
├── README.md                       # This file
└── .env.example                    # Environment variable template
```

---

## 🔐 Environment Variables

```env
# n8n
WEBHOOK_URL=https://your-ngrok-url.ngrok-free.app
N8N_ENCRYPTION_KEY=your_random_32_character_string

# Slack
SLACK_BOT_TOKEN=xoxb-your-slack-bot-token

# Groq
GROQ_API_KEY=gsk_your-groq-api-key

# Airtable
AIRTABLE_PAT=patYourAirtablePersonalAccessToken
```

---

## 💡 Key Technical Decisions & Lessons Learned

**1. Slack webhooks require HTTPS**
Slack's Event API only sends to verified HTTPS endpoints. ngrok provides this tunnel for local development. For production, any HTTPS domain works (Render, Oracle Cloud Free Tier, etc.)

**2. Slack URL verification challenge**
When registering a webhook URL, Slack sends a `url_verification` challenge that your endpoint must echo back immediately. The first Code node handles this before any other processing runs.

**3. Bot loop prevention is critical**
Without checking `event.bot_id` and `event.subtype`, the bot would reply to its own messages, causing an infinite loop that exhausts your API quota instantly.

**4. Groq vs Gemini**
Initially attempted Google Gemini API but hit `limit: 0` quota errors due to Google Cloud project restrictions. Switched to Groq which has a more generous and reliable free tier for new accounts.

**5. Unix timestamp conversion**
Slack sends timestamps as Unix floats (e.g. `1779033759.738109`). Airtable expects ISO 8601 strings. Conversion:
```javascript
new Date($json.timestamp * 1000).toISOString()
```

**6. n8n item aggregation**
n8n processes items in a pipeline — when Airtable Search returns multiple records, each downstream node runs once per item. The escalation workflow uses a Code node to aggregate all items into one before the Slack node, so only one message is sent regardless of how many overdue tickets exist.

**7. n8n IF node dual branching**
The escalation workflow connects two separate node chains from the IF true output — one for the aggregated Slack message and one for the per-item Airtable update. n8n supports multiple connections from a single output.

---

## 🔮 Planned Improvements

- [ ] Deploy to Oracle Cloud Free Tier for 24/7 uptime
- [ ] Add `/resolve [ticket-id]` Slack slash command to close tickets
- [ ] Email notifications via Gmail for weekly summary (Currently through slack)
- [ ] Dashboard view in Airtable with ticket trend charts
- [ ] Sentiment analysis layer to improve priority scoring
- [ ] Response time tracking (CreatedAt vs ResolvedAt)

---

## 👤 Author

**Laiba Mohsin**
Final year CS student | Automation & AI integrations

---

## 📄 License

MIT — feel free to fork and build on top of this.
