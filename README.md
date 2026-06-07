Week 10 Final Capstone — Production Upgrade
*Eric Munjiru & Pearl Wangechi* · Moringa School · June 2026

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [What Changed from v1](#what-changed-from-v1)
3. [Architecture](#architecture)
4. [Project Structure](#project-structure)
5. [Prerequisites](#prerequisites)
6. [Environment & Credentials Setup](#environment--credentials-setup)
7. [Installation & Import Guide](#installation--import-guide)
8. [Configuration](#configuration)
9. [Usage Guide](#usage-guide)
10. [Workflow Breakdown](#workflow-breakdown)
11. [Error Handling & Fault Tolerance](#error-handling--fault-tolerance)
12. [Operational Logging](#operational-logging)
13. [Model Strategy & Fallbacks](#model-strategy--fallbacks)
14. [Deployment](#deployment)
15. [Troubleshooting](#troubleshooting)

---

## Project Overview

This workflow automates the creation and distribution of a weekly tech newsletter for *Nairobi Nexus*, a Kenyan tech publication. Every Friday at 2 PM, the system:

1. Pulls the latest articles from *5 Kenyan tech news sources* simultaneously
2. Runs a *role-based AI agent* per source to summarise and reframe content
3. Merges all successful summaries and passes them to a *Chief Editor AI agent* (Kamau) who writes a witty, Kenyan-voiced newsletter intro
4. Cleans the output and *drafts an email* to the marketing team for approval
5. *Logs the run* — timestamp, sources used, newsletter body, and models used — to a Google Sheet

The v2 upgrade transforms this from a functional prototype into a *production-grade, fault-tolerant, auditable system*.

---

## What Changed from v1

The original Week 8 workflow ran well on happy paths but had four critical weaknesses. This upgrade addresses all of them:

| Area | v1 Prototype | v2 Production |
|---|---|---|
| *AI output format* | Free text — prompt discipline only | Strict JSON enforced via Structured Output Parsers on all 6 agents |
| *Source failure handling* | One failed URL stopped the entire workflow | continueErrorOutput on all 11 HTTP + agent nodes — workflow always completes |
| *Model strategy* | Mistral only — single point of failure | 3-tier: Mistral primary → Gemini fallback → Claude Haiku (editorial) via OpenRouter |
| *Auditability* | No log — nothing to review after the fact | Timestamped Google Sheets row after every run: sources used, model IDs, body, status |
| *Output cleanliness* | Raw AI text including markdown artifacts | Code cleaner node strips backticks, headers, asterisks, and emojis before sending |
| *Node count* | ~30 nodes | 52 nodes — full production pipeline |

---

## Architecture


┌─────────────────────────────────────────────────────────────────────┐
│  TRIGGER: Weekly Schedule (Friday 14:00)                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                       │
   ┌────▼────┐          ┌──────▼──────┐         ┌─────▼──────┐
   │TechCabal│          │Citizen Dig. │         │  Techweez  │  ...+2 more
   └────┬────┘          └──────┬──────┘         └─────┬──────┘
        │ continueErrorOutput  │ continueErrorOutput   │ continueErrorOutput
   ┌────▼────┐          ┌──────▼──────┐         ┌─────▼──────┐
   │ Data    │          │  Data       │         │  Data      │
   │ Cleaner │          │  Cleaner    │         │  Cleaner   │
   └────┬────┘          └──────┬──────┘         └─────┬──────┘
        │                      │                       │
   ┌────▼──────────────────────▼───────────────────────▼──────┐
   │  Role-Based Agent (per source)                            │
   │  Primary: Mistral Small  |  Fallback: Google Gemini       │
   │  Output: Structured Output Parser (strict JSON schema)    │
   └──────────────────────────┬────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Merging Intelligence│  ← aggregates all successful branches
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Code in JavaScript  │  ← combines results array
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────────────────────┐
                    │  The Chief Editor (Kamau)             │
                    │  Model: Claude Haiku via OpenRouter   │
                    │  Tone: Witty, Kenyan, ≤200 words     │
                    └──────────┬──────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Code: Clean Output  │  ← strips markdown artifacts
                    └──────────┬──────────┘
                               │
              ┌────────────────┼──────────────────┐
              │                                   │
   ┌──────────▼──────────┐           ┌────────────▼──────────┐
   │  Gmail Draft         │           │  Google Sheets Log    │
   │  → Marketing Team    │           │  (NewsletterManager)  │
   └─────────────────────┘           └───────────────────────┘


---

## Project Structure


automated-newsletter-generator/
│
├── README.md                          # This file
│
├── workflows/
│   ├── newsletter_generator_v2.json   # Main production workflow (import this into n8n)
│   └── newsletter_generator_v1.json   # Original Week 8 prototype (reference only)
│
├── docs/
│   ├── capstone_presentation.pdf      # Panel presentation document
│   ├── architecture_diagram.png       # Full workflow canvas screenshot
│   └── node_configs/                  # Screenshots of each node configuration
│       ├── schedule_trigger.png
│       ├── role_based_agent_techcabal.png
│       ├── structured_output_parser.png
│       ├── merging_intelligence.png
│       ├── chief_editor.png
│       ├── code_cleaner.png
│       ├── google_sheets_log.png
│       └── gmail_draft.png
│
├── screenshots/
│   ├── workflow_full_canvas.png       # Full workflow screenshot
│   ├── execution_success_run.png      # Successful execution log
│   ├── execution_partial_run.png      # Partial success (source failure scenario)
│   ├── google_sheets_log_evidence.png # Log sheet with real run data
│   └── model_tracking_columns.png     # Model Used column evidence
│
└── evidence/
    ├── fallback_trigger_test.png      # Gemini fallback activation screenshot
    └── error_handling_test.png        # continueErrorOutput in action


---

## Prerequisites

- *n8n* — v1.0+ (self-hosted or n8n Cloud). [Install guide →](https://docs.n8n.io/hosting/)
- *Node.js* — v18.17.0 or higher (required for self-hosted n8n)
- *npm* — v9+ or *pnpm* v8+
- A *Google account* with access to:
  - Google Sheets (for the operational log)
  - Gmail (for newsletter drafts)
- API keys for:
  - [Mistral AI](https://console.mistral.ai/) — primary agent model
  - [Google Gemini / PaLM API](https://aistudio.google.com/) — fallback model
  - [OpenRouter](https://openrouter.ai/) — Chief Editor (Claude Haiku routing)

---

## Environment & Credentials Setup

This workflow uses *5 credential connections* in n8n. Set these up under *Settings → Credentials* before importing the workflow.

### 1. Mistral Cloud API

Credential type : Mistral Cloud API
Name            : Mistral Cloud account
API Key         : <your Mistral API key>


### 2. Google Gemini (PaLM) API

Credential type : Google PaLM API
Name            : Google Gemini(PaLM) Api account
API Key         : <your Gemini API key from Google AI Studio>


### 3. OpenRouter API

Credential type : OpenRouter API
Name            : OpenRouter account
API Key         : <your OpenRouter API key>

> OpenRouter routes to Claude Haiku for the Chief Editor node. Ensure your OpenRouter account has credits and anthropic/claude-haiku is enabled.

### 4. Google Sheets OAuth2

Credential type : Google Sheets OAuth2 API
Name            : Google Sheets account
Auth method     : OAuth2
Scopes          : https://www.googleapis.com/auth/spreadsheets

Authenticate with the Google account that owns the NewsletterManager sheet.

### 5. Gmail OAuth2

Credential type : Gmail OAuth2 API
Name            : Gmail account 2
Auth method     : OAuth2
Scopes          : https://www.googleapis.com/auth/gmail.send


---

## Installation & Import Guide

### Self-hosted n8n (recommended for full control)

bash
# Install n8n globally
npm install n8n -g

# Start n8n
n8n start

# n8n runs at http://localhost:5678


### Import the workflow

1. Open n8n in your browser
2. Go to *Workflows → Import from file*
3. Select workflows/newsletter_generator_v2.json
4. The workflow will import with all node configurations intact
5. Reconnect each credential (n8n will prompt you — match the credential names above)

### Set up the Google Sheets log

Create a Google Sheet named *NewsletterManager* with a tab called *Drafts* and the following column headers in row 1:

| A | B | C | D |
|---|---|---|---|
| Timestamp | Sources Used | Body | Model Used |

Copy the sheet URL and update the *Append row in sheet* node's documentId with your sheet URL.

---

## Configuration

All key parameters can be adjusted directly in the workflow without touching node logic:

| Parameter | Node | Default | How to change |
|---|---|---|---|
| *Schedule day* | Schedule Trigger | Every 7 days (Friday) | Edit the daysInterval and triggerAtHour fields |
| *Send time* | Schedule Trigger | 14:00 (2 PM EAT) | Change triggerAtHour |
| *Primary model* | All Role-Based Agent nodes | mistral-small-latest | Change model in each Mistral Cloud node |
| *Fallback model* | All Role-Based Agent nodes | Google Gemini | Swap the Gemini node for another LLM |
| *Editorial model* | The Chief Editor | Claude Haiku (via OpenRouter) | Change model string in OpenRouter node |
| *Newsletter tone* | The Chief Editor prompt | Witty Kenyan tech editor | Edit the system prompt in The Chief Editor node |
| *Max word count* | The Chief Editor prompt | 200 words | Edit the prompt constraint |
| *Gmail recipient* | Gmail draft node | pearl.kibui@student.moringaschool.com | Update sendTo field |
| *Log sheet* | Append row in sheet | NewsletterManager → Drafts | Update documentId with your sheet URL |

---

## Usage Guide

### Running manually (testing)
1. Open the workflow in n8n
2. Click *Execute Workflow* (the play button)
3. Watch the execution panel — each branch runs in parallel
4. Check the *Drafts* tab in your NewsletterManager Google Sheet for the log entry
5. Check Gmail (as the configured recipient) for the newsletter draft

### Running on schedule
1. Toggle the workflow to *Active* using the toggle in the top-right
2. The Schedule Trigger fires automatically every 7 days at 14:00
3. No further action needed — the log sheet is your weekly evidence trail

### Verifying a run
Open the NewsletterManager Google Sheet and check the latest row:
- *Timestamp* — confirms when it ran
- *Sources Used* — X/5 sources tells you how many succeeded
- *Body* — the newsletter text that was sent
- *Model Used* — shows which model(s) generated the output (e.g. mistral-small, gemini-2... indicates fallback activated on one branch)

---

## Workflow Breakdown

### Stage 1 — Data Collection (5 parallel branches)

Each branch fetches from one source independently:

| Branch | Source | URL |
|---|---|---|
| TechCabal | Tech & startup news | techcabal.com/category/newsletters/ |
| Citizen Digital | Business & tech | citizen.digital/news/tech/26 |
| Techweez | East Africa tech | techweez.com |
| Tech-Ish | Kenyan tech commentary | tech-ish.com |
| TechArena | Kenyan tech news | techarena.co.ke |

Every HTTP Request node has continueErrorOutput: true — a failed fetch routes to the error output and is recorded in the log, without affecting other branches.

### Stage 2 — Data Cleaning

Each branch has a dedicated *Code (JavaScript)* cleaner node that:
- Strips raw HTML tags
- Extracts readable article text
- Normalises whitespace and encoding
- Passes clean text to the role-based agent

### Stage 3 — Role-Based AI Agents (per source)

Each agent is a *LangChain agent* with:
- *Primary model*: Mistral Small (fast, cost-efficient, strong JSON compliance)
- *Fallback model*: Google Gemini (different provider = different failure profile)
- *Memory*: Simple Memory (buffer window, keyed per execution)
- *Output*: Structured Output Parser enforcing a strict JSON schema
- *Prompt*: Role-specific — e.g. the TechCabal agent is briefed differently from the TechArena agent to reflect each publication's voice

### Stage 4 — Merging Intelligence

The *Merge* node collects results from all branches that completed successfully. Whether 3, 4, or 5 branches ran, this node aggregates them into a single array passed to the Chief Editor. A *Code in JavaScript* node formats the combined results cleanly.

### Stage 5 — The Chief Editor (Kamau)

The final AI agent, running on *Claude Haiku via OpenRouter*:
- Receives all successfully merged article summaries
- Writes a single ≤200-word newsletter intro in Kamau's voice: witty, Kenyan, tech-savvy
- Records which model it used in its output
- Has needsFallback: true — OpenRouter auto-routes if Haiku is unavailable

*System persona excerpt:*
> "You are Kamau, a witty Kenyan tech editor writing engaging newsletter intros for African startup and technology audiences. Use occasional Kenyan slang naturally (Bazeng, mambo ni matatu, si mchezo, maze). Sound like a smart insider summarizing what matters in tech this week."

### Stage 6 — Output Cleaning

The *Code to clean the markdown* node strips any residual formatting from the AI output:
- Removes `  ` markdown code blocks
- Strips `#`, `*`, `_`, `>` formatting symbols
- Removes emojis
- Normalises newlines and whitespace

### Stage 7 — Output & Logging (parallel)

Two nodes run in parallel after cleaning:

**Gmail Draft** — sends the clean newsletter to the marketing team (`sendAndWait` mode — team must approve before it goes out)

**Google Sheets Log** — appends a row to NewsletterManager → Drafts with:
- `Timestamp` — ISO datetime of execution
- `Sources Used` — count of successful sources (e.g. `5/5 sources`)
- `Body` — full newsletter text
- `Model Used` — comma-separated list of model IDs used across all agents

---

## Error Handling & Fault Tolerance

### continueErrorOutput — how it works

n8n's `continueErrorOutput` setting routes failed items through the **error output** of a node instead of stopping execution. This is enabled on:

- All 5 HTTP Request (source fetch) nodes
- All 5 role-based agent nodes
- The Chief Editor agent node

This means **no single failure stops the workflow**. Failed branches are silently isolated; successful branches continue to the merge stage.

### What each failure type does

| Failure | Detection | Recovery |
|---|---|---|
| Source URL unreachable / 4xx / 5xx | `continueErrorOutput` on HTTP node | Branch exits early; other 4 branches unaffected |
| AI agent returns malformed JSON | Structured Output Parser catches schema mismatch | Branch error-routed; merge receives remaining valid results |
| Primary model timeout / quota exceeded | `continueErrorOutput` + Gemini fallback wired to same agent | Gemini activates on the same execution without restart |
| Chief Editor failure | `needsFallback: true` + OpenRouter auto-routing | OpenRouter switches to available model automatically |
| All 5 sources fail | Merge receives empty set | Chief Editor still runs; log records `0/5 sources`; Gmail draft sent with notice |

### Partial success handling

The **Merging Intelligence** node is configured to accept results from whichever branches completed. The log's `Sources Used` field makes partial success immediately visible — `3/5 sources` tells the team exactly what happened, without anyone opening n8n.

---

## Operational Logging

After every execution, a row is appended to the **NewsletterManager Google Sheet** (Drafts tab):

| Column | What is recorded | Operational value |
|---|---|---|
| Timestamp | ISO datetime of the run | Confirms when the workflow executed |
| Sources Used | `X/5 sources` | Immediately shows if any sources failed |
| Body | Full newsletter text | Proof of what was sent; replayable |
| Model Used | Model IDs from all agents | Shows which models ran; Gemini appearing indicates fallback activated |

**If someone asks "what happened last Friday?"** — open the sheet, find the row with Friday's timestamp, and every question is answered in under 30 seconds.

A **missing row** means the run failed entirely — the absence of a log entry is itself a signal to the ops team.

---

## Model Strategy & Fallbacks


Per-source agents:
  PRIMARY  →  Mistral Small (mistral-small-latest)
                 ↓ on error/timeout
  FALLBACK →  Google Gemini

Chief Editor:
  PRIMARY  →  Claude Haiku via OpenRouter
                 ↓ if Haiku unavailable
  AUTO     →  OpenRouter routes to next available Anthropic model


**Why Mistral as primary?** Fast inference, low cost, strong JSON schema compliance, minimal hallucination on structured extraction tasks.

**Why Gemini as fallback?** Runs on Google infrastructure — a different cloud provider from Mistral means the two are unlikely to fail simultaneously. Different failure profile = genuine resilience.

**Why Claude Haiku for the Chief Editor?** The editorial step produces the one piece of text the audience actually reads. Haiku has the strongest narrative coherence and tone control of the models available. The higher cost is justified for a single final-output call.

---

## Deployment

### n8n Cloud (simplest)
1. Create an account at [n8n.io](https://n8n.io)
2. Import the workflow JSON
3. Add credentials under Settings → Credentials
4. Activate the workflow
5. n8n Cloud handles hosting, restarts, and uptime

### Self-hosted (Docker — recommended for production)

bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n


For persistent deployments, use Docker Compose with a PostgreSQL backend. See the [n8n self-hosting guide](https://docs.n8n.io/hosting/installation/docker/).

**Environment variables to set:**

env
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_password
N8N_HOST=your-domain.com
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-domain.com/
```

---

## Troubleshooting

*Workflow imports but credentials show as disconnected*
> Go to Settings → Credentials and re-authenticate each one. n8n credential IDs are instance-specific and won't transfer between installations.

*One or more source branches always fail*
> Test the source URL directly in your browser. Some news sites block server-side HTTP requests (bot protection). If blocked, consider using an RSS feed URL for that source instead.

*Structured Output Parser failing on every run*
> The AI model is returning text that doesn't match the expected schema. Open the agent node, click on the last execution, and inspect the raw output. Tighten the prompt: add "Return ONLY valid JSON. No markdown, no backticks, no explanation." to the end of the system prompt.

*Gemini fallback not activating*
> Confirm your Google Gemini API key is valid and the googlePalmApi credential is correctly linked to the Gemini model nodes. Test by temporarily using an invalid Mistral key to force a fallback trigger.

*Google Sheets log not appending*
> Confirm the documentId URL in the Append row node matches your sheet URL exactly (including the gid parameter). Confirm the OAuth2 credential has spreadsheets scope. Check that the sheet tab name matches Drafts.

*Gmail draft not sending*
> The node uses sendAndWait mode — the email is sent as a draft awaiting approval, not immediately. Check the Gmail Drafts folder. If nothing appears, confirm the gmailOAuth2 credential has gmail.send scope.

*Execution times out before all branches complete*
> Increase the execution timeout in n8n Settings → General. Default is 1 hour; for 5 parallel AI branches this is usually sufficient, but slow model responses can push it.

*OpenRouter returning model unavailable errors*
> Log in to [openrouter.ai](https://openrouter.ai), check your credits balance, and confirm anthropic/claude-haiku is available in your plan. You can swap to anthropic/claude-3-haiku-20240307 as an explicit model string in the OpenRouter node.

---

## Authors

| | |
|---|---|
| *Eric Munjiru* | Workflow architecture, AI systems, model fallback strategy, structured output enforcement, GitHub & documentation |
| *Pearl Wangechi* | Fault tolerance implementation, operational logging, execution evidence, non-technical documentation |

---

Moringa School — Automation & AI Integration · Week 10 Final Capstone · June 2026
