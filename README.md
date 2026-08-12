AI Career Copilot

An automated job-discovery and screening pipeline that scrapes job listings, scores each one against a candidate profile using Claude (Anthropic's AI model), and routes the results into a spreadsheet, a CRM pipeline, and a real-time Telegram notification — with zero manual review of raw listings.

Built as a personal tool to solve a real problem: manually browsing job boards is slow and noisy. This pipeline turns "check LinkedIn every day" into "get pinged only when something is actually worth looking at."

Why this project

Most job search tooling stops at aggregation — it shows you more listings, not better ones. This project's core bet: filtering and scoring should happen before a human ever sees a listing, not after. That reframes the problem from "browse efficiently" to "build a funnel," which is the same problem growth and lifecycle teams solve for lead qualification — just pointed at job postings instead of leads.

The project is designed to demonstrate, in one working system:

API integration across multiple third-party services
Business-process automation design
CRM pipeline modeling
Applied use of AI for structured decision support
Data-flow architecture
Version control and technical documentation
Shipping something to a genuinely working, daily-usable state — not just a prototype
What it does (current build)
Collects job listings from LinkedIn on a schedule (via an Apify scraping actor).
Filters out irrelevant roles by category and recency before any expensive processing happens.
Scores each remaining listing with Claude: fit score (0–100), matching skills, skill gaps, a tailored application recommendation, and a plain-language summary — structured as JSON, not free text.
Logs every scored listing to a spreadsheet for a full audit trail.
Creates a CRM deal card for each listing, so the application process (applied → response → interview → offer) can be tracked like a sales pipeline — because that's functionally what a job search is.
Notifies via Telegram in real time, so the loop from "posted" to "I know about it" is minutes, not days.
Architecture
Apify (LinkedIn scraper) — runs the scrape, then fires a webhook on run completion
Zapier — fetch the full result set from the scrape
Zapier — split the result set into individual listings
Zapier — filter: company tier, category match, recency, duplicates (only listings that pass move on)
Claude API — structured scoring (JSON in / JSON out)
Parse + normalize the model's response (JavaScript)
Fan out to three destinations, in parallel, for each scored listing:
Google Sheets — audit log
HubSpot Deal — pipeline card
Telegram alert — real-time ping

Stack: Apify (scraping), Zapier (orchestration), Claude API / Anthropic (scoring), Google Sheets (logging, MVP stage), HubSpot (pipeline CRM), Telegram Bot API (notifications), GitHub (version control).

Component roles

Apify — sourcing, scraping, and cleaning listing data; hands off structured JSON to the orchestration layer.

Zapier — the orchestrator. Triggers automations, filters listings, calls external APIs, passes data between services, and handles step-level error states.

Claude API — performs the actual listing analysis. Returns, per listing: a fit score, matching skills, missing skills, a tailored recommendation, and a plain-language summary. (Planned: cover-letter drafts, see Roadmap.)

HubSpot CRM — every listing that clears the filter becomes a deal card, with fields for company, role, salary, location, source, URL, AI score, matching/missing skills, status, notes, and next action.

Telegram — sends a notification only for listings that clear both the relevance filter and the scoring threshold, e.g.:

New vacancy

Python Backend Developer
Example Inc.

Match: 91%
Recommended to apply.
CRM pipeline
New → Matches → Application Sent → Employer Response →
Interview → Test Task → Final Interview → Offer

Additional statuses: Not a Fit · Rejected · Closed · On Hold

Example output

A listing that clears the filter produces something like:

json
{
  "ai_score": 62,
  "matching_skills": ["influencer relationship management", "campaign management", "communication skills"],
  "missing_skills": ["paid media / performance marketing", "e-commerce marketing experience"],
  "recommendation": "Moderate fit. Lead with concrete influencer-negotiation case studies and campaign metrics when applying.",
  "summary": "Full-cycle influencer marketing role — sourcing, campaign management, and content strategy — at a fast-growing company with fairly general requirements."
}

...which becomes a spreadsheet row, a CRM card in the "New" stage, and a Telegram message, automatically, within a couple of minutes of the listing appearing on LinkedIn.

Design decisions worth calling out
Filter before scoring, not after. Model calls are the most expensive step in the pipeline (in both latency and cost). Category, recency, and company-tier filtering happens before any listing reaches Claude, so cost scales with relevant listings, not total listings scraped.
Structured output, not prose. The model is prompted to return strict JSON (ai_score, matching_skills, missing_skills, recommendation, summary) — no markdown, no commentary — so downstream steps can parse it deterministically instead of regex-scraping free text.
Exact-match company filtering, not substring matching. The company allowlist filter normalizes casing, punctuation, and corporate suffixes (Inc., Corp., LLC, etc.) and then requires an exact match — not .includes(). Substring matching produced false positives (e.g. "Apple Valley Insurance" matching "Apple", "Nike Town Retail" matching "Nike"); exact match trades a few missed edge cases for avoiding wrong matches, which matters more for this filter's purpose.
The CRM stage mirrors a sales funnel on purpose. Job applications behave like a pipeline with drop-off at every stage (applied → response → interview → offer). Modeling it as CRM deal stages makes conversion visible instead of anecdotal.
Notification is a filter, not a feed. The Telegram channel only receives listings that already passed the scoring threshold — it's designed to be checked in seconds, not scrolled.
Status

Currently a working v1 (MVP), running on personal infrastructure and used daily for an active job search. MVP scope: GitHub repo + README, Apify source, one Zap pipeline, AI scoring, HubSpot CRM, Telegram notifications, structured AI Score output.

Roadmap

v2.0

Multiple job-board sources, not just LinkedIn
Salary analysis across listings
Skill-development recommendations based on recurring gaps
Auto-drafted cover letters
Interview-prep question generation
Weekly summary reports (CRM → AI → Telegram)
Applicant funnel analytics (conversion rate by stage)

v3.0

Web dashboard with authentication
Candidate profile management
Resume parsing and profile-based matching
Historical application statistics and data export
Known gaps in the current build
Duplicate detection — not yet implemented; planned via a unique listing ID check (e.g. against Zapier Storage) before a listing re-enters the pipeline.
Company-tier filtering — an allowlist-based filter (Fortune Global 500 / Forbes Global 2000 / Forbes Top 100 Digital Companies) exists as a standalone script but isn't yet wired into the live pipeline.
Cost-aware orchestration — per-task platform pricing (Zapier) scales worse than direct API usage at higher listing volumes; evaluating a move to a lightweight scheduled script for orchestration as usage grows.

Built solo, end to end — scraping config, workflow orchestration, prompt design, filter logic, and CRM/notification integration.
