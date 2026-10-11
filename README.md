# HalalReady

A team of AI agents that helps a small Malaysian food factory prepare its halal certification application.
The owner drops in their files: registration, recipes, supplier lists, certificates and photos of the premises.
The agents sort the files, build the ingredient list, check every supplier certificate, chase suppliers for
missing or expired ones, draft the internal halal documents, self-check the premises from photos, and
assemble an application pack. The owner approves every message and document before it goes anywhere.

HalalReady never says a product is halal or not halal. It reports what is documented and what is missing.
All sample data is fictional.

Built for the AI Hackerdorm Dormathon 2026, Track 2 (Tencent WorkBuddy).

## How it fits together

```
 WorkBuddy (agent team)                HalalReady                          Owner
 ─────────────────────                 ──────────                          ─────
 Coordinator           ┐
 Certificate Checker   │   MCP     ┌──────────────┐   HTTP   ┌──────────┐        ┌───────────┐
 Supplier Chaser       ├─────────► │  connector/  ├────────► │ backend/ │ ◄────► │ dashboard │
 Document Writer       │  (stdio   │  server.py   │          │ FastAPI  │  SSE   │  (browser)│
 On-site Inspector     │  or HTTP) │  16 tools    │          │ + SQLite │        └───────────┘
 Renewal Watch         │           └──────────────┘          └────┬─────┘
 Auditor               ┘                                          │ WhatsApp / Telegram
                                                                  ▼
                                                              Suppliers
```

- **WorkBuddy** runs the agents. Every agent reaches HalalReady through one shared MCP connector.
- **The connector** turns each tool call into a call to the backend. It carries the rules every agent
  follows, so they hold whichever model is behind WorkBuddy.
- **The backend** owns the data and does all the calculations: certificate expiry, certificate status
  and the readiness score. Agents read these values and never work them out themselves.
- **The dashboard** shows the owner what each agent is doing live. It is also where the owner approves
  plans, supplier messages and documents.

## WorkBuddy connection

HalalReady plugs into WorkBuddy as a custom MCP connector called `halalready`.

### What the connector gives the agents

| Area | Tools |
|---|---|
| Teamwork | `create_task` hands work to another agent, `get_my_tasks` picks up your own work, `complete_task` closes it |
| Live activity | `post_event` is called after every step, so the dashboard shows each agent working in real time |
| The case | `get_case`, `get_files`, `read_file` (returns images, so the agent sees certificates and photos), `sort_file` |
| Records | `save_ingredients`, `get_certificates`, `save_certificate`, `save_site_finding`, `save_document` |
| The owner | `request_approval` queues a supplier message, document or question until the owner says yes |
| Output | `set_status` writes the one-line summary for the owner, `assemble_pack` builds the application pack |

The connector gives every agent the same instructions:

- Post an event after every step.
- Hand work over through tasks, never by guessing.
- Never send anything yourself. Supplier messages and documents go to the owner through `request_approval`.
- Never calculate dates, certificate status or the readiness score. Read them from the backend.
- Never say anything is halal or not halal.

### Register it in WorkBuddy

`connector/workbuddy-mcp.json` is ready to paste into WorkBuddy's MCP / Connectors settings:

```json
{
  "mcpServers": {
    "halalready": {
      "command": "C:/Documents/halalal/.venv/Scripts/python.exe",
      "args": ["C:/Documents/halalal/connector/server.py"],
      "env": { "BACKEND_URL": "http://localhost:8002", "CONNECTOR_TOKEN": "" }
    }
  }
}
```

WorkBuddy starts the connector itself over stdio. If you'd rather run it as a server, start
`.venv\Scripts\python connector\server.py --http` and register `http://127.0.0.1:8001/mcp` instead.

### Verified

`scripts/test_handoff.py` starts the connector over stdio, exactly as WorkBuddy does, and plays a full
handoff between two agents:

```
Connector tools (16): assemble_pack, complete_task, create_task, get_case, ...

Certificate Checker
  belacan certificate status: expired (-62 days)
  handed task #1 to supplier_chaser
Supplier Chaser
  picked up task #1: Ask Belacan Melaka for a renewed certificate (in_progress)
  approval #1 waiting for the owner
  owner approved -> sent via manual

PASS: handoff visible in events
```

## The agent team

| Agent | Job |
|---|---|
| Coordinator | Sorts the owner's files, plans the work and hands it out, keeps the owner updated |
| Certificate Checker | Builds the ingredient list from the recipes and checks every supplier certificate (issuer, expiry, brand) |
| Supplier Chaser | Drafts short, polite messages in Malay and English asking suppliers for missing or renewed certificates |
| Document Writer | Drafts the internal halal documents (halal policy and others) from the business's real details |
| On-site Inspector | Pre-audit self-check of the store, processing, packing, cleaning and waste zones from photos |
| Renewal Watch | Wakes up when a supplier certificate is about to expire and asks the Supplier Chaser to chase it |
| Auditor | Checks every document against the requirement checklist and sends weak work back to the Writer |

Each role's instructions are in `agents/prompts/`. Shared rules are in `agents/prompts/_shared.md`.
The rule files are in `rules/`: recognised certifiers and the required sections of each document.

## Run it

```
python -m venv .venv
.venv\Scripts\pip install -r backend\requirements.txt -r connector\requirements.txt -r agents\requirements.txt
copy env.example .env          # then set AGENT_MODE=workbuddy and BACKEND_URL=http://localhost:8002

.venv\Scripts\python -m uvicorn backend.app:app --port 8002
```

Then:

1. Open the dashboard at http://localhost:8002/app/#demo.
2. Drop in the 16 files from `demo_case/upload/` and press **Analyze**. This gives the Coordinator its
   first task.
3. In WorkBuddy, the agents pick up their tasks through the `halalready` connector. Their work shows up
   live on the dashboard.

`AGENT_MODE` chooses who runs the agents:

| Value | Agents run in |
|---|---|
| `workbuddy` | WorkBuddy, through the MCP connector. The backend doesn't start any agents itself. |
| `scripted` | `agents/scripted.py`, a fixed timed demo run that needs no model |
| `gemini` | `agents/runner.py`, the built-in runner on Gemini |

## Owner in the loop

- **Plans:** the Coordinator's plan waits for the owner's OK before the specialists start.
- **Supplier messages:** the owner picks WhatsApp, Telegram or email and approves the text. In `manual`
  mode nothing leaves the laptop.
- **Supplier replies:** replies come back through `/webhooks/*`. A WhatsApp message with no reply is
  re-sent on Telegram after `FALLBACK_AFTER_MINUTES`.
- **Documents:** every drafted document waits for the owner's approval before it goes into the pack.
- **Application pack:** a readiness checklist with a score, the open items, and every document and
  certificate, ready for the owner to submit on MYeHALAL.

## Project layout

```
backend/     FastAPI app, SQLite database, date and score calculations, messaging, webhooks
connector/   MCP server for WorkBuddy (server.py) and the WorkBuddy config (workbuddy-mcp.json)
agents/      agent instructions (prompts/), built-in Gemini runner, scripted demo run
dashboard/   live owner dashboard: input tray, agent activity, approvals, application pack
rules/       recognised certifiers, document requirements
demo_case/   fictional demo files (upload/) and follow-up files (follow_up/)
sample_case/ fictional sample case with ANSWER_KEY.md
scripts/     test_handoff, reset_db, replay_events, tunnel, demo case generators
tests/       unit tests for calculations, messaging, office files and rules
```

More detail: `CONTRACT.md` (events, tasks and API), `DEMO.md` (demo runbook),
`HalalReady-proposal.md` (the proposal).

## Tests

```
.venv\Scripts\python -m pytest tests
.venv\Scripts\python scripts\test_handoff.py --backend http://localhost:8002
```
