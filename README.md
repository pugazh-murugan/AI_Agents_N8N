# AI_Agents_N8N

A collection of exportable [n8n](https://n8n.io) workflows that show different ways to build AI agents: a human-approved content pipeline, a reusable MCP tool server, and agents that use those tools from chat, Telegram and a web app.

## Workflows

| File | Trigger | What it does | Active in export |
| --- | --- | --- | --- |
| [AI LinkedIn Post – Approval to Sheets.json](<AI LinkedIn Post – Approval to Sheets.json>) | n8n Form | Writes and reviews a LinkedIn post, asks for approval by email, then logs approved posts to Google Sheets | No |
| [MCP Server – Multi Tool.json](<MCP Server – Multi Tool.json>) | MCP Server Trigger | Exposes Gmail, Sheets, Docs, Calendar, Calculator and SerpApi as one MCP server | **Yes** |
| [Ai Agent mcp enabled.json](<Ai Agent mcp enabled.json>) | Chat message | Chat agent with memory that calls tools from two MCP servers | No |
| [Multi AI Agent New Workflow.json](<Multi AI Agent New Workflow.json>) | Telegram message | Telegram assistant that uses the MCP server, a research sub-agent and human approval for email | No |
| [Web App.json](<Web App.json>) | Webhook (`mywebook`) | Backend for a web app: a personal-assistant agent with the same tools, returned as JSON | No |

### 1. AI LinkedIn Post – Approval to Sheets

Human-in-the-loop content pipeline.

```
Topic Form → Writer → Reviewer → Send for Approval (Gmail) → Approved? → Save to Google Sheets
```

- **Topic Form** collects `Topic` (required), `Audience` and `Tone` (Professional, Casual, Inspirational or Storytelling).
- **Writer** (OpenAI `gpt-4.1-mini`) drafts one 120–200 word post with a hook, a closing question and 3–5 hashtags.
- **Reviewer** (OpenAI `gpt-4.1-mini`) edits the draft and returns only the final plain-text post.
- **Send for Approval** emails the post using Gmail *Send and Wait* with **Approve & Save** and **Reject** buttons. The workflow pauses until you click one.
- **Approved?** sends only approved posts on. Rejected posts end the run.
- **Save to Google Sheets** appends a row with the columns `Date`, `Topic`, `Post` and `Status` (set to `Approved`).

### 2. MCP Server – Multi Tool

A single [MCP](https://modelcontextprotocol.io) server that other agents and MCP clients can use. It is served at the path `multi-tool-mcp`.

| Tool | Service | Purpose |
| --- | --- | --- |
| `gmail_tool` | Gmail | Send email |
| `sheets_get_row` | Google Sheets | Read rows |
| `sheets_append_row` | Google Sheets | Append a row (title, date, time) |
| `docs_create` | Google Docs | Create a document |
| `docs_update` | Google Docs | Update a document (uses the id or URL from `docs_create`) |
| `calendar_tool` | Google Calendar | Create an event |
| `calendar_get_events` | Google Calendar | List events |
| `calculator` | n8n Calculator | Do maths |
| `Google search in SerpApi` | SerpApi | Google web search |
| `Amazon search in SerpApi` | SerpApi | Amazon product search |

This is the only workflow that is active in the export, because the other agents depend on it.

### 3. Ai Agent mcp enabled

A minimal chat agent for trying MCP.

- **Trigger:** n8n chat message.
- **Model:** OpenAI `gpt-4.1-mini`, with *Simple Memory* (window buffer).
- **Tools:**
  - **MCP Client** connects to the Multi Tool server over SSE and exposes only the Google search tool.
  - **MCP n8n Instance** connects to the n8n instance's built-in MCP server (`/mcp-server/http`) using OAuth2.

### 4. Multi AI Agent New Workflow

A Telegram personal assistant that combines several agents.

```
Telegram Trigger → AI Agent → Send a text message
```

- **Main agent:** OpenAI `gpt-4.1-mini`, with **DeepSeek** (`deepseek-v4-pro`) as an automatic fallback model.
- **Memory:** per-chat session, keyed by the Telegram chat id.
- **Tools:**
  - **MCP Server – Multi Tool** gives access to all the tools above.
  - **AI Agent Tool** is a research sub-agent (OpenAI `gpt-5-mini` plus SerpApi Google search) for live web questions.
  - **HITL** wraps the Gmail send tool, so the agent has to get your approval in Telegram before an email goes out.
- **System prompt:** the agent knows the date and timezone (Asia/Kolkata), must call tools for live facts, uses `+05:30` calendar times with a 1-hour default, and replies in short plain text.

### 5. Web App

The same assistant behind an HTTP endpoint, so any front end can call it.

```
Webhook → AI Agent → Respond to Webhook
```

- **Webhook** path is `mywebook`, with response mode *Respond to Webhook* and CORS open to `*`.
- The agent reads the question from the query string: `?message=...`.
- **Model:** OpenAI `gpt-4.1-mini`.
- **Tools:** Gmail, Sheets, Docs, Calendar, Calculator and SerpApi, wired in directly rather than through MCP.
- **Response:** `{ "reply": "<agent output>" }`

```bash
curl "https://<your-n8n-host>/webhook/mywebook?message=What%27s%20on%20my%20calendar%20today%3F"
```

Use `/webhook-test/mywebook` while testing in the editor. Tighten the `*` CORS setting before you expose it publicly.

## How the pieces fit together

```
                         ┌──────────────────────────────┐
 Chat ─► Ai Agent mcp ──►│                              │
                         │  MCP Server – Multi Tool     │──► Gmail · Sheets · Docs
 Telegram ─► Multi Agent ►│  (path: multi-tool-mcp)      │──► Calendar · Calculator
                         │                              │──► SerpApi (Google, Amazon)
                         └──────────────────────────────┘

 Web App (webhook) ──► same tools, attached directly
 LinkedIn form     ──► standalone approval pipeline
```

## Setup

### Requirements

- An n8n instance (self-hosted or cloud). The MCP Server Trigger, agent, HITL and *Send and Wait* nodes need a recent n8n version.
- Accounts and n8n credentials for the services you use:

| Credential | Used by |
| --- | --- |
| OpenAI API | All agents and the LinkedIn writer and reviewer |
| DeepSeek API | Fallback model in the Telegram workflow |
| Gmail OAuth2 | Approval email and `gmail_tool` |
| Google Sheets OAuth2 | LinkedIn log and `sheets_*` tools |
| Google Docs OAuth2 | `docs_*` tools |
| Google Calendar OAuth2 | `calendar_*` tools |
| SerpApi | Google and Amazon search |
| Telegram API | Telegram trigger, reply and HITL |
| MCP OAuth2 | Connection to the n8n built-in MCP server (workflow 3) |

### Import

1. In n8n, choose **Workflows → Import from file** and select a `.json` file. You can also paste the file's contents into the canvas.
2. Open each node that shows a credential warning and select your own credential.
3. Update the values below, then save.
4. **Activate** the *MCP Server – Multi Tool* workflow first, since the other agents depend on it.

### Things to change after importing

These values come from the original author's setup:

- **Google Sheet:** the Sheets nodes point at a sheet named *Student Details*. Select your own spreadsheet and tab. The LinkedIn log needs the columns `Date`, `Topic`, `Post` and `Status`.
- **Approval recipient:** the `sendTo` address on *Send for Approval* is hard-coded. Set it to your own email.
- **MCP endpoint:** *Ai Agent mcp enabled* and *Multi AI Agent New Workflow* have the endpoint URL hard-coded to the original author's n8n instance. Replace it with `https://<your-n8n-host>/mcp/multi-tool-mcp`.
- **Persona and region:** the prompts assume a developer in Bangalore (Asia/Kolkata, `+05:30`). Adjust them for your timezone.
- **Models:** model names (`gpt-4.1-mini`, `gpt-5-mini`, `deepseek-v4-pro`) are set per node. Change them to models available to your accounts.

## Security notes

- The exported JSON contains credential *names* only, never secrets. Check any new export before committing it.
- The MCP server can send email and edit Docs, Sheets and Calendar. Protect it with authentication before exposing it publicly.
- The Telegram agent asks for approval before it sends email, but the Web App agent does not. Add an approval step if you expose that endpoint.
- Do not commit API keys, OAuth tokens or `.env` files.

## Repository layout

```
.
├── AI LinkedIn Post – Approval to Sheets.json
├── MCP Server – Multi Tool.json
├── Ai Agent mcp enabled.json
├── Multi AI Agent New Workflow.json
├── Web App.json
└── README.md
```
