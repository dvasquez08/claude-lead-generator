# Building an AI Lead-Generation System with Claude Code + Tavily + Prospeo

A step-by-step guide to building a lead-generation system that runs inside **Claude Code**,
using **Tavily** for research and **Prospeo** for verified contact enrichment — designed to
stay inside both tools' free tiers.

The clever part of the design: Claude Code does all the reasoning on your existing
subscription (a flat cost), so the only metered spend is Tavily + Prospeo credits — and
Prospeo is only ever touched on the specific leads you choose to pursue.

---

## How it works

```
/find-leads   ──►  Tavily discovery + signal detection + Fit×Intent ranking  ──►  ranked file
                                      │
                        you review the file and pick who's worth chasing
                                      │
/enrich L001 L004   ──►  Prospeo finds the decision-maker + verified email for those only
```

- **Discovery is cheap and broad** — Tavily basic search finds companies matching your Ideal
  Customer Profile (ICP), then a second pass looks for live buying signals (trigger events).
- **Ranking is the value-add** — every lead is scored on **Fit** (does it match the ICP) and
  **Intent** (is there a signal to act on now), then bucketed into A / B / C tiers.
- **Enrichment is gated** — Prospeo only runs on leads you explicitly select, which is what
  keeps you inside the free tier.

---

## What you'll need

- **Node.js** — only used so `npx` can launch the Prospeo MCP server. (Claude Code itself
  does not need Node.) Grab the LTS build from [nodejs.org](https://nodejs.org).
- **Claude Code** — the terminal agent that orchestrates everything.
- **A Tavily API key** — free tier, 1,000 credits/month — from [app.tavily.com](https://app.tavily.com).
- **A Prospeo API key** — free tier — from [prospeo.io](https://prospeo.io).

---

## Project structure

```
lead-gen-claude/
├── CLAUDE.md                     # the "brain": ICP, ranking rubric, output schema, rules
├── .mcp.json                     # Tavily + Prospeo MCP server config
├── .env.example                  # template for your API keys
├── .gitignore                    # keeps keys + generated lead data out of git
├── README.md
├── .claude/
│   └── commands/
│       ├── find-leads.md         # the discovery + ranking command
│       └── enrich.md             # the gated Prospeo enrichment command
├── leads/                        # generated lead files (gitignored)
└── .cache/                       # cached tool results (gitignored)
```

> **Windows note up front:** the `.mcp.json` file and the `.claude` folder both start with a
> leading dot. Windows Explorer frequently drops the dot (or won't let you create it), which
> silently breaks the project. See the [Troubleshooting](#troubleshooting-the-gotchas) section —
> this was the single biggest time-sink in the build.

---

## Setup, step by step

### 1. Get the project files in place

Put all the files above into a fresh project folder, keeping the structure intact. The
`.claude/commands/` folder and `.mcp.json` **must** sit at the project root.

If you're on Windows, create the dot-folder from the terminal rather than Explorer to avoid
the dropped-dot problem:

```powershell
mkdir .claude\commands
```

### 2. Install Node.js

Download the **LTS** version from [nodejs.org](https://nodejs.org), install, then verify in a
new terminal:

```
node --version
npm --version
```

Both printing a version means you're set. (Already have Node 18+? Skip the install.)

### 3. Install Claude Code

The **native installer** is the recommended method — it has no dependencies and auto-updates.

- **Windows (PowerShell):** `irm https://claude.ai/install.ps1 | iex`
  _(or via WinGet:_ `winget install -e --id Anthropic.ClaudeCode`_)_
- **macOS / Linux / WSL:** `curl -fsSL https://claude.ai/install.sh | bash`
- **Homebrew:** `brew install --cask claude-code`

**Then open a brand-new terminal** — the installer adds Claude Code to your PATH, which only
takes effect in a fresh shell session. Verify:

```
claude --version
```

> If `claude` is "not recognized" even after installing, it's almost always the PATH-needs-a-
> fresh-terminal thing. Open a new window. Still stuck? Run `claude doctor`.

### 4. Get your API keys

- Tavily → [app.tavily.com](https://app.tavily.com)
- Prospeo → [prospeo.io](https://prospeo.io)

### 5. Set your API keys as environment variables

The `.mcp.json` reads your keys from the environment via `${TAVILY_API_KEY}` and
`${PROSPEO_API_KEY}`, so they need to be set before you launch Claude Code.

**Quick / session-only** (keys last only for that one terminal window):

- PowerShell:
  ```powershell
  $env:TAVILY_API_KEY="tvly-your-key"
  $env:PROSPEO_API_KEY="your-key"
  ```
- Bash (Linux / macOS / Git Bash):
  ```bash
  export TAVILY_API_KEY="tvly-your-key"
  export PROSPEO_API_KEY="your-key"
  ```

**Permanent / recommended** (set once, every future terminal has them):

- Windows — `setx` (applies to **new** terminals only, not the current one):
  ```powershell
  setx TAVILY_API_KEY "tvly-your-key"
  setx PROSPEO_API_KEY "your-key"
  ```
- Linux / macOS — add to your shell profile:
  ```bash
  echo 'export TAVILY_API_KEY="tvly-your-key"' >> ~/.bashrc
  echo 'export PROSPEO_API_KEY="your-key"' >> ~/.bashrc
  source ~/.bashrc
  ```

> **Security tradeoff:** the permanent method stores keys in plaintext in your user profile.
> Fine on your own machine; avoid on shared or public computers. Either way, `.env` and any
> key-bearing files are gitignored — never commit your keys.

### 6. Register the MCP servers

If your `.mcp.json` is in place and correctly named, Claude Code auto-detects it and prompts
you to **trust the project's servers** on first launch — approve that prompt.

If auto-detection doesn't work (or you'd rather be explicit), register them directly — this
bakes your keys in at registration time, so it works regardless of shell env:

```
claude mcp add --transport http tavily "https://mcp.tavily.com/mcp/?tavilyApiKey=$env:TAVILY_API_KEY"
claude mcp add prospeo --env PROSPEO_API_KEY=$env:PROSPEO_API_KEY -- npx -y @prospeo/prospeo-mcp-server
```

_(On bash, use `$TAVILY_API_KEY` / `$PROSPEO_API_KEY` instead of the `$env:` form.)_

Verify:

```
claude mcp list
```

You want **tavily** and **prospeo** both showing connected. Prospeo may lag a few seconds the
first time while `npx` downloads its package.

### 7. Customize your ICP

Open `CLAUDE.md` and edit **Section 1 (Ideal Customer Profile)**. This is the single most
important step — every run ranks against it. Fill in all fields consistently; leftover
placeholder values from a different industry will pull your results off-target.

Example (a home-services automation ICP):

```markdown
- What they do / industry: Small home-services businesses (HVAC, plumbing, electrical,
  landscaping), owner-operated.
- Company size: ~5–50 employees
- Geography: Canada and United States (tighten to your region for a sharper list)
- Tech / other signals (intent): Still handling scheduling, quoting, invoicing, and
  follow-up manually; hiring admin / office / dispatch / ops roles; recently opened a
  second location.
- Who we sell to (target role): Owner, General Manager, or Operations Manager
- What we offer: [one line describing the problem you solve for them]
```

For an automation/services business, define "intent" as evidence of **manual-work pain**
(hiring admin/ops roles, a second location, rising volume without systems) rather than the
funding/sales-hiring signals a software company would look for.

### 8. Launch and confirm

From your project folder, in a terminal that has your keys:

```
claude
```

- Type `/mcp` → confirm tavily and prospeo show green.
- Type `/` → confirm your custom commands (`/find-leads`, `/enrich`) appear in the list.

---

## Running the system

### Free sanity check (spends nothing)

Before any credit-spending run, confirm Prospeo authenticates on your free key:

```
call Prospeo's account-info tool and show my credits
```

This is a free call. If it returns your balance, you're good to go.

### Find leads

```
/find-leads
```

…or refine a single run: `/find-leads focus on Calgary HVAC`.

Watch it work through discovery → signal detection → ranking, then open
`leads/leads-<today>.md` (grouped by tier) to review.

### Pick and enrich

Choose the IDs worth pursuing and enrich only those:

```
/enrich L001 L004 L009
```

Only those leads get a Prospeo lookup. Everything else stays untouched. The verified work
emails get appended to your leads file.

---

## Troubleshooting (the gotchas)

These are the real issues hit while building on Windows — almost all trace back to the two
leading-dot names.

**`/find-leads` says "Unknown command"**
The `.claude/commands/` folder (or the files inside it) isn't where Claude Code expects.
Confirm with `dir .claude\commands` — you should see `find-leads.md` and `enrich.md`. If the
folder is named `claude` without the dot, rename it: `Rename-Item "claude" ".claude"`.

**MCP servers don't appear in `/mcp` (only account connectors show)**
Your `.mcp.json` isn't being read. The usual cause on Windows: it saved as `mcp.json` (no
dot) or `.mcp.json.txt` (Notepad + hidden extensions). Check with `dir .mcp.json`; rename with
`Rename-Item "mcp.json" ".mcp.json"`. Or skip the file entirely and use the two
`claude mcp add` commands from Step 6.

**`claude` is "not recognized" after installing**
PATH only updates in a **new** terminal. Open a fresh window. Persisting? `claude doctor`.

**Servers connect but Tavily fails on the first search**
Empty API key — you launched from a terminal that didn't have the keys set. Quit, set the
keys (or use the permanent method), relaunch from that same window.

**Keys keep disappearing between sessions**
`$env:` / `export` are session-only. Use the permanent method in Step 5 (`setx` / `.bashrc`).

---

## Cost control

- **Tavily** — discovery uses _basic_ depth (1 credit/search) and caches to `.cache/`, so
  re-runs (or re-shooting a recording) don't re-spend credits. A full discovery run stays
  well under 100 of the 1,000 monthly free credits.
- **Prospeo** — gated behind your explicit `/enrich` selection. Email finds are ~0.5 credit
  each; the system never pulls mobile numbers (those cost far more). `/enrich` checks your
  balance (a free call) before spending anything.

---

## Compliance

This system finds business/firmographic data and work contacts for B2B outreach. Keep it at
the company and professional-role level, and run any outreach under a lawful basis with a
clear opt-out (GDPR legitimate interest / CASL / CAN-SPAM as applicable). Enriched lead files
contain real names and emails — they're gitignored by default; keep them that way. _Not legal
advice._

---

_Built with Claude Code, Tavily, and Prospeo._
