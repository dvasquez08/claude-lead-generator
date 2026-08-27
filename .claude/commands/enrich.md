---
description: Enrich ONLY the selected leads with a verified work contact via Prospeo (Option B)
argument-hint: <lead-ids, e.g. L001 L004 L009>
allowed-tools: mcp__prospeo__*, Read, Write, Edit
---

Enrich only the leads I explicitly list here with verified contact info, following the
enrichment rules in `CLAUDE.md` Section 5.

Leads to enrich: **$ARGUMENTS**

Steps:

1. If no IDs were given above, stop and ask me which lead IDs to enrich. Never enrich the
   whole list.
2. Call the prospeo **account-info** tool first (this is free, spends no credits) and show me
   credits remaining. If it's under ~10, warn me and stop instead of partially enriching.
3. Load today's `leads/leads-<today>.csv` and select only the rows whose `id` I listed.
4. For each selected lead, use the prospeo **people-search** tool with the lead's **company
   domain** plus the **target role / seniority filter** from the ICP (`CLAUDE.md` Section 1).
   Take the single best-matching decision-maker; that call returns the person and their
   verified work email (Option B).
   - One contact per company. **Work email only — never request mobile numbers.**
   - If the prospeo tool names differ from what's described, list the connected prospeo tools
     first and use the closest match.
5. Append `contact_name, contact_title, work_email, email_confidence, enriched_at` to those
   rows only. Leave every un-selected lead untouched.
6. Report what was enriched and the running credit total, and note any lead where no verified
   email was found (don't spend extra credits guessing).
