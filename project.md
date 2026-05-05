# ccledger

**Cost-per-activation accounting for Claude Code skills. Ships as a Claude Code plugin.**

> Find out which skills earn their rent.

## What this is

A Claude Code plugin that quietly does two things in the background:

1. Tracks the **static cost** of every skill you have installed — frontmatter tokens, body tokens, hook injections — and updates this whenever you add or remove a plugin.
2. Logs every **skill activation** during your real sessions — which skill fired, when, where.

A single `/ccledger:report` command joins the two and tells you, per skill: cost to keep installed × times actually used × verdict (`keep` / `review` / `drop`).

The user-facing surface is one slash command you run maybe once a month. Everything else runs as hooks — invisibly, with no perceptible latency, with no entries in your conversation context.

## What already exists 

- `hardness1020/skills-analytics` — runtime observer plugin. Hooks-based, logs invocations and nested file access into SQLite, has a local dashboard. Mature, well-engineered.
- Anthropic's OpenTelemetry — `claude_code.skill_activated` events native, with `skill.name`, `invocation_trigger`, `skill.source`, `plugin.name`. Requires `OTEL_LOGS_EXPORTER` set.
- `hqhq1025/skill-optimizer`, `JNLei/skill-optimizer`, `alexgreensh/token-optimizer` — skill-by-skill SKILL.md compression. Operate on individual files.
- Alexey Pelykh's gist — empirically derived the ~16,000-char metadata budget (~42 skills before truncation).

ccledger overlaps deliberately with skills-analytics on the activation-logging side (~100 lines duplicated, see Path-A note below) in exchange for being one install instead of two. The novel contribution is **static cost accounting + ROI synthesis** — which nothing currently provides.

### Path

We log activations ourselves rather than reading skills-analytics' SQLite. Trade-off accepted:
- **Cost:** ~100 lines of `PreToolUse` hook code that's similar to skills-analytics'.
- **Benefit:** one plugin to install, no schema coupling to a third-party project, no "but you also need to install X" caveat in the README.

If skills-analytics is also installed, ccledger writes to its own DB anyway. They don't conflict (different SQLite files, different hook script paths), they just both observe activations independently.

## Verifiable success criteria

A user installs ccledger via `/plugin install`, uses Claude Code normally for a week, runs `/ccledger:report`, and sees:

1. A budget breakdown — total tokens always-on, eager metadata, lazy bodies — with per-plugin and per-skill rows.
2. A table of every skill with `cost_to_keep` (eager metadata tokens), `activations_last_7d`, `tokens_per_activation`, and a verdict column.
3. A list of skills installed but never activated.
4. The output is correct — sample 3 skills, hand-count their metadata token cost, ccledger's number is within ±10% (using char/4 heuristic, see Phase 0).

If those four work, MVP is done.

## Plugin structure

```
ccledger/
├── PROJECT.md                          # this file
├── README.md                           # short user-facing doc, written last
├── .claude-plugin/
│   └── plugin.json                     # plugin manifest
├── hooks/
│   ├── session_start_scan.py           # fires on SessionStart, ~150 lines stdlib
│   └── pre_tool_use_log.py             # fires on PreToolUse(Skill), ~50 lines stdlib
├── commands/
│   └── ccledger-report.md              # /ccledger:report slash command
├── scripts/
│   ├── report.py                       # builds the joined report, ~200 lines stdlib
│   └── render_html.py                  # writes the HTML report, ~100 lines stdlib
├── skills/
│   └── ccledger/
│       └── SKILL.md                    # tiny, optional, ~50 tokens metadata
├── references/                         # Phase 0 outputs live here
│   ├── static-budget-model.md
│   ├── hook-surface.md
│   ├── plugin-layout.md
│   └── tokenizer.md
└── tests/
    └── fixtures/                       # tiny synthetic plugins for unit tests
```

**Stdlib only.** No `tiktoken`, no `click`, no `pydantic`. `sqlite3`, `pathlib`, `json`, `hashlib`, `re`, `html.escape` cover everything. The whole plugin should be readable end-to-end.

**Storage:** `~/.ccledger/db.sqlite`. Created on first hook fire. Schema migrations: just don't have any in v1.

## Phased plan

Each phase has a verification step. Do not start phase N+1 until phase N is verified.

### Phase 0 — Diagnosis (no plugin code yet)

Resolve these before writing implementation. The cost of being wrong propagates.

1. **What does Claude Code actually load into context at session start?** With 30 skills installed, does the system inject all 30 frontmatter blocks, or only matches? Read `code.claude.com/docs/en/agents-and-skills`, run `claude --debug`, capture an actual session-start system prompt, count the metadata. → output: `references/static-budget-model.md` describing what loads when, with a worked example from a real session.
2. **What hook events does Claude Code expose, and what data do they receive?** Specifically, does `PreToolUse` on the Skill tool give us the skill name as a parameter, or do we have to parse it from elsewhere? GitHub issue #35319 suggests the parameter passing is fragile. → output: `references/hook-surface.md` documenting hook signatures, the exact JSON payload our scripts will receive, and any known gotchas.
3. **What's already in `~/.claude/plugins/cache/`?** Inspect the layout. Is `plugin.json` always present? Are SKILL.md files predictably nested? Are there hook files we need to count toward the always-on budget? → output: `references/plugin-layout.md` with the directory structure of one real plugin (Superpowers) annotated.
4. **Tokenizer accuracy.** Pick 5 SKILL.md frontmatter blocks. Char/4 heuristic vs `tiktoken cl100k_base`, record the discrepancy. Goal isn't matching Anthropic's tokenizer — goal is "does char/4 rank skills the same way `tiktoken` does." → output: `references/tokenizer.md` stating "we use char/4; relative ranking matches tiktoken on N samples; absolute counts within ±15%."

Phase 0 is information-gathering. Output is four short reference files. **Do not start coding before this is done.** If a question can't be answered from docs and inspection, write it down as a known unknown — don't guess.

### Phase 1 — Plugin skeleton + static scanner

Stand up the plugin manifest, register the `SessionStart` hook, get it to fire and write a no-op log. Then implement the scan:

- Walk `~/.claude/plugins/cache/`, `~/.claude/skills/`, and project-local `.claude/skills/`.
- For each plugin and each skill, classify content into:
  - **Always-on**: SessionStart hooks, system prompt injections.
  - **Eager**: SKILL.md frontmatter (name + description fields).
  - **Lazy**: SKILL.md body.
  - **Passive**: contents of `references/` and `scripts/`.
- Tokenize each (char/4). Hash the inventory; only re-scan if the hash changed since last session. This is critical — the hook must be cheap when nothing has changed.
- Write to SQLite.

→ verify: hand-pick 3 skills, count chars manually, confirm ccledger's stored values match. Install one new plugin, start a new session, confirm ccledger detects it and updates the DB. Time the hook on a hot path (no changes) — should be < 50ms.

### Phase 2 — Activation logging

Register the `PreToolUse` hook for the Skill tool. Append one row per activation: timestamp, skill name, project path, trigger type. Keep the script tiny — open SQLite, INSERT, close, exit.

→ verify: invoke 5 different skills in a test session, confirm 5 rows appear in the DB with correct skill names. Time the hook on tool calls — should be < 20ms (this fires on every Skill use, perceptible latency would be a deal-breaker).

### Phase 3 — Report

The `/ccledger:report` slash command runs `scripts/report.py`, which:
- Joins the latest scan with the activation table over a configurable window (default 30 days).
- Computes per-skill cost-to-keep, activations, tokens-per-activation.
- Applies a verdict heuristic (cost > N AND activations == 0 → `drop`, etc. — pick thresholds in Phase 3, don't overfit).
- Writes a single HTML file to `/tmp/ccledger-report-<timestamp>.html` and prints the path.

→ verify: report makes one actionable recommendation the human user agrees with ("you've never used `superpowers:using-git-worktrees`, consider disabling").

### Out of scope for MVP (explicit non-goals)

- Co-activation matrix / trigger-drift analysis (interesting, but Phase 4+).
- Real-time monitoring or daemon-style live dashboard.
- SKILL.md rewriting recommendations beyond keep/review/drop.
- Multi-user / team analytics.
- A served dashboard (we write a static HTML file; no localhost server in v1).
- Cross-platform GUI.
- Comparing different framework stacks.
- Detecting "skill conflicts."

## Open questions

These are flagged so they get answered, not so they block Phase 0.

- Do MCP tool descriptions count toward the same metadata budget as skills?
- Does the system prompt budget change between Sonnet and Opus? Between Claude Code and the API?
- For project-scoped plugins (`.claude/skills/` inside a repo), should the report aggregate across projects or split per project?
- How should we handle skills behind feature flags or that only fire in subagents?
- If a hook script crashes, does Claude Code surface the error, swallow it, or break the session? (Critical: our hooks must fail closed and silent, never block the user.)

## What's assumed but not yet verified

These are seams. Listed so they're visible, not hidden in code.

- Char/4 approximates Claude's tokenizer well enough for *relative* skill ranking. Probably true; ranking-correctness is the actual bar, not absolute accuracy.
- `~/.claude/plugins/cache/` is the canonical install location across macOS, Linux, Windows. Verify all three in Phase 0.
- `PreToolUse` hooks receive the skill name reliably enough to log it. GitHub issue #35319 suggests this might be fragile — Phase 0 must confirm.
- Hook execution failures are non-fatal to the user's session. If they're fatal, we wrap every hook script in a try/except that logs and exits 0.

## Working principles

- **Diagnosis before repair.** Phase 0 exists for a reason.
- **Simplicity first.** Stdlib only. No package; no compilation step.
- **Surgical changes.** Adding a feature should not refactor existing code.
- **Goal-driven execution.** Every phase ends with a verification step.
- **Stop when confused.** Ask the human; don't guess.
- **Inversion.** Before each phase, write the most likely failure mode. If it's "Anthropic ships this natively next month," consider whether the phase is still worth doing.
- **The medium is the message.** A plugin about token budgets must itself have a tiny token budget. If our plugin grows past ~500 tokens of always-on context cost, we have failed.

## First action

Read this file. Then start Phase 0. Do not write any code in `hooks/`, `commands/`, `skills/`, or `scripts/` until all four `references/*.md` files exist and have been reviewed by the human.