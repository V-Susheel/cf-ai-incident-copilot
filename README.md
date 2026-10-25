# cf_ai_incident-copilot

An AI **on-call incident copilot** built entirely on Cloudflare. You describe a production incident in chat (or by voice); the copilot helps you triage it in real time, **remembers** what's been established (severity, affected services, facts, timeline), and when things are stable, a **durable Workflow** drafts a blameless postmortem.

> Live demo: `https://cf-ai-incident-copilot.<your-subdomain>.workers.dev` *(add your link after `npm run deploy`)*

<!-- Add a screenshot after deploying: ![screenshot](docs/screenshot.png) -->

## How it maps to the assignment

| Requirement | Implementation |
|---|---|
| **LLM** | Llama 3.3 70B (`@cf/meta/llama-3.3-70b-instruct-fp8-fast`) on **Workers AI**, streamed token-by-token. A second, low-temperature call extracts structured incident data as JSON after each turn. |
| **Workflow / coordination** | **Agents SDK** `Agent` (a Durable Object) coordinates each incident session. A **Cloudflare Workflow** (`PostmortemWorkflow`) runs a 5-step, retried, checkpointed pipeline: load incident → summarize impact → analyze root cause → derive action items → save back to the agent via Durable Object RPC. |
| **User input (chat / voice)** | Chat UI served by **Workers Static Assets** over a WebSocket to the agent. Optional voice input via the browser Web Speech API (Chrome/Edge). |
| **Memory / state** | Each incident is its own Agent instance. **Agent state** (`setState`) holds the structured record (title, severity, status, services, learned facts, postmortem). The Agent's embedded **SQLite** (`this.sql`) stores the full transcript and timeline. Close the tab, come back to the same URL, and everything is still there. |

## Architecture

```
 Browser (public/index.html)
   │  WebSocket  /agents/incident-agent/<incident-id>
   ▼
 Worker (src/index.ts) ── routeAgentRequest ──▶ IncidentAgent  (Durable Object, src/agent.ts)
                                                 │  state: severity, status, services, facts, postmortem
                                                 │  sqlite: messages, timeline
                                                 │
                                                 ├─▶ Workers AI · Llama 3.3   (streamed reply)
                                                 ├─▶ Workers AI · Llama 3.3   (JSON memory extraction)
                                                 │
                                                 └─▶ PostmortemWorkflow (src/workflow.ts)
                                                       step: load incident      ◀── RPC to agent
                                                       step: summarize impact   ──▶ Llama 3.3
                                                       step: analyze root cause ──▶ Llama 3.3
                                                       step: action items       ──▶ Llama 3.3
                                                       step: save to agent      ──▶ RPC → broadcast to UI
```

### Per-turn flow
1. User message is written to SQLite and broadcast to every open tab for that incident.
2. The prompt = system prompt + **incident record** (from state + timeline) + last 20 turns.
3. Llama 3.3 streams the reply over the WebSocket.
4. A second LLM pass returns JSON (`severity`, `status`, `services`, `new_facts`, `timeline_event`), which is validated and merged into state. The side panel updates live.

Because the record is fed back into every prompt, the copilot doesn't re-ask for things it already knows, even after the transcript window has scrolled past them.

## Run it

**Prerequisites:** Node.js 18+, a Cloudflare account (free plan works), Workers AI enabled.

```bash
git clone https://github.com/<you>/cf_ai_incident-copilot.git
cd cf_ai_incident-copilot
npm install
npx wrangler login
```

### Local development
```bash
npm run dev
```
Open http://localhost:8787. Durable Objects and Workflows run locally in `workerd`; Workers AI calls are proxied to your Cloudflare account (this is why `wrangler login` is required even for local dev, and why it may incur small Workers AI usage).

### Deploy
```bash
npm run deploy
```
Wrangler prints the `*.workers.dev` URL. Open it; each new incident gets its own URL hash (`#inc-20261025-ab12`) that you can bookmark or share with a teammate. Both of you see the same live session.

### Type-check
```bash
npm run typecheck   # generates Env types from wrangler.jsonc, then runs tsc
```

## Try it

1. Click one of the example incidents (or write your own), e.g.
   *"Checkout API p99 latency jumped from 200ms to 4s about 10 minutes ago. Error rate is 8%. We deployed payments-service v2.14 at 14:02 UTC."*
2. Follow up with what you find: *"Rolled back to v2.13 at 14:20, latency is recovering."* Watch severity, status, services, facts, and the timeline update on the right.
3. Refresh the page or open the URL in another tab: the conversation and record are restored from the Durable Object.
4. Click **Generate postmortem**. The Workflow runs (you can watch it with `npx wrangler workflows instances list postmortem-workflow`) and the draft appears under **View**, with a copy-to-Markdown button.
5. **New incident** starts a fresh agent instance with empty memory; **Reset** clears the current one.

## Project layout

```
src/
  index.ts      Worker entry: /api/health, agent routing, static assets
  agent.ts      IncidentAgent: WebSocket protocol, chat turn, memory extraction, SQLite, RPC methods
  workflow.ts   PostmortemWorkflow: durable multi-step postmortem generation
  llm.ts        Workers AI helpers (streaming SSE parsing, JSON extraction)
public/
  index.html    Chat UI (vanilla JS, no build step) + voice input
  vendor/       marked + DOMPurify (Markdown rendering, sanitized)
wrangler.jsonc  AI, Durable Object, Workflow, and assets bindings
PROMPTS.md      AI prompts used to build this project
```

## Design notes and trade-offs

- **Plain `Agent` instead of `AIChatAgent`.** I wanted control over the memory model (structured record + transcript) and a framework-free UI, so the agent speaks a small JSON protocol over the WebSocket (`snapshot`, `message`, `token`, `assistant_end`, `record`, `error`) and disables the SDK's built-in protocol frames.
- **Workflow for the postmortem, not the chat turn.** Chat needs low latency, so it runs inline in the Durable Object. The postmortem is long, multi-call, and should survive transient model errors, which is exactly what Workflows' per-step retries and checkpointing are for. If step 4 fails, steps 1–3 are not re-run.
- **Two-pass memory.** The reply model is free to be conversational; a separate temperature-0 call does structured extraction, and the server validates enum fields before writing state, so a malformed model output can't corrupt the record.
- **Safety.** Model output is rendered through DOMPurify; user input is length-capped; the copilot is instructed to label guesses as hypotheses and prefer reversible mitigations.

## Possible next steps
- Tool calls (e.g., query a metrics API or status page) using the Agents SDK MCP client
- Scheduled check-ins with `this.schedule()` (e.g., "remind me to post a status update every 30 minutes")
- Realtime voice using Cloudflare Realtime instead of browser speech recognition
