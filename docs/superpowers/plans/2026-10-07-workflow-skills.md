# Workflow Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Chisel generate a plugin-independent workflow (triage into Direct / Quick / Feature tiers; `dev_feature`, `dev_plan`, `dev_execute`, `dev_debug`, `dev_tdd` skills; TASK folders with Markdown + HTML views; a user manual) for both Claude Code and GitHub Copilot.

**Architecture:** Chisel stays pure Markdown. New templates go under `.chisel/templates/workflow/`, `.chisel/templates/task/`, and `.chisel/templates/manual.md.template`. Each workflow template is one source for both targets, with `{{IF_CLAUDE}}…{{/IF_CLAUDE}}` / `{{IF_COPILOT}}…{{/IF_COPILOT}}` blocks resolved at generation time. `.chisel/body.md` is updated to generate them, enforce cleanup, and end with an explanation phase.

**Tech Stack:** Markdown templates, one self-contained HTML template (inline CSS, no JS libraries). No build, no runtime.

**Spec:** `docs/superpowers/specs/2026-10-07-workflow-skills-design.md` — read it before starting any task.

## Global Constraints

- Pure Markdown + one static HTML template. No scripts, packages, or build steps are added to the shipped folders (`.chisel/`, `.claude/`, `.github/`).
- **Never run `/dev-chisel` inside this repository.** Mandatory cleanup would delete Chisel's source. All end-to-end testing happens on scratch copies outside the repo, after the branch is pushed.
- Work on branch `feature/workflow-skills`. Commit after every task. Commit messages end with `Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>`.
- Conditional block syntax matches the existing convention (`{{IF_PREPRODUCTION}}…{{/IF_PREPRODUCTION}}`): `{{IF_CLAUDE}}…{{/IF_CLAUDE}}`, `{{IF_COPILOT}}…{{/IF_COPILOT}}`.
- Workflow skill templates: frontmatter must be the **first lines** of the generated file, so the attribution line `<!-- Adapted from superpowers by Jesse Vincent (MIT) -->` goes on the first line **after** the frontmatter.
- Workflow skill templates: 80–150 lines each (excluding frontmatter). Check with `wc -l`.
- Claude frontmatter: `name: <skill>` + `description: …`. Copilot frontmatter: `description: …` + `mode: agent`. Match the existing driver-mode files.
- Generated skill names (exact): `dev_feature`, `dev_plan`, `dev_execute`, `dev_debug`, `dev_tdd`.
- TASK folder path (exact): `.docs/tasks/TASK-NNN-<slug>/`, archive `.docs/tasks/done/`. Slug: lowercase ASCII, words joined by `-`, max 40 chars, non-alphanumerics dropped.
- Status values (exact, in order): `brainstorming`, `specced`, `planned`, `in-progress n/m`, `done`.
- Work item IDs `WI-1…WI-n`; parallel groups `A, B, C…` executed in order.
- Every skill reads `.docs/architecture.md` and the rules file (`.claude/rules/rules.md` / `.github/instructions/rules.instructions.md`) before acting.
- Write in Chisel's existing voice: imperative, terse, bold lead-ins, tables where they help.

## Review Focus

1. **Re-run / "Modify existing" on a project scaffolded by old Chisel** — `todo.md` already has entries and tallies; adding the TASK format must keep existing entries and continue the tally, not reset it. → Task 3 (todo template convention text) + Task 9 (body.md Phase 5/7 instruction) + verification grep.
2. **Opening the HTML fails** (WSL, SSH, headless, no default browser) — must print the absolute path and continue, never abort the skill. → Task 1 (shared "render & open" procedure) checked in Tasks 4, 5, 8.
3. **Same-group work items turn out to share a file at execution time** (plan was wrong or code moved) — `dev_execute` must run those items one after another, not in parallel. → Task 6.
4. **Mainseat during `dev_execute`** — agents return snippets, so nothing is applied yet; `dev_execute` must wait for the user to confirm each item was applied before review/ticking. → Task 6.
5. **Odd TASK titles** (spaces, punctuation, non-ASCII, very long) — slug rule must produce a valid, unique folder name; collision with an existing folder appends `-2`. → Task 4.

---

### Task 1: HTML view template

**Files:**
- Create: `.chisel/templates/workflow/task-view.html.template`

**Interfaces:**
- Produces placeholders consumed by Tasks 4, 5, 6, 8, 9: `{{TITLE}}`, `{{SOURCE_FILE}}`, `{{TOC}}`, `{{CONTENT}}`.
- Produces the **"Render & open" procedure** text (Step 2) that Tasks 4, 5, 6 paste into their skills verbatim.

- [ ] **Step 1: Create the template with this exact content**

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{{TITLE}}</title>
<style>
:root{--bg:#fbfaf7;--fg:#1f2328;--muted:#5d646d;--line:#e3e0d8;--accent:#2f6f5e;--code:#f1efe9;--banner:#fff4d6}
@media (prefers-color-scheme:dark){:root{--bg:#16181b;--fg:#e6e6e3;--muted:#9aa0a6;--line:#2c3036;--accent:#7cc4ad;--code:#202328;--banner:#3a3220}}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--fg);font:16px/1.6 system-ui,-apple-system,"Segoe UI",sans-serif}
.banner{background:var(--banner);border-bottom:1px solid var(--line);padding:8px 16px;font-size:13px;color:var(--muted)}
.wrap{max-width:1100px;margin:0 auto;padding:24px 16px;display:grid;grid-template-columns:220px 1fr;gap:32px}
nav{position:sticky;top:16px;align-self:start;font-size:14px}
nav a{display:block;color:var(--muted);text-decoration:none;padding:2px 0}
nav a:hover{color:var(--accent)}
main{min-width:0}
h1{font-size:28px;margin:0 0 8px}h2{font-size:21px;margin-top:36px;border-bottom:1px solid var(--line);padding-bottom:4px}h3{font-size:17px;margin-top:24px}
a{color:var(--accent)}
code{background:var(--code);padding:1px 5px;border-radius:4px;font-size:.9em}
pre{background:var(--code);padding:12px;border-radius:6px;overflow-x:auto}pre code{padding:0}
table{border-collapse:collapse;width:100%;margin:12px 0;font-size:14px;display:block;overflow-x:auto}
th,td{border:1px solid var(--line);padding:6px 10px;text-align:left;vertical-align:top}
th{background:var(--code)}
.groups{display:grid;grid-template-columns:repeat(auto-fit,minmax(240px,1fr));gap:12px;margin:12px 0}
.group{border:1px solid var(--line);border-radius:8px;padding:10px}
.group h4{margin:0 0 8px;font-size:14px;color:var(--accent)}
.wi{border-left:3px solid var(--accent);padding:4px 8px;margin:6px 0;font-size:14px}
.wi.done{opacity:.6;text-decoration:line-through}
.wi small{display:block;color:var(--muted)}
@media (max-width:760px){.wrap{grid-template-columns:1fr}nav{position:static}}
</style>
</head>
<body>
<div class="banner">Generated from <code>{{SOURCE_FILE}}</code> — edit the Markdown, not this file.</div>
<div class="wrap">
<nav>{{TOC}}</nav>
<main>
{{CONTENT}}
</main>
</div>
</body>
</html>
```

- [ ] **Step 2: Record the shared "Render & open" procedure**

This exact block is pasted into `dev_feature`, `dev_plan`, and `dev_execute` (Tasks 4–6), and referenced by body.md Phase 7/9 (Task 9). Keep it here as the canonical copy (do not create a file for it):

```markdown
### Render & open

1. Read `.docs/templates/task-view.html` and the source `.md`.
2. Convert the Markdown to HTML yourself: headings get `id`s (lowercase, hyphenated); tables → `<table>`; fenced code → `<pre><code>` with `<`, `>`, `&` escaped; checkboxes → `☐` / `☑`.
3. Fill `{{TITLE}}` (the doc's H1), `{{SOURCE_FILE}}` (the .md path), `{{TOC}}` (one `<a href="#id">` per H2), `{{CONTENT}}`.
4. Write the result next to the source with the same name and `.html` extension, replacing any previous version.
5. Open it: Windows `start "" "<path>"`, macOS `open "<path>"`, Linux `xdg-open "<path>"`. If the command fails or there is no desktop (SSH, WSL without a browser, container), print the absolute path and continue — never stop the skill over this.
```

Note: the template is **copied** into the target project at `.docs/templates/task-view.html` during generation (Task 9), because `.chisel/templates/` is deleted by cleanup.

- [ ] **Step 3: Verify**

Run: `grep -c "{{TITLE}}\|{{SOURCE_FILE}}\|{{TOC}}\|{{CONTENT}}" .chisel/templates/workflow/task-view.html.template`
Expected: `4`
Run: `grep -c "<script" .chisel/templates/workflow/task-view.html.template`
Expected: `0`
Open the file in a browser via the Render & open command for your OS; confirm it loads (placeholders visible as text) and switches palette when the OS is in dark mode.

- [ ] **Step 4: Commit**

```bash
git add .chisel/templates/workflow/task-view.html.template
git commit -m "Add self-contained HTML view template for specs, plans, manual"
```

---

### Task 2: TASK folder templates (spec + plan)

**Files:**
- Create: `.chisel/templates/task/spec.md.template`
- Create: `.chisel/templates/task/plan.md.template`

**Interfaces:**
- Produces spec section headings (exact, consumed by Tasks 4, 5, 6): `## Goal`, `## Context`, `## Decisions`, `## Design`, `## Out of scope`, `## Acceptance criteria`, `## Open questions`.
- Produces work-item format (exact, consumed by Tasks 5, 6): heading `### WI-<n> — <title>   [group <X> · agent: <agent-name>]`, then lines `Depends on:`, `Files:`, `Done when:`, then `- [ ]` steps. A completed item's heading gets ` ✅` appended.
- Like `.docs/templates/task-view.html`, these two files are copied into the target project at `.docs/templates/` (Task 9).

- [ ] **Step 1: Create `spec.md.template` with this exact content**

```markdown
# TASK-{{TASK_NUMBER}} — {{TASK_TITLE}}

**Status:** {{STATUS}} · **Priority:** {{PRIORITY}} · **Todo entry:** `.docs/todo.md`

## Goal

What this TASK delivers and for whom, in 2–4 sentences.

## Context

Relevant existing code, docs, and constraints discovered while exploring. Link files by path.

## Decisions

| # | Decision | Why |
|---|---|---|
| D1 | … | … |

## Design

How it works: components, data flow, interfaces, error handling. Use subsections (`###`) per component.

## Out of scope

What this TASK deliberately does not do.

## Acceptance criteria

1. Each criterion is concrete and checkable (a command, a behavior, an observable result).

## Open questions

None — or a list. A spec with open questions is not approved.
```

- [ ] **Step 2: Create `plan.md.template` with this exact content**

````markdown
# TASK-{{TASK_NUMBER}} — {{TASK_TITLE}} · Plan

**Spec:** `spec.md` · **Progress:** 0/{{WI_COUNT}}

## Groups

Groups run in order. Items inside one group run in parallel and never touch the same file.

| Group | Work items | Depends on groups |
|---|---|---|
| A | WI-1, WI-2 | — |

## Coverage

| Acceptance criterion | Work items |
|---|---|
| 1 | WI-1 |

## Work items

### WI-1 — <title>   [group A · agent: <agent-name>]
Depends on: —
Files: `path/to/file`
Done when: <concrete, checkable condition>
- [ ] <step>
````

- [ ] **Step 3: Verify**

Run: `grep -c "^## " .chisel/templates/task/spec.md.template`
Expected: `7`
Run: `grep -E "^### WI-1 — .*\[group A · agent: " .chisel/templates/task/plan.md.template`
Expected: one matching line.

- [ ] **Step 4: Commit**

```bash
git add .chisel/templates/task/
git commit -m "Add TASK folder templates for spec and plan"
```

---

### Task 3: Rules, todo, and index templates

**Files:**
- Modify: `.chisel/templates/rules.md.template` (replace the `## Workflow` section; add two sections after it)
- Modify: `.chisel/templates/todo.md.template` (add TASK entry format + statuses to the Convention block)
- Modify: `.chisel/templates/CLAUDE.md.template` (add Workflow section; context files mention manual + tasks)

**Interfaces:**
- Consumes: skill names, status values, folder paths from Global Constraints.
- Produces: the triage rule text that the manual (Task 8) summarizes. Section heading names (exact): `## Workflow — Triage`, `## Workflow — Direct`, `## Workflow — Quick`, `## Workflow — Feature`, `## Verify Before Claiming Done`, `## Review Feedback`.

- [ ] **Step 1: Replace the `## Workflow` section of `rules.md.template`** (from `## Workflow` up to, not including, `## Missing Setup Data`) with:

```markdown
## Workflow — Triage

Before acting on any request, pick a tier. Ask the two questions in order:

1. **Does any feature signal fire?** — a new feature or capability; spans several subsystems or ~5+ files; design decisions the user hasn't made; naturally splits into independent pieces. → If yes, propose in one line: *"This looks like a feature — run it as a TASK with a spec, or keep it quick?"* and wait.
2. **Do all direct conditions hold?** — localized (one function / a few lines / one file); the request says exactly what to do; no design choice; no new dependency. → If yes, **Direct**.
3. Otherwise → **Quick**.

When unsure, pick the heavier tier. Escalation is one-way: if work turns out bigger than its tier, stop, say so, and step up. The user can override either way ("just do it" / `/dev_feature`). `BUG` entries go through `/dev_debug` in any tier above Direct. Driver-mode applies to every tier.

## Workflow — Direct

Make the change → verify (run the relevant test, or at least build/lint) → report in one or two lines. No plan, no approval, no todo entry. If it touches a second file or breaks a call site, stop and switch to Quick.

## Workflow — Quick

**1. Explore** — load relevant `.docs/` files, then read the affected code.
**2. Plan** — present a short plan in chat; wait for approval.
**3. Implement** — only after approval.
**4. Test** — {{TEST_INSTRUCTIONS}}
**5. Housekeeping** — update `.docs/` if something meaningful changed. Move the entry from `todo.md` to `todo_done.md`. If a new system doesn't fit an existing doc, ask whether to create one; keep the index's Context Files list in sync. If scope, stack, or run instructions change, update the index ("What This Project Does" / "How to Run").

## Workflow — Feature

`/dev_feature` (brainstorm → `spec.md`) → `/dev_plan` (work items → `plan.md`) → `/dev_execute` (agents implement, `code-reviewer` checks, TASK closes). Each step waits for approval and offers the next. Everything for the TASK lives in `.docs/tasks/TASK-NNN-<slug>/`; Markdown is the source of truth, the `.html` files are generated views.

## Verify Before Claiming Done

Never say "done", "fixed", or "passing" without running the check in this turn and showing its output. No "should work".

## Review Feedback

Treat review feedback (from `code-reviewer` or a person) as claims to verify, not orders. Check each point against the code; apply what holds, push back with reasons on what doesn't.
```

Keep the existing `{{TEST_INSTRUCTIONS}}` placeholder exactly once (it is now inside Quick step 4).

- [ ] **Step 2: Update the Agents section of `rules.md.template`** — after the sentence "Always tell the user which agent you are invoking and why before launching it." add:

```markdown
`code-reviewer` is always present — `/dev_execute` uses it to check every work item.
```

- [ ] **Step 3: Update `todo.md.template`** — append inside the `> **Convention:**` blockquote (after the tally line):

```markdown
> **Feature TASKs** use one line linking to their folder: `- **TASK-012** [P1] User authentication — \`planned\` 3/7 → .docs/tasks/TASK-012-user-auth/`
> Status moves `brainstorming` → `specced` → `planned` → `in-progress n/m` → `done`. Only `/dev_feature`, `/dev_plan`, `/dev_execute` change it.
> TASK numbers share the TASK tally with quick TASK entries — never restart numbering.
```

- [ ] **Step 4: Update `CLAUDE.md.template`** — insert before `## Agents`:

```markdown
## Workflow

Requests are triaged into Direct / Quick / Feature (see rules). Feature work: `/dev_feature` → `/dev_plan` → `/dev_execute`; bugs: `/dev_debug`; test-first: `/dev_tdd`. TASK folders live in `.docs/tasks/`. User manual: `.docs/manual.md`.
```

- [ ] **Step 5: Verify**

Run: `grep -c "^## Workflow — \(Triage\|Direct\|Quick\|Feature\)$" .chisel/templates/rules.md.template`
Expected: `4`
Run: `grep -c "{{TEST_INSTRUCTIONS}}" .chisel/templates/rules.md.template`
Expected: `1`
Run: `grep -c "## Driver-Mode" .chisel/templates/rules.md.template`
Expected: `1` (Driver-Mode section untouched)
Run: `grep -c "never restart numbering" .chisel/templates/todo.md.template`
Expected: `1` (Review Focus #1)

- [ ] **Step 6: Commit**

```bash
git add .chisel/templates/rules.md.template .chisel/templates/todo.md.template .chisel/templates/CLAUDE.md.template
git commit -m "Add three-tier triage workflow to rules, TASK format to todo, workflow pointer to index"
```

---

### Task 4: `dev_feature` skill template

**Files:**
- Create: `.chisel/templates/workflow/dev_feature.md.template`

**Interfaces:**
- Consumes: spec headings (Task 2), Render & open block (Task 1), status values, slug rule.
- Produces: a TASK folder containing `spec.md` + `spec.html`, todo entry at `specced`.

- [ ] **Step 1: Write the template.** Structure (in this order):

1. Frontmatter in conditional blocks:
   ```
   {{IF_CLAUDE}}
   ---
   name: dev_feature
   description: Start a feature TASK — brainstorm with the user one question at a time, write spec.md, render spec.html, get approval. Use when triage proposes the Feature flow or the user asks for a feature.
   ---
   {{/IF_CLAUDE}}
   {{IF_COPILOT}}
   ---
   description: Start a feature TASK — brainstorm with the user one question at a time, write spec.md, render spec.html, get approval.
   mode: agent
   ---
   {{/IF_COPILOT}}
   ```
2. Attribution comment line.
3. One-line intro: *"You are executing `/dev_feature` — turn a feature idea into an approved spec."*
4. `## Before starting` — read architecture + rules file; if the user named an existing TASK, resume it (read its `spec.md`, continue from its status) instead of creating a new one.
5. `## Steps`, numbered:
   1. **Create the TASK** — next TASK number from the todo tally; slug rule from Global Constraints; **if the folder already exists, append `-2`, `-3`…** (Review Focus #5); create `.docs/tasks/TASK-NNN-<slug>/`; add the todo line with status `brainstorming`; increment the tally.
   2. **Explore** — read only the code and docs the feature touches.
   3. **Ask** — one question per message, multiple choice where possible, recommended option first; cover purpose, constraints, success criteria. Stop asking when you could write the acceptance criteria.
   4. **Propose approaches** — 2–3 with trade-offs; lead with your recommendation.
   5. **Write `spec.md`** from `.docs/templates/spec.md` — every section filled; Decisions table lists what the user chose.
   6. **Self-review** — no placeholders/TBDs, no contradictions, each acceptance criterion checkable, scope fits one TASK. If too big, propose splitting into several TASKs and stop.
   7. **Render & open** — paste the Task 1 block verbatim as a `###` subsection.
   8. **Approval gate** — ask the user to review `spec.html`. On changes: edit `spec.md`, re-render, ask again. On approval: status → `specced`, then offer: *"Spec approved. Run `/dev_plan` next?"*
6. `## Rules` — one question per message; never write code in this skill; never mark `specced` without explicit approval; driver-mode does not apply (this skill writes only `.docs/` files).
{{IF_CLAUDE}} note inside Explore: "use the `architect` agent for exploration if it exists". {{IF_COPILOT}} counterpart: "explore in-session".

- [ ] **Step 2: Verify**

Run: `sed '/^---$/,/^---$/d' .chisel/templates/workflow/dev_feature.md.template | wc -l`
Expected: between 80 and 150.
Run: `grep -c "IF_CLAUDE\|IF_COPILOT" .chisel/templates/workflow/dev_feature.md.template`
Expected: an even number ≥ 4 (every block opened and closed).
Run: `grep -c "append \`-2\`" .chisel/templates/workflow/dev_feature.md.template`
Expected: `1`
Run: `grep -c "print the absolute path and continue" .chisel/templates/workflow/dev_feature.md.template`
Expected: `1` (Review Focus #2)

- [ ] **Step 3: Commit**

```bash
git add .chisel/templates/workflow/dev_feature.md.template
git commit -m "Add dev_feature skill template (brainstorm to approved spec)"
```

---

### Task 5: `dev_plan` skill template

**Files:**
- Create: `.chisel/templates/workflow/dev_plan.md.template`

**Interfaces:**
- Consumes: approved `spec.md` (status `specced`); work-item format (Task 2); Render & open (Task 1).
- Produces: `plan.md` + `plan.html`, todo entry at `planned`.

- [ ] **Step 1: Write the template.** Same frontmatter/attribution pattern as Task 4 (`name: dev_plan`; description: "Turn an approved TASK spec into a plan of work items with agents, dependencies, files, and parallel groups; render plan.html; get approval."). Sections:

1. `## Before starting` — read architecture + rules; locate the TASK (argument, or the single TASK at `specced`; if several, ask which); refuse if status isn't `specced` ("Run `/dev_feature` first").
2. `## Steps`:
   1. **Derive work items** — each item is one coherent change an agent can finish and a reviewer can judge alone. Fill agent (from the project's agents; main session if none fits), Depends on, Files (exact paths), Done when (a command or observable check), steps.
   2. **Assign groups** — topological order by dependencies; items in the same group must have **disjoint `Files:`** — if two overlap, move one to the next group.
   3. **Coverage** — fill the Coverage table; every acceptance criterion maps to ≥1 item. Any gap → add an item.
   4. **Write `plan.md`** from `.docs/templates/plan.md`; set Progress `0/n`.
   5. **Render & open** — Task 1 block verbatim, plus: "For `plan.html`, also render the Groups section as `<div class="groups">` with one `<div class="group"><h4>Group X</h4>` per group and one `<div class="wi">` per item (`<small>` line: agent · depends on); add class `done` to ticked items."
   6. **Approval gate** — on approval: status → `planned`, offer `/dev_execute`.
3. `## Rules` — no code in this skill; never put two items touching the same file in one group; never mark `planned` without approval.

- [ ] **Step 2: Verify**

Run: `sed '/^---$/,/^---$/d' .chisel/templates/workflow/dev_plan.md.template | wc -l` → 80–150.
Run: `grep -c "disjoint" .chisel/templates/workflow/dev_plan.md.template` → ≥1.
Run: `grep -c 'class="groups"' .chisel/templates/workflow/dev_plan.md.template` → `1`.

- [ ] **Step 3: Commit**

```bash
git add .chisel/templates/workflow/dev_plan.md.template
git commit -m "Add dev_plan skill template (spec to parallel work items)"
```

---

### Task 6: `dev_execute` skill template + reviewer work-item mode

**Files:**
- Create: `.chisel/templates/workflow/dev_execute.md.template`
- Modify: `.chisel/templates/agents/code-reviewer.md.template` (`## When Invoked`)

**Interfaces:**
- Consumes: `plan.md` at `planned` or `in-progress n/m`; work-item format; reviewer agent; driver-mode directive text from `rules.md.template`; Render & open.
- Produces: implemented code, ticked items, todo `in-progress n/m` → `done`, folder moved to `.docs/tasks/done/`, entry moved to `todo_done.md`.

- [ ] **Step 1: Update reviewer** — replace the `## When Invoked` list in `code-reviewer.md.template` with:

```markdown
## When Invoked

- **Reviewing a work item** (the prompt contains a `WI-` item and its *Done when*): review only the files that item lists, check the *Done when* condition (run it if it's a command), and end with `VERDICT: PASS` or `VERDICT: FAIL — <reasons>`.
- **General review**: run `git diff` to see recent changes, identify modified files, and begin immediately.
```

- [ ] **Step 2: Write `dev_execute.md.template`.** Frontmatter/attribution as Task 4 (`name: dev_execute`; description: "Execute an approved TASK plan — dispatch work items to agents group by group, review each with code-reviewer, track progress, verify acceptance criteria, close the TASK."). Sections:

1. `## Before starting` — read architecture + rules + the TASK's `spec.md` and `plan.md`; status must be `planned` or `in-progress`; **resume from the first unticked item** (skip items whose heading ends with ✅); note the conversation's driver-mode.
2. `## For each group, in order`:
   1. Collect the group's unticked items. **Re-check file overlap now** — if two items list the same file, or a file another item is already editing, run those items sequentially (Review Focus #3).
   2. Dispatch each item to its agent with: the full WI text, the spec path, "follow `/dev_tdd` for code changes", and in mainseat the directive *"Operate in mainseat mode — provide ready-to-paste snippets, do not call Edit/Write."*
      {{IF_CLAUDE}}Launch independent items in a single message with multiple Agent tool calls so they run in parallel.{{/IF_CLAUDE}}
      {{IF_COPILOT}}If a subagent tool is available, use it per item; otherwise implement items one by one in this session, following the agent's chat-mode instructions.{{/IF_COPILOT}}
   3. **Mainseat:** after an agent returns snippets, wait for the user to say they applied them before reviewing (Review Focus #4).
   4. **Review** each item with `code-reviewer` (work-item mode). `FAIL` → send the reasons back to the same agent once; second `FAIL` → stop and report to the user with the reviewer's reasons.
   5. **Track** — on `PASS`: tick the item's steps, append ` ✅` to its heading, update Progress in `plan.md` and `in-progress n/m` in `todo.md`, re-render `plan.html` (Render & open, but don't re-open the browser on every item — open once per group).
3. `## Close the TASK`:
   1. Run every acceptance criterion's check and show the output (Verify Before Claiming Done).
   2. Update `.docs/` (architecture, other docs) for what changed.
   3. Ask the user to confirm the TASK is done. On yes: status `done`; move the todo line to `todo_done.md`; move the folder to `.docs/tasks/done/`.
4. `## Rules` — never skip review; never tick an item without `PASS`; stop on blockers rather than improvising outside the plan; if the plan is wrong, stop and suggest re-running `/dev_plan`.
5. Include the Render & open block (Task 1) verbatim as `### Render & open`.

- [ ] **Step 3: Verify**

Run: `sed '/^---$/,/^---$/d' .chisel/templates/workflow/dev_execute.md.template | wc -l` → 80–150.
Run: `grep -c "first unticked" .chisel/templates/workflow/dev_execute.md.template` → ≥1.
Run: `grep -c "sequentially" .chisel/templates/workflow/dev_execute.md.template` → ≥1.
Run: `grep -c "applied them" .chisel/templates/workflow/dev_execute.md.template` → ≥1.
Run: `grep -c "VERDICT: PASS" .chisel/templates/agents/code-reviewer.md.template` → `1`.

- [ ] **Step 4: Commit**

```bash
git add .chisel/templates/workflow/dev_execute.md.template .chisel/templates/agents/code-reviewer.md.template
git commit -m "Add dev_execute skill template and work-item review mode for code-reviewer"
```

---

### Task 7: `dev_debug` and `dev_tdd` skill templates

**Files:**
- Create: `.chisel/templates/workflow/dev_debug.md.template`
- Create: `.chisel/templates/workflow/dev_tdd.md.template`

**Interfaces:**
- Consumes: rules file sections `Verify Before Claiming Done`; `{{TEST_INSTRUCTIONS}}`-equivalent knowledge (the skill tells the agent to use the project's test command from the rules).
- Produces: nothing other tasks depend on except the names.

- [ ] **Step 1: Write `dev_debug.md.template`.** Frontmatter/attribution pattern (`name: dev_debug`; description: "Find and fix the root cause of a bug or failing test — reproduce, gather evidence, test one hypothesis at a time, fix, add a regression test, verify."). Sections:
  1. `## Iron rule` — no fix before the root cause is identified with evidence.
  2. `## Steps` — 1 Reproduce (exact command/steps; if you can't reproduce, gather more information from the user — don't guess) · 2 Gather evidence (error text, stack trace, logs, recent changes via `git log`/`git diff`, the data at the failure point) · 3 Trace backwards from the symptom to where the bad value originates · 4 One hypothesis, stated explicitly, tested with the smallest possible check · 5 Fix the root cause, not the symptom · 6 Regression test that fails without the fix · 7 Verify with real output.
  3. `## Two-strike rule` — after 2 failed fixes, stop; list what you assumed and what the evidence says; discuss with the user before a third attempt.
  4. `## Housekeeping` — if a todo `BUG` entry exists, follow Quick/Feature housekeeping.

- [ ] **Step 2: Write `dev_tdd.md.template`.** Frontmatter/attribution pattern (`name: dev_tdd`; description: "Test-first development — write a failing test, watch it fail, write the minimal code to pass, refactor."). Sections:
  1. `## Before starting` — find the project's test command in the rules file; **if the project has no test setup, stop and propose setting one up** (Quick task, or a Feature TASK if substantial) before continuing.
  2. `## Cycle` — Red (one test for one behavior; run it; it must fail for the expected reason — if it passes, the test is wrong) · Green (minimal code to pass; run all related tests) · Refactor (only with tests green; re-run).
  3. `## Rules` — no production code without a failing test first; never weaken a test to make it pass; test behavior, not implementation details; mocks only at system boundaries.
  4. `## Exceptions` — config, generated code, pure docs: tell the user you're skipping TDD and why.

- [ ] **Step 3: Verify**

Run: `for f in dev_debug dev_tdd; do sed '/^---$/,/^---$/d' .chisel/templates/workflow/$f.md.template | wc -l; done` → each 80–150. If a skill is naturally shorter, expand the rationale under each step until it's ≥80; never pad with filler.
Run: `grep -c "no test setup" .chisel/templates/workflow/dev_tdd.md.template` → ≥1.

- [ ] **Step 4: Commit**

```bash
git add .chisel/templates/workflow/dev_debug.md.template .chisel/templates/workflow/dev_tdd.md.template
git commit -m "Add dev_debug and dev_tdd skill templates"
```

---

### Task 8: User manual template

**Files:**
- Create: `.chisel/templates/manual.md.template`

**Interfaces:**
- Consumes: tier names, skill names, folder paths, driver-mode commands.
- Produces placeholders filled by Task 9: `{{PROJECT_NAME}}`, `{{AGENTS_TABLE}}`, `{{TEST_INSTRUCTIONS}}`, `{{IF_GREENFIELD}}…{{/IF_GREENFIELD}}`, plus `{{IF_CLAUDE}}`/`{{IF_COPILOT}}` for paths.

- [ ] **Step 1: Write the template** in plain, friendly language (reader: the developer, not the assistant). Sections:
  1. `# {{PROJECT_NAME}} — Working with your AI assistant`
  2. `## Just ask` — you talk normally; the assistant sorts every request into Direct / Quick / Feature. Three-row table: tier · what it's for · what you'll see (Direct: it just does it and shows the check; Quick: a short plan to approve; Feature: it asks whether to run a TASK).
  3. `## Features, step by step` — `/dev_feature` → questions → `spec.html` opens → approve → `/dev_plan` → `plan.html` → approve → `/dev_execute` → progress in `todo.md` → confirm done. One sentence each. Mention: edit the `.md`, not the `.html`; a TASK can be resumed in a new conversation.
  4. `## Commands` — table: `/dev_feature`, `/dev_plan`, `/dev_execute`, `/dev_debug`, `/dev_tdd`, `/dev_mainseat`, `/dev_backseat`, one line each.
  5. `## Agents` — `{{AGENTS_TABLE}}` (name · what it's for · edits code?).
  6. `## Where things live` — `.docs/architecture.md`, `todo.md`, `todo_done.md`, `tasks/`, `tasks/done/`, `templates/` (don't edit unless you want to change the format), rules file path ({{IF_CLAUDE}}`.claude/rules/rules.md`{{/IF_CLAUDE}}{{IF_COPILOT}}`.github/instructions/rules.instructions.md`{{/IF_COPILOT}}).
  7. `## Tips` — say "just do it" to skip the feature question; say "make this a feature" to force it; tests run with: {{TEST_INSTRUCTIONS}}.
  8. `{{IF_GREENFIELD}}## Start here\nYour todo list has a kickoff TASK: define the project scope and architecture. Run `/dev_feature` on it.{{/IF_GREENFIELD}}`

- [ ] **Step 2: Verify**

Run: `grep -c "dev_feature\|dev_plan\|dev_execute\|dev_debug\|dev_tdd\|dev_mainseat\|dev_backseat" .chisel/templates/manual.md.template` → ≥7.
Run: `wc -l .chisel/templates/manual.md.template` → ≤120 (keep it short).

- [ ] **Step 3: Commit**

```bash
git add .chisel/templates/manual.md.template
git commit -m "Add user manual template"
```

---

### Task 9: body.md — phases, generation, cleanup, explanation

**Files:**
- Modify: `.chisel/body.md`
- Modify: `.claude/skills/dev-chisel/SKILL.md`, `.github/prompts/dev-chisel.prompt.md` (description only, if it mentions optional cleanup — currently it doesn't; verify and leave unchanged if so)

**Interfaces:**
- Consumes: every template from Tasks 1–8 and their placeholders.
- Produces: the end-to-end scaffolder behavior tested in Task 11.

- [ ] **Step 1: Phase 1** — add item 6: *"Detect the test setup (test directories, test scripts in the manifest, test framework config). Record the test command if found — it fills `{{TEST_INSTRUCTIONS}}` and the manual."*

- [ ] **Step 2: Phase 2** — replace the whole section with a mandatory acknowledgement:

```markdown
## Phase 2: Self-Cleanup Notice

Before any other questions, show this notice and wait for acknowledgement:

> **Heads-up — this scaffolder is one-shot.**
>
> When setup finishes, `/dev-chisel` **will** remove its own source from this project:
> - `.chisel/` — the scaffolder instructions and template library
> - `.claude/skills/dev-chisel/` — the Claude Code skill entry
> - `.github/prompts/dev-chisel.prompt.md` — the Copilot prompt entry
>
> Everything generated stays. You will **not** be able to run `/dev-chisel` again in this project without reinstalling it.
>
> **Type "ok" to acknowledge and continue.**

If the user does not acknowledge, stop the run without creating any files.
```

- [ ] **Step 3: Phase 4** — change the summary line `**Self-cleanup at end:** [yes / no]` to `**Self-cleanup at end:** yes (acknowledged)`.

- [ ] **Step 4: Phase 5** — add after the Skip/Overwrite/Backup list: *"`.docs/todo.md` and `.docs/todo_done.md` with existing entries are never replaced wholesale in **Modify existing** mode: add the new Convention lines and keep every entry and tally as-is."* (Review Focus #1)

- [ ] **Step 5: Phase 6 Agents** — state that `code-reviewer` is always included (shown as "included", not offered); the user picks any other agents. Remove nothing else. Add a `### Workflow` note: *"No question — the Direct / Quick / Feature workflow and all five workflow skills are always generated."*

- [ ] **Step 6: Phase 7** — add a `### Workflow conditionals` paragraph: resolve `{{IF_CLAUDE}}`/`{{IF_COPILOT}}` by keeping the target's block and deleting the other (markers included); resolve `{{IF_GREENFIELD}}` the same way. Then extend the file lists:
  - Claude target: `.claude/skills/{dev_feature,dev_plan,dev_execute,dev_debug,dev_tdd}/SKILL.md` from `.chisel/templates/workflow/<name>.md.template`.
  - Copilot target: `.github/prompts/<name>.prompt.md` from the same templates.
  - Shared: `.docs/tasks/.gitkeep`, `.docs/tasks/done/.gitkeep`; `.docs/templates/task-view.html`, `.docs/templates/spec.md`, `.docs/templates/plan.md` (copied verbatim — their placeholders are filled later by the skills, **not** now); `.docs/manual.md` from `manual.md.template`, then `.docs/manual.html` using the Render & open procedure (render only; open in Phase 9).
  - Update the "Exception" sentence: verbatim-copy files are `driver-mode/*` and the three `.docs/templates/*` files.

- [ ] **Step 7: Phase 8 → Self-Cleanup** — move the cleanup section to Phase 8 and make it unconditional. Remove the "If the user answered no" text. Extend **Do NOT delete** with: `.claude/skills/dev_{feature,plan,execute,debug,tdd}/`, `.github/prompts/dev_{feature,plan,execute,debug,tdd}.prompt.md`, `.docs/templates/`, `.docs/manual.*`.

- [ ] **Step 8: Phase 9 → Explain the System** — replace the old Phase 8 Summary with:

```markdown
## Phase 9: Explain the System

1. Open `.docs/manual.html` (platform command; print the path if it fails).
2. In chat, keep it short and loose — a few lines, no file-by-file walkthrough:
   - What was set up (one line per area: instructions, rules, agents, workflow skills, `.docs/`) and any files backed up.
   - How to work: "Just ask — I'll pick Direct, Quick, or Feature. For a feature I'll ask first."
   - Greenfield: the kickoff TASK in `.docs/todo.md` is the first step — run `/dev_feature` on it.
   - "Everything else is in the manual that just opened (`.docs/manual.md`)."
3. Confirm the scaffolder source was removed.
```

- [ ] **Step 9: Important Rules** — update: "Phase 2 acknowledgement is required before anything else; cleanup always runs." Replace "Phase 2 confirmation runs before any generation — … regardless of which they pick" accordingly. Add: "Generated rules MUST include the four `Workflow —` sections and the Driver-Mode section." Add: "Never leave `{{IF_…}}` markers in generated files."

- [ ] **Step 10: Verify**

Run: `grep -n "^## Phase" .chisel/body.md`
Expected: Phases 1–9 in order, Phase 8 titled `Self-Cleanup`, Phase 9 titled `Explain the System`.
Run: `grep -c "Y / n\|answered \*\*no\*\*\|answered no" .chisel/body.md` → `0`.
Run: `grep -c "dev_execute" .chisel/body.md` → ≥2 (generate + do-not-delete).
Run: `grep -c "keep every entry and tally" .chisel/body.md` → `1`.
Read `.chisel/body.md` top to bottom once, checking that every template from Tasks 1–8 is referenced in Phase 7.

- [ ] **Step 11: Commit**

```bash
git add .chisel/body.md .claude/skills/dev-chisel/SKILL.md .github/prompts/dev-chisel.prompt.md
git commit -m "body.md: mandatory cleanup, workflow generation, explanation phase"
```

---

### Task 10: README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update sections**
  - **How it works → The run, phase by phase:** new 9-phase list (Phase 2 acknowledgement; Phase 8 cleanup; Phase 9 explanation + manual).
  - **New section "Workflow: Direct, Quick, Feature"** after "Specialized agents": the triage questions, the three tiers table, the feature flow (`/dev_feature` → `/dev_plan` → `/dev_execute`), TASK folders, HTML views, `/dev_debug`, `/dev_tdd`.
  - **What gets generated:** add workflow skills/prompts, `.docs/manual.*`, `.docs/tasks/`, `.docs/templates/` to both trees.
  - **Specialized agents:** `code-reviewer` always included.
  - **Design principles:** replace "Clean up after itself — one-shot by default" with "Clean up after itself — always one-shot; you acknowledge it up front."
  - **Repository layout:** add `workflow/`, `task/`, `manual.md.template`.
  - **Contents** list: add the new section.
  - **Acknowledgements** section before License: "The workflow skills are adapted from [superpowers](https://github.com/obra/superpowers) by Jesse Vincent (MIT)."

- [ ] **Step 2: Verify**

Run: `grep -c "dev_feature" README.md` → ≥2.
Run: `grep -c "one-shot by default\|opt out" README.md` → `0`.
Run: `grep -c "^## Acknowledgements" README.md` → `1`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "README: document workflow tiers, feature flow, mandatory cleanup, manual"
```

---

### Task 11: Push, then end-to-end test on scratch copies

**Files:** none in the repo (scratch directories only).

- [ ] **Step 1: Placeholder sweep in the repo**

Run: `grep -rn "TBD\|FIXME" .chisel/ README.md` → no output.
Run: `grep -L "Adapted from superpowers" .chisel/templates/workflow/*.md.template` → no output (every skill template has attribution).

- [ ] **Step 2: Push** (only after the user approves pushing)

```bash
git push -u origin feature/workflow-skills
```

- [ ] **Step 3: Build scratch projects** (outside the repo, e.g. under the session scratchpad):
  - `greenfield/` — empty git repo.
  - `withtests/` — tiny Python project: `pyproject.toml`, `src/app/listutil.py` with an unsorted-list function, `tests/test_listutil.py`, `pytest` configured.
  - `notests/` — same project without `tests/` and without pytest config.
  Copy `.chisel/`, `.claude/`, `.github/` from the repo into each (never move).

- [ ] **Step 4: Run the scenarios** — run `/dev-chisel` in each scratch project (a fresh assistant session opened in that directory; for Copilot, VS Code with Copilot Chat in agent mode):

| Scenario | Target | Pass when |
|---|---|---|
| greenfield | Claude | Phase 2 blocks until "ok"; all files of spec §4.2 exist; `.docs/todo.md` has kickoff TASK; manual.html opens; `.chisel/` and `dev-chisel` entries gone; workflow skills still present |
| withtests | Claude | Then in a new session: "sort the list in listutil" → Direct (no plan prompt, shows test output); "fix: X returns None for empty input" (planted bug) → Quick via `/dev_debug`; "add a CLI with import/export" → Feature proposal → spec.html → plan.html → execute → TASK in `tasks/done/` |
| withtests | Copilot | Prompts exist under `.github/prompts/`; `/dev_feature` → `/dev_plan` → `/dev_execute` completes (sequential is fine) |
| notests | either | `/dev_tdd` exists; invoking it proposes setting up tests first |
| resume | Claude | Interrupt `/dev_execute` after the first group; new session `/dev_execute` continues at the first unticked item |

- [ ] **Step 5: Cross-cutting checks on every generated scratch project**

Run: `grep -rn "{{" .claude .github .docs | grep -v "{{TODO" | grep -v ".docs/templates/"` → no output (no unresolved placeholders; `.docs/templates/` placeholders are intentional).
Run: `grep -rn "IF_CLAUDE\|IF_COPILOT\|IF_GREENFIELD" .claude .github .docs` → no output.
Open each `.html` in light and dark OS mode; confirm readability and the plan group view.

- [ ] **Step 6: Record results** — fix any failure on the branch (new commit per fix, re-run the failing scenario), then report the scenario table with pass/fail to the user. Merge to `main` only on the user's explicit approval.
