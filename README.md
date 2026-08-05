# AI Lead Qualification & Routing System

An n8n automation that scores and routes inbound leads using a single AI judgment call, with human approval gates before anything risky — creating a deal, or disqualifying a real lead — actually happens.

Built as a portfolio project for **Northlane Digital**, a fictional B2B marketing/web-design agency, to demonstrate "judgment-layer automation": mostly deterministic workflow logic, one real AI decision point, and a human in the loop wherever a wrong call would actually cost something.

---

## The Problem

> "We get maybe 40-60 leads a month across our website form, LinkedIn ads, and referrals, but half of them are students doing research, freelancers who can't afford us, or totally the wrong industry. My sales guy spends his morning just figuring out who's even worth calling, and by the time he gets to the good ones, they've already booked with a competitor. I don't need more leads — I need someone to tell me which 10 out of 60 actually matter, and put them straight in front of him."

---

## How It Works

**1. Intake** — `Webhook` receives a Webflow form submission. *(Architecture also supports Calendly/Typeform and LinkedIn Lead Ads as additional triggers feeding the same pipeline — documented, not built into this demo, since the downstream logic is identical regardless of source.)*

**2. Normalize** — `Edit Fields` maps the incoming payload into one consistent shape (name, email, company, message, source) so everything downstream can rely on a predictable structure.

**3. Duplicate check** — `Search records` looks up the lead's email in Airtable. `Duplicate Detection` + `Duplicate Check` branch on whether it's a repeat submission:
   - **Duplicate found** → `Send a message` notifies the team instead of re-running the full pipeline on someone already in the system.
   - **New lead** → continues to enrichment.

**4. Enrichment (deterministic)** — `Lead Enrichment` pulls in cheap, code-level signals (business vs. free email domain, company info) *before* any AI call — so the AI isn't reasoning from bare form text alone, and tokens aren't spent on data a simple lookup can already provide.

**5. AI decision point** — `Basic LLM Chain` (Google Gemini) + `Structured Output Parser` analyzes the enriched lead and returns strict, validated JSON:
   - `industry_guess`
   - `budget_signal`
   - `intent_level`
   - `lead_score` (0–100)
   - `disqualify_reason`
   - `suggested_next_step`

   This is the **only** AI decision point in the workflow. It classifies and scores — it does not draft outreach, does not decide what happens next, and does not touch the CRM. That's left to deterministic logic and a human.

**6. Deterministic routing** — `Hot Lead Check` and `Warm Lead Check` branch on `lead_score`:
   - **≥ 70 → Hot**
   - **40–69 → Warm**
   - **< 40 → Disqualified**

**7. Human approval gates** — placed only on the two lanes where a wrong AI call is actually costly:
   - **Hot lane:** `Send message and wait for response` pauses the workflow and asks a rep to approve in Slack. `If1` reads the response — **Approved** → `Create Hot Lead` → `Send a message1` (team notified). **Rejected** → routed into `Create Warm Lead` instead of being lost.
   - **Disqualified lane:** `Send message and wait for response1` asks for confirmation before writing off a lead. `If` reads the response — **Approved** → `Create Disqualified Lead` → `Send a message2`. **Rejected** → also routed back into `Create Warm Lead` for a second look.
   - **Warm lane has no gate.** A wrong medium-confidence call costs little, so adding a human checkpoint here would just be approval-theater — it flows straight to `Create Warm Lead` → `Slack Alert – Warm Lead`.

**8. Output** — every path ends in both an Airtable record and a Slack notification, so nothing is created or closed silently, and nothing quietly disappears.

---

## Architecture

<img width="1221" height="651" alt="Screenshot 2026-08-01 at 1 58 27 AM" src="https://github.com/user-attachments/assets/8e992593-427f-4f01-b86e-b32350ff94aa" />

```
Webhook → Edit Fields → Search Records → Duplicate Check
                                              ├── Duplicate → Notify
                                              └── New → Lead Enrichment → AI Scoring (Gemini + Structured Output Parser)
                                                                              └── Score Routing
                                                                                    ├── Hot (≥70)   → Human Approval → Approved: Create Deal + Notify   | Rejected: → Warm
                                                                                    ├── Warm (40-69) → Create Record + Notify (no gate)
                                                                                    └── Disqualified (<40) → Human Approval → Approved: Mark Closed + Notify | Rejected: → Warm
```

---

## Key Design Decisions

- **AI does one job: classify and score.** It never drafts outreach, never touches the CRM directly, and never makes the final call on its own — that separation is what keeps this "judgment-layer," not "autonomous."
- **Enrichment happens before the AI call, not instead of it.** Cheap, deterministic signals (email domain type, company lookup) are gathered first so the AI has real context and isn't burning tokens re-deriving what code can check directly.
- **Approval gates are placed by cost of error, not by category.** Hot and Disqualified both get a human checkpoint because a wrong call there is expensive (wasted sales time, or a real client turned away). Warm gets none, because a wrong mid-range score just means a lead sits in nurture a little longer — adding a gate there would slow the system down for no real benefit.
- **Rejections don't delete data.** If a human overrides the AI in either direction, the lead is re-filed into Warm rather than discarded — giving it a second look instead of a dead end.
- **Duplicate detection runs before the AI call**, so the same lead resubmitting (e.g., filling the form, then also booking a call) doesn't get scored twice or clutter the pipeline.
- **Error handling is a separate workflow** (`workflows/error-handler.json`), triggered by n8n's Error Trigger and posting failures to Slack — kept isolated so main-pipeline logic stays readable.

---

## Tools Used

- **n8n** — workflow orchestration
- **Google Gemini** — AI scoring/classification (via Basic LLM Chain + Structured Output Parser)
- **Airtable** — lead database (Hot / Nurture / Disqualified views)
- **Slack** — approval gates (Send and Wait for Approval) and team notifications
- **Webflow** (demo trigger) — with Calendly/Typeform and LinkedIn Lead Ads as documented, drop-in alternate sources

---

## Repo Contents

```
workflows/
├── ai-lead-qualification-routing.json   → main workflow, importable directly into n8n
└── error-handler.json                   → error-handling workflow
docs/
└── project-brief.md                     → full project brief (problem, prompt/schema, node list, demo shot list)
```

To use: import either JSON file via n8n's **Workflows → Import from File**, then reconnect your own Airtable base, Slack workspace, and Gemini/OpenAI credentials — credentials are never included in the export.

---

---

## Note

This is a portfolio project built around a fictional company (Northlane Digital) and fictional lead data, created to demonstrate n8n + AI workflow design for freelance/agency work. Not built on or affiliated with any real client's data.
