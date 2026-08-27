---
description: Discover, rank, and file qualified leads matching the ICP (no contact enrichment)
argument-hint: [optional refinement, e.g. "focus on Toronto fintech"]
allowed-tools: mcp__tavily__*, Read, Write, Edit, Bash
---

Generate a ranked lead list following the rules in `CLAUDE.md`. Do not ask me to restate the
ICP — read it from `CLAUDE.md`.

Refinement for this run (optional): **$ARGUMENTS**

Steps:

1. Read the ICP (Section 1) and ranking rubric (Section 2) from `CLAUDE.md`. Apply the
   refinement above on top of the ICP if one was given.
2. **Discovery** — use the tavily search tool at **basic** depth. Run several angled queries
   (by sub-segment, by geography, by buying signal) rather than one broad query. Cap results
   ~10 per query. Check `.cache/` before each search and reuse cached results; write new
   results to `.cache/`.
3. Drop anything hitting a disqualifier in the ICP.
4. **Signal detection** — for each surviving candidate, run one tavily search with a recent
   `time_range` to find a trigger event (hiring for the target role, funding, launch,
   leadership change, stated pain). Record the specific trigger or `none found`.
5. **Rank** — score Fit and Intent per the rubric, assign an A/B/C tier, and write a one-line
   `fit_reason`, the `trigger`, and a one-sentence `suggested_angle`.
6. **Write output** using the exact schema in `CLAUDE.md` Section 3:
   - `leads/leads-<today>.csv`
   - `leads/leads-<today>.md` — grouped by tier (A first), for on-screen review.
7. Print a compact tier-grouped table to the terminal: `id · company · tier · trigger`.

Hard rules:
- **Do NOT call prospeo.** No contact info, no emails, no people lookups in this command.
- Discovery is basic-depth only — no `advanced`/`extract` for the broad sweep.
- End by telling me how to enrich: e.g. `/enrich L001 L004` for the ones I want to pursue.
