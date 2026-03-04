# Link Prospecting Workflow

Automated guest-post / link prospecting using SerpAPI + Python MCP Server + n8n + Google Sheets.

## How It Works

1. You provide a **niche keyword** (e.g. "digital marketing", "fitness", "SaaS")
2. 20 pre-built Google search query templates are populated with your niche
3. SerpAPI runs each query and collects SERP results
4. Results are deduplicated, filtered, and written to Google Sheets

```
n8n (orchestrator)
  └─▶ Python MCP Server (port 8000)
        └─▶ SerpAPI → Google Search results
  └─▶ Google Sheets (Prospects tab)
```

## Prerequisites

- Docker + Docker Compose
- [SerpAPI account](https://serpapi.com/) (free tier: 100 searches/month)
- Google Cloud project with **Sheets API** enabled
- Google OAuth2 credentials (see setup below)

---

## Setup

### 1. Clone & configure environment

```bash
cp .env.example .env
```

Edit `.env` and fill in:
- `SERPAPI_API_KEY` — from your SerpAPI dashboard
- `N8N_ENCRYPTION_KEY` — generate with: `python -c "import secrets; print(secrets.token_hex(16))"`

### 2. Prepare Google Sheets

Create a new Google Sheet. In the first tab (rename it **Prospects**), add these headers in row 1, columns A–M:

```
niche | domain | url | title | snippet | source_query | category | position | status | da_score | notes | discovered_at | run_id
```

Copy the Sheet ID from the URL:
`https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit`

### 3. Set up Google OAuth2 in n8n

1. In [Google Cloud Console](https://console.cloud.google.com/), enable the **Google Sheets API**
2. Create **OAuth 2.0 Client ID** credentials (Web application type)
3. Add `http://localhost:5678/rest/oauth2-credential/callback` as an authorized redirect URI
4. Note your Client ID and Client Secret

### 4. Start the stack

```bash
docker-compose up --build -d
```

- MCP Server: http://localhost:8000
- n8n: http://localhost:5678

### 5. Configure n8n

1. Open http://localhost:5678, create your account
2. Go to **Credentials** → Add **Google Sheets OAuth2** credential
   - Enter your Client ID and Client Secret from step 3
   - Click Connect to authorize
3. Go to **Workflows** → Import → paste contents of `n8n/workflow.json`
4. In the imported workflow, open the **Write to Google Sheets** node:
   - Set the Document ID to your Sheet ID
   - Set the credential to the one you just created
5. In the **Set Variables** node, change `niche` to your target niche

### 6. Run it

Click **Execute Workflow** in n8n. Watch prospects populate in your Google Sheet.

---

## MCP Server Tools

The Python MCP server exposes 3 tools at `http://localhost:8000/mcp`:

| Tool | Description |
|------|-------------|
| `get_query_templates(niche)` | Returns 20 queries with niche substituted. No API credits used. |
| `search_prospects(query, niche, num_results)` | Single SerpAPI call. Use in n8n loops for fine-grained control. |
| `batch_search(queries, niche, num_results, delay_seconds)` | Runs all queries with built-in rate-limit pacing. |

### Test the MCP server directly

```bash
# List available tools
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'

# Get query templates for a niche
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_query_templates","arguments":{"niche":"fitness"}}}'
```

---

## Query Templates (20 total)

| Category | Template |
|----------|----------|
| write_for_us | `"{niche}" "write for us"` |
| write_for_us | `"{niche}" "write for us" site:*.com` |
| write_for_us | `intitle:"write for us" "{niche}"` |
| write_for_us | `inurl:"write-for-us" "{niche}"` |
| guest_post | `"{niche}" "guest post"` |
| guest_post | `"{niche}" "guest post guidelines"` |
| guest_post | `"{niche}" "become a contributor"` |
| guest_post | `intitle:"guest post" "{niche}"` |
| guest_post | `inurl:"guest-post" "{niche}"` |
| submit | `"{niche}" "submit a post"` |
| submit | `"{niche}" "submit an article"` |
| submit | `"{niche}" "contribute" "article"` |
| submit | `"{niche}" "contributor guidelines"` |
| contributor_page | `"{niche}" inurl:"contributors"` |
| contributor_page | `"{niche}" "guest author"` |
| contributor_page | `"{niche}" "written by a guest"` |
| paid | `"{niche}" "sponsored post"` |
| paid | `"{niche}" "advertise" "sponsored content"` |
| advanced | `"{niche}" intitle:"submit" intitle:"article"` |
| advanced | `"{niche}" inurl:"write" inurl:"us"` |

---

## Google Sheets Column Reference

| Col | Header | Notes |
|-----|--------|-------|
| A | niche | Your input niche |
| B | domain | Domain (dedup key, www. stripped) |
| C | url | Full URL |
| D | title | Page title |
| E | snippet | SERP snippet |
| F | source_query | Which template found it |
| G | category | write_for_us / guest_post / paid / etc. |
| H | position | SERP rank (1–10) |
| I | status | new → contacted → declined → placed |
| J | da_score | Fill manually (or add Moz API later) |
| K | notes | Manual notes |
| L | discovered_at | ISO timestamp |
| M | run_id | Workflow run identifier |

---

## SerpAPI Usage & Cost

| Plan | Searches/month | Cost | Runs possible (20 queries/run) |
|------|---------------|------|-------------------------------|
| Free | 100 | $0 | 5 |
| Starter | 5,000 | ~$50 | 250 |
| Production | 15,000 | ~$130 | 750 |

---

## Local Development (without Docker)

```bash
# Install dependencies
pip install -r requirements.txt

# Set environment variables
cp .env.example .env
# Edit .env with your SERPAPI_API_KEY

# Run MCP server
cd mcp_server
python server.py

# Run n8n separately (follow n8n local install docs)
# Update MCP endpoint URL in workflow.json:
# "http://mcp-server:8000/mcp" → "http://localhost:8000/mcp"
```

---

## Project Structure

```
LinkProspecting/
├── mcp_server/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── server.py           # FastMCP app + tool definitions
│   ├── serpapi_client.py   # SerpAPI wrapper with retry & throttle
│   ├── queries.py          # 20 query templates + substitution
│   └── models.py           # Pydantic data models
├── n8n/
│   └── workflow.json       # n8n workflow (import this)
├── .env.example            # Environment variable template
├── requirements.txt        # Python dependencies
├── docker-compose.yml      # Docker stack definition
└── README.md
```
