# AI Lead Qualification & Routing System

An n8n automation that scores and routes inbound leads using a single AI 
judgment call, with human approval gates before anything risky (creating a 
deal, disqualifying a real lead) actually happens.

## The Problem
[Northlane Digital problem statement from the brief]

## How It Works
[Trigger → Normalize → Duplicate check → Enrich → AI score → Branch → 
Human approval → Output — summarized, link to full brief in /docs]

## Architecture
[Paste the flowchart image or describe the flow]

## Key Design Decisions
- AI does one thing: score + classify. It doesn't draft outreach or make 
  the final call.
- Human approval sits only on the two lanes where a wrong AI call is 
  costly (hot, disqualified) — warm flows through automatically.
- Rejected leads route to Nurture instead of being discarded.
- Error handling is a separate workflow (see /workflows/error-handler.json).

## Tools Used
n8n, Google Gemini, Airtable, Slack

## Demo
[Link to your demo video once recorded]

## Note
Built as a portfolio project with fictional company data (Northlane Digital).
