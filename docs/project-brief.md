# Project Brief: AI Lead Qualification & Routing System

**Platform:** n8n | **AI task:** Structured extraction + scoring | **Target niche:** Agencies / B2B services

---

## 1. Fictional Company + Industry

**Northlane Digital** — a 14-person B2B marketing/web-design agency that runs paid ads and SEO for mid-market clients (roughly $2M–$50M revenue). They get inbound leads from a Webflow contact form, a "Book a Call" Calendly page, and a LinkedIn lead-gen ad campaign — three sources dumping into one shared inbox with no consistent triage.

---

## 2. Problem Statement (client's voice)

> "We get maybe 40-60 leads a month across our website form, LinkedIn ads, and referrals, but half of them are students doing research, freelancers who can't afford us, or totally the wrong industry. My sales guy spends his morning just figuring out who's even worth calling, and by the time he gets to the good ones, they've already booked with a competitor. I don't need more leads — I need someone to tell me which 10 out of 60 actually matter, and put them straight in front of him."

---

## 3. Trigger

**Architecture supports multiple sources; the built workflow runs on one.**
- **Built trigger:** Webflow form submission → `Webhook` (instant).
- **Documented, not built:** Calendly/Typeform "Book a Call" webhook, and LinkedIn Lead Gen Form via scheduled polling (LinkedIn doesn't reliably push webhooks for lead forms outside their Marketing Partner program). Both are drop-in additions — same normalize step, same AI node, just a different trigger feeding it.

---

## 4. Full Step-by-Step Workflow

1. **Trigger** — `Webhook` fires on form submission.
2. **Normalize** — `Edit Fields` maps the incoming payload into one common object: `{name, email, company, message, source, submitted_at}`.
3. **Duplicate check** — `Search records` (Airtable) checks whether this email already exists.
   - `Duplicate Detection` + `Duplicate Check` branch on the result:
     - **Duplicate found →** `Send a message` notifies the team instead of re-scoring someone already in the system.
     - **New lead →** continues to enrichment.
4. **Enrichment (deterministic)** — `Lead Enrichment` pulls in cheap, code-level signals (business vs. free email domain, company info) before any AI call.
5. **AI decision point** — `Basic LLM Chain` (Google Gemini) + `Structured Output Parser` returns strict JSON: `industry_guess`, `budget_signal`, `intent_level`, `lead_score`, `disqualify_reason`, `suggested_next_step`.
6. **Deterministic routing** — `Hot Lead Check` and `Warm Lead Check` branch on `lead_score`:
   - **≥ 70 → Hot**
   - **40–69 → Warm**
   - **< 40 → Disqualified**
7. **Human approval gates:**
   - **Hot lane:** `Send message and wait for response` pauses and asks a rep to approve in Slack. `If1` reads the result — **Approved →** `Create Hot Lead` → `Send a message1`. **Rejected →** routed into `Create Warm Lead` instead of being lost.
   - **Disqualified lane:** `Send message and wait for response1` asks for confirmation before writing off a lead. `If` reads the result — **Approved →** `Create Disqualified Lead` → `Send a message2`. **Rejected →** also routed to `Create Warm Lead`.
   - **Warm lane:** no gate — flows straight to `Create Warm Lead` → `Slack Alert – Warm Lead`.
8. **Output** — every path ends in both an Airtable record and a Slack notification.

---

## 5. Exact AI Prompt/Schema at the Decision Point

**System prompt:**
```
You are a lead qualification analyst for a B2B marketing/web-design agency called Northlane Digital. 
Northlane's ideal client is a company with 10-250 employees, an identifiable marketing budget, 
and a clear need for paid ads, SEO, or web design help. Northlane does NOT serve individual 
freelancers, students, or non-business inquiries.

You will be given a lead's submitted information. Analyze it and return ONLY a JSON object — 
no preamble, no markdown formatting, no explanation outside the JSON.

Score conservatively. If information is missing or ambiguous, reflect that with a lower 
confidence and a mid-range score rather than guessing high.
```

**User prompt (templated with lead data):**
```
Lead data:
- Name: {{name}}
- Email: {{email}}
- Email domain type: {{business_or_free}}
- Company: {{company}}
- Company size (if known): {{company_size}}
- Source: {{source}}
- Message/notes submitted: "{{message}}"

Return a JSON object matching exactly this schema:
{
  "industry_guess": string,
  "budget_signal": "low" | "medium" | "high",
  "intent_level": "browsing" | "comparing" | "ready_to_buy",
  "lead_score": integer (0-100),
  "disqualify_reason": string | null,
  "suggested_next_step": string (max 20 words)
}
```

**Why this schema, not more:** it deliberately stops at "score + reason + next step." It does NOT draft outreach or make the routing decision itself — that's left to the deterministic IF nodes and the human approval gates. This is the line that keeps the system "judgment-layer" rather than autonomous: the AI classifies, the workflow and a human decide what happens next.

---

## 6. Exact Node/Module List (n8n)

1. `Webhook` — Webflow form submission
2. `Edit Fields` — normalize into common schema
3. `Search records` (Airtable) — duplicate check by email
4. `Duplicate Detection` + `Duplicate Check` (IF) — branch on duplicate found
5. `Send a message` (Slack) — duplicate-found notification
6. `Lead Enrichment` — deterministic enrichment (email domain type, company info)
7. `Basic LLM Chain` (Google Gemini Chat Model) — the AI scoring call
8. `Structured Output Parser` — validates the AI's JSON against the schema
9. `Hot Lead Check` (IF) — branch on score ≥ 70
10. `Warm Lead Check` (IF) — branch on score 40–69 vs. < 40
11. `Send message and wait for response` — hot-lane Slack approval gate
12. `If1` — reads the hot-lane approval result
13. `Create Hot Lead` (Airtable) → `Send a message1` (Slack)
14. `Create Warm Lead` (Airtable) → `Slack Alert – Warm Lead`
15. `Send message and wait for response1` — disqualified-lane Slack approval gate
16. `If` — reads the disqualified-lane approval result
17. `Create Disqualified Lead` (Airtable) → `Send a message2` (Slack)

*Separate workflow: `Northlane – Error Handler`, using n8n's `Error Trigger` → Slack alert, kept isolated from the main pipeline so it doesn't clutter the primary canvas.*

---

## 7. Third-Party Tools/APIs

- **Webflow** — live trigger (Calendly/Typeform + LinkedIn Ads API documented as drop-in alternates)
- **Google Gemini** — the AI scoring call (via Basic LLM Chain + Structured Output Parser)
- **Slack** — approval gates (Send and Wait for Response) + team notifications
- **Airtable** — lead database (Hot / Warm / Disqualified views), duplicate lookup

---

## 8. Human Approval Gate

Two gates, placed only where a wrong AI call is costly, both built on n8n's native **Send and Wait for Response** node:
1. **Hot lead confirmation** — a rep approves before the lead becomes an Airtable "deal" record and the team is notified. Rejecting re-files the lead into Warm.
2. **Disqualification approval** — someone confirms before a lead is marked closed. Rejecting also re-files into Warm, giving it a second look instead of discarding it.

**Warm leads flow automatically — no gate.** A wrong mid-range score costs little, so a checkpoint there would only slow the system down without protecting anything meaningful.

This gate placement — protecting the two lanes with real downside, leaving the safe middle alone — was tested live: a rejected hot lead was confirmed (via the node's execution output) to correctly land in the Warm branch rather than the Hot branch, proving the fork logic works, not just that it looks correct on the canvas.

---

## 9. Output/Deliverable

- A live Airtable base with Hot / Warm / Disqualified views, each record carrying the AI's reasoning (score, industry guess, budget signal, suggested next step) for auditability.
- Slack notifications on every outcome — approval requests, confirmations, duplicate flags — so nothing is created, closed, or skipped silently.
- A separate, isolated error-handling workflow so failures are caught and reported without cluttering the main pipeline logic.

---

## 10. Honest Estimated-Impact Line

This is a demo build, not a client's live data, so no fabricated stats. Framed honestly: *"For an agency triaging 40-60 inbound leads a month by hand, a system like this typically removes 60-90 minutes of daily manual sorting and — more importantly — gets hot leads in front of sales within minutes of submission instead of hours, which is usually where deals are actually won or lost."*

---

*See the repo's `workflow/` folder for the importable n8n JSON, and `assets/` for screenshots of the running system, including the Slack approval card and Airtable output views.*
