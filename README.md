# AI Lead Qualification — Website Form (v1)

An n8n automation that ingests website form submissions, validates them, uses an LLM to score and qualify each lead, logs the result to Airtable, and instantly notifies the sales team on Slack and Telegram for hot and warm leads.

---

## 1. Business Problem

Marketing agencies typically receive leads through a website contact form with no structure and no prioritization. Every submission — from a serious buyer with a defined budget to a spam bot or a casual tire-kicker — lands in the same inbox and gets the same treatment. This creates three concrete problems:

- **Slow response times.** Sales reps manually read every submission to decide if it's worth a reply, which delays outreach to genuinely hot leads.
- **No prioritization.** There is no consistent way to tell a high-intent prospect apart from a low-quality inquiry until a human reads it.
- **Fragmented records.** Leads live in an inbox instead of a structured, searchable database that the sales team can work from.

The result is missed revenue: hot leads go cold while the team is busy triaging noise.

## 2. Solution

This workflow automatically:

1. Captures every website form submission via a webhook.
2. Validates that the minimum required information is present.
3. Sends the lead's message to an AI model that extracts structured details (name, email, budget, service requested) and assigns a **seriousness score (1–10)**.
4. Classifies the lead into a tier — **Hot**, **Warm**, or **Cold** — based on that score.
5. Saves the fully structured lead record to Airtable as the single source of truth.
6. Instantly notifies the team in **Slack** and **Telegram** for Hot and Warm leads, so the highest-value prospects get a fast human follow-up.

This turns an unstructured inbox problem into a structured, scored, and routed pipeline — with zero manual triage.

## 3. Architecture

```
┌─────────────────┐
│  Website Form    │
│  (HTML/JS POST)  │
└────────┬─────────┘
         │ POST JSON { name, email, phone, message }
         ▼
┌─────────────────────┐
│   Form Webhook       │  n8n-nodes-base.webhook
│   /lead-form-qualify │
└────────┬─────────────┘
         ▼
┌─────────────────────┐
│  Normalize Lead Data │  Code node
│  (map + trim fields) │
└────────┬─────────────┘
         ▼
┌─────────────────────────┐
│  Validate Required       │  IF node
│  Fields (name + message) │
└────┬─────────────────┬───┘
     │ valid            │ invalid
     ▼                  ▼
┌───────────────┐   ┌────────────────────────┐
│ AI Qualify     │   │ Respond - Missing Info  │
│ Lead (OpenAI)  │   │ HTTP 400                │
└────────┬───────┘   └────────────────────────┘
         ▼
┌─────────────────────┐
│  Parse AI Output      │  Code node
│  (score → tier logic) │
└────────┬─────────────┘
         ▼
┌─────────────────────┐
│   Save to Airtable    │  Create record
└────────┬─────────────┘
         ▼
┌─────────────────────┐
│  Route by Lead Score  │  Switch node (Hot / Warm / Cold)
└───┬───────────┬───────┘
    │ Hot/Warm   │ Cold
    ▼            ▼
┌─────────────────────┐   ┌──────────────────┐
│  Notify Slack +      │   │  Respond Success   │
│  Notify Telegram     │   │  (no notification) │
└────────┬─────────────┘   └──────────────────┘
         ▼
┌─────────────────────┐
│   Respond Success     │  HTTP 200
└─────────────────────┘
```

**Design principle:** the workflow always responds to the webhook caller (either a 400 on invalid input or a 200 on success), and every lead — regardless of tier — is written to Airtable. Only Hot and Warm leads trigger real-time human notifications, keeping Slack/Telegram signal-to-noise high.

## 4. Workflow Breakdown

| # | Node | Type | Responsibility |
|---|------|------|-----------------|
| 1 | **Form Webhook** | `webhook` | Receives the POST request from the website form at `/lead-form-qualify`. Response is deferred to a `Respond to Webhook` node later in the flow. |
| 2 | **Normalize Lead Data** | `code` | Reads the raw body, maps loosely-named fields (`name`/`full_name`, `phone`/`whatsapp`, `message`/`details`/`description`) into a consistent schema, trims whitespace, and stamps `received_at` and `source: "form"`. |
| 3 | **Validate Required Fields** | `if` | Ensures `contact_name` and `raw_message` are both non-empty before spending an AI call on the lead. |
| 4a | **AI Qualify Lead** | `httpRequest` | On valid input, calls the OpenAI Chat Completions API (`gpt-4o-mini`) with a system prompt instructing it to return a strict JSON object: `contact_name`, `contact_email`, `budget`, `service_requested`, `seriousness_score` (1–10), `seriousness_reason`. |
| 4b | **Respond - Missing Info** | `respondToWebhook` | On invalid input, immediately returns `HTTP 400` with an explanatory error message, short-circuiting the rest of the flow. |
| 5 | **Parse AI Output** | `code` | Parses the AI's JSON response (with a fallback if parsing fails), merges it with the normalized lead data, and derives `lead_tier` from `seriousness_score`: **hot** (≥8), **warm** (≥5), **cold** (<5). |
| 6 | **Save to Airtable** | `airtable` | Creates one record per lead in the configured Base/Table, writing Name, Email, Phone, Budget, Status (tier), Score, service_requested, seriousness_reason, and Notes (raw message). |
| 7 | **Route by Lead Score** | `switch` | Branches on the Airtable record's `Status` field into three outputs: `Hot`, `Warm`, `Cold`. |
| 8a | **Notify Slack** | `slack` | Posts a formatted alert (name, email, phone, service, budget, score, reasoning) to a Slack channel — fires for both Hot and Warm branches. |
| 8b | **Notify Telegram** | `telegram` | Sends the same formatted alert to a Telegram chat via bot — fires for both Hot and Warm branches. |
| 9 | **Respond Success** | `respondToWebhook` | Returns `HTTP 200` with `{ status: "received", lead_tier }` back to the website form, regardless of tier. |

## 5. Tools & Tech Stack

- **Orchestration:** [n8n](https://n8n.io) (self-hosted or cloud)
- **AI Model:** OpenAI `gpt-4o-mini` via the Chat Completions API (`response_format: json_object` for guaranteed structured output)
- **Database / CRM layer:** Airtable
- **Team notifications:** Slack (OAuth2) + Telegram Bot API
- **Ingress:** n8n Webhook node (POST, JSON body) — pluggable into any HTML/JS website form
- **Authentication:** Header Auth (Bearer token) for OpenAI, OAuth2 for Slack, Bot API token for Telegram, Personal Access Token for Airtable

## 6. Data Schema

### Incoming payload (from website form)

```json
{
  "name": "string (or full_name)",
  "email": "string",
  "phone": "string (or whatsapp)",
  "message": "string (or details / description)"
}
```

### Normalized lead object (post `Normalize Lead Data`)

```json
{
  "source": "form",
  "contact_name": "string",
  "contact_email": "string",
  "contact_phone": "string",
  "raw_message": "string",
  "received_at": "ISO 8601 timestamp"
}
```

### AI qualification output (from OpenAI)

```json
{
  "contact_name": "string | null",
  "contact_email": "string | null",
  "budget": "string",
  "service_requested": "string",
  "seriousness_score": "integer (1-10)",
  "seriousness_reason": "string"
}
```

### Final Airtable record schema

| Field | Type | Source |
|---|---|---|
| `Name` | string | AI-extracted, falls back to form input |
| `Email` | string | AI-extracted, falls back to form input |
| `Phone` | string | Form input |
| `Budget` | string | AI-extracted |
| `service_requested` | string | AI-extracted |
| `Score` | number | AI-extracted (1–10) |
| `seriousness_reason` | string | AI-extracted |
| `Status` | option (`hot` / `warm` / `cold`) | Derived from `Score` |
| `Notes` | string | Raw lead message |

## 7. Edge Cases & Error Handling

- **Missing required fields:** If `contact_name` or `raw_message` is empty after normalization, the workflow short-circuits and responds `HTTP 400` with a descriptive error, without calling the AI model — this avoids wasted API spend on unusable submissions.
- **Malformed AI response:** `Parse AI Output` wraps the JSON parse in a try/catch. If the model returns non-JSON or fails to follow the schema, the lead is not dropped — it's saved with safe defaults (`seriousness_score: 0`, `service_requested: "unclear - AI parse failed"`) and routed to **Cold**, ensuring no lead is silently lost.
- **Loosely-named form fields:** `Normalize Lead Data` tolerates multiple possible key names (`name`/`full_name`, `phone`/`whatsapp`, `message`/`details`/`description`) so the workflow isn't brittle to minor changes in the website form's field names.
- **Every lead is persisted:** Regardless of tier or AI failure, every valid submission is written to Airtable — Slack/Telegram notifications are the only thing gated by score, not the record itself.
- **Consistent webhook response:** The caller always receives a JSON response (`400` on invalid input, `200` on success) so the frontend form can reliably show a confirmation or error state to the user.
- **Known limitation — Cold branch condition:** The `Route by Lead Score` switch node's "Cold" rule currently checks `{{ $json.fields['Attachment Summary'].state }} === 'cold'` instead of `{{ $json.fields.Status }} === 'cold'`. Since `Attachment Summary` is not a populated field on this record, cold leads will not match the intended condition and may fall through unrouted. **Action item:** update the Cold rule's `leftValue` to `{{ $json.fields.Status }}` to match the Hot/Warm rules before going live.

## 8. Testing & Validation

Recommended validation steps before activating the workflow in production:

1. **Happy path:** Submit a form payload with a clear, detailed message and a budget mentioned (e.g. *"We need a full paid social campaign, budget around $5,000/month, want to start next month"*). Confirm it lands in Airtable as `hot` and triggers both Slack and Telegram alerts.
2. **Warm path:** Submit a vaguer message with some intent but no budget/timeline. Confirm it's tagged `warm` and still notifies the team.
3. **Cold path:** Submit a low-intent or generic message (e.g. *"just curious what you do"*). Confirm it's tagged `cold`, is saved to Airtable, and does **not** trigger Slack/Telegram — then verify the Cold-branch fix noted in section 7.
4. **Missing fields:** Submit a payload with an empty `name` or `message`. Confirm the webhook returns `HTTP 400` and no OpenAI call is made (check n8n execution log / OpenAI usage dashboard).
5. **Malformed AI output:** Temporarily point `AI Qualify Lead` at a mock endpoint returning invalid JSON to confirm the fallback in `Parse AI Output` saves a `cold`/score-0 record instead of failing the execution.
6. **Field-name variance:** Submit a payload using `full_name`/`whatsapp`/`details` instead of `name`/`phone`/`message` to confirm normalization still works.
7. **Load / duplicate check:** Submit several leads in quick succession to confirm Airtable records are created without collisions and both notification channels stay in sync.
8. **Use the n8n Test URL** first for all of the above, then switch to the Production URL and activate the workflow only after all cases pass.

## 9. Screenshots / Demo Placeholder

- [ ] <img width="1877" height="777" alt="image" src="https://github.com/user-attachments/assets/1c7e190f-7f92-42da-a366-8594ec9a9085" />
 
- [ ] <img width="1272" height="801" alt="image" src="https://github.com/user-attachments/assets/c78fc30e-1b23-4a02-bc23-f155059e6c98" />

- [ ] <img width="747" height="199" alt="image" src="https://github.com/user-attachments/assets/352f7d21-dd64-44c6-978f-108f1bb3a3cb" />

- [ ] <img width="633" height="217" alt="image" src="https://github.com/user-attachments/assets/5a5f0620-44c2-4160-b06b-f7684fec6098" />

- [ ] <img width="549" height="190" alt="image" src="https://github.com/user-attachments/assets/564d3d8b-0324-4fd7-b857-48630745c30b" />

---

**Version:** v1 (Website Form source only) · **Status:** In development — WhatsApp (Meta Cloud API) and Meta Lead Ads sources to be added on the same normalized schema.
