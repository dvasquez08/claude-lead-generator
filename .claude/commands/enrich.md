---
description: Enrich ONLY the selected leads with a verified work email (Option A — ~1 credit each)
argument-hint: <lead-ids, e.g. L001 L004 L009>
allowed-tools: mcp__tavily__*, mcp__prospeo__*, Read, Write, Edit
---

Enrich only the leads I explicitly list here — one verified work email per company, at a
predictable cost of ~1 Prospeo credit per lead. Follow these steps exactly.

Leads to enrich: **$ARGUMENTS**

**Why this order matters (cost):** a Prospeo email FIND costs 1 credit, but a domain/people
SEARCH can return several people and bill 1 credit for EACH verified email — that's how a
3-lead run can cost 9 credits. To stay at ~1 credit per lead, identify the specific
decision-maker with Tavily FIRST, then do a single direct Prospeo email find for that one
named person. Do NOT use Prospeo's domain/people search in this command.

Steps:

1. If no IDs were given above, stop and ask me which lead IDs to enrich. Never enrich the
   whole list.
2. Call the Prospeo account-info tool first (free, spends no credits) and show me credits
   remaining. If under ~10, warn me and stop.
3. Load today's `leads/leads-<today>.csv` and select only the rows whose `id` I listed.
4. For each selected lead, in this order:
   a. **Identify the person with Tavily (no Prospeo yet).** Search for the single
   decision-maker in the target role from the ICP (`CLAUDE.md` Section 1) at that company
   — e.g. the owner, GM, or operations manager. Get their full name and confirm the
   company domain from public sources. Cache Tavily results to `.cache/`.
   b. If you cannot confidently identify a named person, record `no decision-maker found` for
   that lead and DO NOT call Prospeo for it — moving on costs nothing; guessing wastes
   credits.
   c. **Find the email with Prospeo** — call the direct email-FINDER tool (the one that takes
   a person's first name + last name + company domain), once, for that single person. This
   is the 1-credit operation. Do NOT call any domain-search / people-search tool. If tool
   names are unclear, list the connected Prospeo tools first and pick the direct finder.
   d. Work email only. One contact per company. Never request mobile numbers (10 credits each).
5. Append `contact_name, contact_title, work_email, email_confidence, enriched_at` to those
   rows only. Leave every un-selected lead untouched.
6. Report, per lead: the contact found (or `no email found`), plus the running Prospeo credit
   total so I can confirm it stayed near 1 credit per lead. Prospeo only charges for a
   verified result, so misses should cost nothing.
