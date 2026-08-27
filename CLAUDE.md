# Lead Generation System

This project turns a target-customer profile into a ranked, qualified lead list, then
enriches only the leads the user chooses to pursue with verified contact info.

You (Claude Code) are the operator. Two MCP servers are connected:

- **tavily** — web search + extraction. Used for discovery, research, and signal detection.
- **prospeo** — B2B people/company data. Used ONLY for contact enrichment, and ONLY on
  leads the user has explicitly selected.

Follow the rules in this file on every run so results stay consistent between runs (and
between takes when recording). If the user asks you to change the ICP or rubric, edit this
file directly.

---

## 1. Ideal Customer Profile (ICP)

> **EDIT THIS SECTION.** These are the values every `/find-leads` run targets. The example
> below is a placeholder — replace it with your real target customer.

- **What they do / industry:** B2B SaaS companies selling sales or marketing software
- **Company size:** 20–200 employees
- **Geography:** Canada and United States
- **Tech / other signals:** Uses HubSpot or Salesforce; has an active outbound sales team
- **Who we sell to (target role):** Head of RevOps, VP Sales, or Head of Marketing
- **What we offer (for the outreach angle):** [one line — what problem you solve for them]

### Disqualifiers (never include these)
- Enterprises over 1,000 employees
- Agencies and consultancies (unless the user overrides)
- Companies with no discoverable website

---

## 2. Ranking rubric — Fit × Intent

Score every candidate on two independent axes, then assign a tier. Always attach a short
plain-language reason and the specific trigger. Tiers, not raw numbers — they read cleaner
and map directly to "who do I chase first."

**Fit** — how well the company matches the ICP above (industry, size, geo, tech):
- **High** — matches the ICP on all major dimensions
- **Medium** — matches most, misses one (e.g. right space, slightly large)
- **Low** — weak match; borderline in-scope

**Intent** — is there a live buying signal / trigger event right now:
- **High** — clear recent trigger (hiring for the target role, funding round, product
  launch, publicly stated pain, leadership change, expansion)
- **Medium** — softer/older signal, or growth indicators without a specific event
- **Low** — no discoverable trigger

**Tier (the output that matters):**
- **Tier A — pursue now:** High fit + (High or Medium intent)
- **Tier B — nurture / selective:** High fit + Low intent, OR Medium fit + High intent
- **Tier C — deprioritize:** everything else

---

## 3. Output schema

Write results to `leads/leads-YYYY-MM-DD.csv` (create if missing; append within a day's run).
Also write a human-readable `leads/leads-YYYY-MM-DD.md` summary grouped by tier for on-screen
review. Columns, in order:

`id, company, domain, hq_location, employee_range, industry, fit, fit_reason, intent, trigger, tier, suggested_angle, source_urls`

- `id` — short stable slug, e.g. `L001`, `L002` (used by `/enrich`)
- `trigger` — the specific signal in a few words, or `none found`
- `suggested_angle` — one sentence: how to open outreach given the trigger
- `source_urls` — 1–3 URLs backing the fit/trigger

Enrichment (`/enrich`) later appends these columns to the same rows:

`contact_name, contact_title, work_email, email_confidence, enriched_at`

---

## 4. Discovery rules (`/find-leads`)

1. Read the ICP and rubric above. Fold in any refinement passed as command arguments.
2. Use **tavily search at `basic` depth** for discovery — it's 1 credit per search. Do NOT
   use `advanced` depth or `extract` for broad discovery; reserve those for researching a
   specific promising company.
3. Cast a wide net: run several angled queries (by sub-segment, by signal, by geo) rather
   than one broad query. Cap `max_results` sensibly (~10) per query.
4. For each surviving candidate, run one signal check using tavily with a recent
   `time_range` (news/last month) to find a trigger event.
5. Score with the rubric, assign a tier, write the reasoning and trigger.
6. Write the CSV + the grouped markdown summary. Print a short tier-grouped table to the
   terminal so the user can review and pick.
7. **Do NOT call prospeo during discovery. Do NOT fetch or reveal any personal contact
   info here.** Discovery produces companies + reasoning only.

---

## 5. Enrichment rules (`/enrich <lead-ids>`) — Option B

Only runs on IDs the user explicitly passes (e.g. `/enrich L001 L004 L009`). This human gate
is the cost control — never enrich the whole list automatically.

1. First call the prospeo **account-info** tool (e.g. `get_account_info`) — this is FREE and
   spends no credits. Show the user credits remaining before spending anything.
2. If remaining credits are low (under ~10), warn and stop rather than partially enriching.
3. For each selected lead, use the prospeo **people-search** tool (e.g. `search_person`) with:
   - the lead's **company domain**, and
   - the **target role / seniority filter** from the ICP (Section 1).
   Take the single best-matching decision-maker. This one call returns the person AND their
   verified work email (Option B — let Prospeo find the person).
4. Default to **one contact per company** and **work email only**. Do NOT request mobile
   numbers — they cost ~10 credits each versus ~0.5 for an email find.
5. Append `contact_name, contact_title, work_email, email_confidence, enriched_at` to that
   lead's row. Leave un-selected leads untouched.
6. If a tool name differs from the examples above, list the connected prospeo tools first and
   use the closest match — don't guess blindly.

---

## 6. Cost guardrails (keep both tools in their free tiers)

- Tavily free tier: 1,000 credits/month. Basic search = 1 credit. A full discovery run of
  ~50 candidates should cost well under 100 credits.
- Prospeo free tier: 75 credits/month; an email find is ~0.5 credit, so ~150 lookups. Only
  ever spent on user-selected leads.
- **Cache raw tool results to `.cache/`** keyed by query, and reuse them within a run and on
  re-runs. Re-shooting a take must not re-spend credits — check the cache before every call.
- Never batch-enrich. Never pull mobile numbers unless the user explicitly insists.

---

## 7. Compliance note

This system finds business/firmographic data and work contacts for B2B outreach. Keep it to
the company and professional-role level. Real individual contact details are personal data —
handle outreach under a lawful basis with a clear opt-out (GDPR legitimate interest / CASL /
CAN-SPAM as applicable). Not legal advice.
