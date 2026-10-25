# PROMPTS.md

This project was built with AI-assisted coding (Claude). This file records the prompts I gave the coding assistant and the prompts the application itself sends to Llama 3.3 at runtime.

---

## Part 1 — Prompts used while building the project

### Prompt 1 — initial build request
> help this: We plan to fast track candidates who complete an assignment to build a type of AI-powered application on Cloudflare. An AI-powered application should include the following components:
> * LLM (recommend using Llama 3.3 on Workers AI), or an external LLM of your choice
> * Workflow / coordination (recommend using Workflows, Workers or Durable Objects)
> * User input via chat or voice (recommend using Pages or Realtime)
> * Memory or state
>
> Find additional documentation here (https://developers.cloudflare.com/agents/).
> Note: AI-assisted coding is encouraged, but you have to submit prompt history.

**What the assistant did with it (summary):**
- Read the current Agents SDK docs (Agent class, `routeAgentRequest`, `this.sql`, `setState`) and the Workflows docs to match current APIs.
- Proposed the concept (on-call incident copilot, matching my SRE/DevOps background) and the component mapping: Llama 3.3 on Workers AI, an `Agent` Durable Object per incident, a Workflow for postmortem generation, a static chat UI on Workers Static Assets.
- Scaffolded `wrangler.jsonc`, `package.json`, `tsconfig.json`; inspected the installed `agents` package type definitions to confirm signatures (`onConnect`, `onMessage`, `shouldSendProtocolMessages`, `getAgentByName`, `name`).
- Wrote `src/llm.ts`, `src/agent.ts`, `src/workflow.ts`, `src/index.ts`, `public/index.html`.
- Fixed type errors reported by `tsc` (Workflow step return values must be serializable, so the RPC result is stringified inside the step; imported `WorkflowStepConfig` from `cloudflare:workers`).
- Verified the build with `wrangler deploy --dry-run`.
- Smoke-tested locally with `wrangler dev`, substituting a mock AI service binding (no Cloudflare login available in the sandbox) to confirm: WebSocket connect → streamed reply → JSON memory update → state persisted across reconnects → Workflow ran all steps and wrote the postmortem back to the agent.
- Took headless-browser screenshots, found that assistant messages didn't render when the Markdown CDN was unreachable, and fixed it by vendoring `marked` and `DOMPurify` into `public/vendor/` with a plain-text fallback.
- Wrote this README and PROMPTS.md.

<!--
Add any further prompts you give an AI assistant below (e.g. "change the color scheme",
"add a scheduled status-update reminder"), in order, verbatim.
-->

---

## Part 2 — Prompts the application sends to Llama 3.3 at runtime

### 2.1 Copilot system prompt (`src/agent.ts`, every chat turn)
The incident record is appended under `INCIDENT RECORD:` on every turn.

```
You are Incident Copilot, an experienced Site Reliability Engineer pairing with an on-call engineer during a live production incident.

How you work:
- Be concise and calm. Engineers are stressed; use short paragraphs and bullet points.
- Drive the incident forward: ask for the single most useful missing signal (error rates, recent deploys, dashboards, logs), then propose the next concrete diagnostic or mitigation step.
- Prefer safe, reversible mitigations first (rollback, feature flag off, scale out, failover) before deep root-causing.
- Give exact commands when useful (kubectl, az, aws, curl, SQL), clearly marked as examples to adapt.
- Never invent facts about the user's systems. If you are guessing, say it is a hypothesis.
- Use the INCIDENT RECORD below as memory of what is already known; do not ask again for things it already contains.
```

### 2.2 Memory extraction prompt (`src/agent.ts`, after every reply, temperature 0)
```
You maintain the structured record for a production incident.
Given the current record and the latest exchange, return ONLY a JSON object with this shape:
{
  "title": string,
  "severity": "SEV1"|"SEV2"|"SEV3"|"SEV4"|"unknown",
  "status": "investigating"|"mitigated"|"resolved",
  "services": string[],
  "new_facts": string[],
  "timeline_event": string|null
}
Only change severity/status if the user's words justify it. Do not repeat facts already in the record.
```

### 2.3 Postmortem Workflow prompts (`src/workflow.ts`)
- **Summarize impact:** system — "You write blameless incident postmortems. Output Markdown only, no preamble." user — write `## Summary` and `## Impact` from the record + transcript.
- **Analyze root cause:** same system prompt, plus "If the root cause is not confirmed in the material, say so and list the leading hypotheses." user — write `## Timeline`, `## Root cause`, `## What went well`, `## What went poorly`.
- **Action items:** system — "You are a pragmatic SRE lead. Output Markdown only, no preamble." user — a table of up to 6 actions with Type (prevent/detect/mitigate), Priority (P0–P2), Owner (TBD), derived from the analysis.
