# Automated Weekly Newsletter Generator

> A fault-tolerant, AI-powered newsletter pipeline built on n8n. Scrapes 5 African tech news sources, uses multiple AI models to curate and write, logs every run, and sends a draft for human approval — automatically, every week.

---

## Project Structure

```
newsletter-generator/
├── workflow/
│   └── Project__Automated_Weekly_Newsletter_Generator__2_.json   # n8n workflow export
├── docs/
│   └── ProjectDocumentation_NewsletterGenerator.md               # Full technical docs
│   └── NewsletterGenerator_Presentation.pptx                     # Stakeholder deck
└── README.md                                                      # This file
```

The entire workflow lives in a single n8n JSON file. There is no application code to deploy separately — n8n executes the workflow from its canvas.

---

## Environment Requirements

| Requirement | Version / Notes |
|---|---|
| **n8n** | v1.x (self-hosted or n8n Cloud) |
| **Node.js** | v18+ (required by n8n) |
| **LangChain nodes** | Bundled with n8n v1.x — no separate install |
| **Internet access** | Required to reach news sources and AI APIs |

### Required API Credentials

You must create credentials in n8n for each of the following:

| Service | n8n Credential Type | Used For |
|---|---|---|
| Mistral Cloud | `mistralCloudApi` | Fallback AI model (all branches) |
| Google Gemini (PaLM) | `googlePalmApi` | Primary AI for 3 source branches |
| OpenRouter | `openRouterApi` | Claude 3.5 Haiku (Chief Editor) + other models |
| Google Sheets | `googleSheetsOAuth2Api` | Audit log (OAuth2 — needs Google account) |
| Gmail | `gmailOAuth2` | Draft delivery and approval gate (OAuth2) |

---

## Installation

### Option A — n8n Cloud (Recommended for Getting Started)

1. Sign up at [n8n.io](https://n8n.io)
2. Create a new workflow
3. Click the **⋮** menu → **Import from file**
4. Upload `Project__Automated_Weekly_Newsletter_Generator__2_.json`
5. Follow [Setup & Configuration](#setup--configuration) below

### Option B — Self-Hosted (Docker)

```bash
# Pull and run n8n
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Open in browser
open http://localhost:5678
```

Then import the workflow JSON via the UI as described in Option A.

### Option C — Self-Hosted (npm)

```bash
npm install -g n8n
n8n start
# Open http://localhost:5678
```

---

## Setup & Configuration

### Step 1 — Import the Workflow

1. In n8n, go to **Workflows** → **New**
2. Click **⋮** → **Import from file**
3. Select the `.json` workflow file
4. The workflow canvas will load with all nodes

### Step 2 — Configure Credentials

For each credential type below, go to **Settings → Credentials → New** in n8n:

#### Mistral Cloud
- Credential name: `Mistral Cloud account`
- API Key: Get from [console.mistral.ai](https://console.mistral.ai)

#### Google Gemini
- Credential name: `Google Gemini(PaLM) Api account`
- API Key: Get from [Google AI Studio](https://aistudio.google.com/app/apikey)

#### OpenRouter
- Credential name: `OpenRouter account 2`
- API Key: Get from [openrouter.ai/keys](https://openrouter.ai/keys)
- Note: Ensure your account has credits for `anthropic/claude-3.5-haiku`

#### Google Sheets (OAuth2)
- Credential name: `Google Sheets account`
- Follow n8n's OAuth2 flow to connect your Google account
- You will need to grant access to Google Sheets

#### Gmail (OAuth2)
- Credential name: `Gmail account`
- Follow n8n's OAuth2 flow to connect your Gmail account
- You will need to grant access to send email

### Step 3 — Configure the Google Sheet

1. Create a new Google Sheet with these column headers in row 1:
   - `Date`
   - `Successful Sources`
   - `Model Used`
   - `Draft Body`
2. Copy the spreadsheet URL
3. In n8n, open the **Append row in sheet** node
4. Update the `documentId` field with your spreadsheet URL

### Step 4 — Configure the Recipient Email

1. Open the **draft to the marketing team for approval.** node (Gmail node)
2. Update the `sendTo` field with the editor's email address

### Step 5 — Activate the Workflow

1. In the workflow editor, toggle **Active** in the top-right corner
2. The workflow will now run automatically on the configured schedule (Fridays at 14:00)
3. To test immediately: click **Test Workflow** in the editor

---

## Usage Guide

### Normal Operation

Once activated, the workflow runs every 7 days at 14:00 with no intervention required.

**What happens:**
1. 5 news sources are scraped simultaneously
2. An AI agent curates the top headline from each source
3. The Chief Editor AI synthesises a 200-word newsletter draft
4. The draft is logged to Google Sheets
5. An email arrives in the editor's inbox with the draft
6. The editor clicks **Approve** or **Reject** in the email
7. (If approved, the newsletter is dispatched — you can add a final send step)

### Manual Execution

To run the workflow on demand without waiting for the schedule:

1. Open the workflow in n8n
2. Click **Test Workflow** (the ▶ button in the top toolbar)
3. The workflow executes once immediately

### Checking the Audit Log

Open your Google Sheet. Each row represents one execution:

- **Date** — when the run started
- **Successful Sources** — e.g. `4/5 sources` (indicates one source failed)
- **Model Used** — which AI models contributed to this newsletter
- **Draft Body** — the full newsletter text for reference

### Monitoring

n8n provides execution history under **Executions** in the sidebar. Each run shows:
- Which nodes succeeded or failed
- Input and output data for each node
- Timestamps and duration

---

## Deployment Instructions

### Production Checklist

- [ ] All 5 credentials configured and tested
- [ ] Google Sheet created with correct column headers
- [ ] Recipient email address updated
- [ ] Workflow activated (toggle is ON)
- [ ] Test execution completed successfully
- [ ] Google Sheet audit log populated with test row
- [ ] Approval email received and confirmed working

### Environment Variables (Self-Hosted)

If running n8n with environment variables instead of UI credentials:

```bash
# Optional: configure n8n base settings
N8N_PORT=5678
N8N_HOST=localhost
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/

# Encryption key for stored credentials (change this!)
N8N_ENCRYPTION_KEY=your-random-encryption-key-here
```

API keys are stored encrypted within n8n's credential store and are not exposed in the workflow JSON.

### Scaling Considerations

- **n8n Cloud:** Handles scaling automatically; suitable for most use cases
- **Self-hosted:** For high reliability, run n8n with a PostgreSQL database backend (not SQLite) and use Docker Compose or Kubernetes
- The workflow itself has no scaling requirements — it runs once per week and completes in under 2 minutes on average

---

## Troubleshooting

### Workflow Does Not Run on Schedule

**Cause:** Workflow is not activated.
**Fix:** Open the workflow and toggle the **Active** switch to ON.

---

### HTTP Request Nodes Fail for All Sources

**Cause:** Network connectivity issue or the n8n instance cannot reach the internet.
**Fix:** Test connectivity from the n8n host: `curl https://techcabal.com`. If self-hosted, check firewall/proxy settings.

---

### AI Agent Returns Empty Output

**Cause:** API key is invalid, exhausted, or the model is unavailable.
**Fix:**
1. Go to **Settings → Credentials** and verify the API key
2. Check your API provider dashboard for quota/billing status
3. The fallback model (Mistral) should activate automatically — if both fail, check both credentials

---

### Structured Output Parser Fails

**Cause:** The AI returned output that could not be parsed even with autoFix.
**Symptom:** The agent node shows an error in the Executions view.
**Fix:**
1. Check the raw AI output in the execution log
2. Strengthen the prompt by adding: "Return ONLY the JSON object. No explanation. No preamble."
3. Consider switching the primary model if this happens repeatedly

---

### Google Sheets Append Fails

**Cause:** OAuth token expired or insufficient permissions.
**Fix:**
1. Go to **Settings → Credentials → Google Sheets account**
2. Click **Reconnect** and re-authenticate with Google
3. Ensure the Google account has edit access to the target spreadsheet

---

### Gmail Approval Email Not Received

**Cause:** Gmail OAuth token expired, or email went to spam.
**Fix:**
1. Reconnect Gmail credentials (same as Google Sheets above)
2. Check spam/junk folder
3. Add the n8n sender address to the safe senders list

---

### Newsletter Draft is Very Short or Off-Topic

**Cause:** Multiple sources failed, leaving the Chief Editor with fewer than 2–3 stories.
**Fix:**
1. Check the Google Sheets log — look at the **Successful Sources** column
2. Check the n8n Executions view for which source nodes errored
3. If sources are regularly failing, consider replacing them with alternative URLs

---

### "Credential not found" Error

**Cause:** The workflow expects a credential with a specific name that doesn't exist.
**Fix:** The credential names in the workflow are:
- `Mistral Cloud account`
- `Google Gemini(PaLM) Api account`
- `OpenRouter account 2`
- `Google Sheets account`
- `Gmail account`

Create credentials with these exact names, or update the node references in the workflow to match your credential names.

---

## Adding a New News Source

To add a 6th source to the pipeline:

1. Add a new **HTTP Request** node pointing to the new URL; set `onError: continueErrorOutput`
2. Connect it to the **Schedule Trigger**
3. Add a new **HTML Cleaner** Code node (copy an existing one) and connect it
4. Add a new **AI Agent** node with a Mistral primary + fallback, structured parser, and memory
5. Connect the agent to the **Merging Intelligence** node (update `numberInputs` to 6)
6. Update the source name in the agent prompt

No other changes are required. The Chief Editor automatically adapts to the number of stories it receives.

---

## License & Credits

- **Workflow Design:** Built as a Week 9 capstone project
- **AI Models:** Mistral AI, Google Gemini, Anthropic Claude (via OpenRouter)
- **Automation Platform:** [n8n](https://n8n.io) (fair-code licence)
- **News Sources:** TechCabal, TechArena, Citizen Digital, Techweez, Tech-ish — all public websites
