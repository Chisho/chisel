# Chisel

**AI scaffolding for any codebase.** Chisel is a one-shot scaffolder that onboards an AI coding assistant — [Claude Code](https://claude.com/claude-code) or [GitHub Copilot](https://github.com/features/copilot) — to your project. Run one slash command, answer a few questions, and Chisel generates a tailored configuration: project instructions, workflow rules, specialized agents, a feature workflow (brainstorm → spec → plan → parallel execution), a task tracker, and living architecture docs — all adapted to your actual tech stack.

Think of it as onboarding a new teammate. Instead of an AI assistant guessing at your conventions on every task, it starts with the context, rules, and roles you'd give a human developer.

> Chisel was built for Claude Code first; GitHub Copilot is a first-class second target. Both are driven by a single source of truth, so the two setups stay in sync.

---

## Contents

- [Why](#why)
- [How it works](#how-it-works)
- [Works on any project — empty or existing](#works-on-any-project--empty-or-existing)
- [What gets generated](#what-gets-generated)
- [Specialized agents](#specialized-agents)
- [Workflow: Direct, Quick, Feature](#workflow-direct-quick-feature)
- [Driver mode: mainseat vs. backseat](#driver-mode-mainseat-vs-backseat)
- [Installation](#installation)
- [Usage](#usage)
- [Design principles](#design-principles)
- [Repository layout](#repository-layout)
- [Acknowledgements](#acknowledgements)
- [License](#license)

---

## Why

Out of the box, an AI assistant knows nothing about *your* project — your architecture, your conventions, what to avoid, how you track work. You end up re-explaining the same context in every session.

Chisel front-loads that once. It explores your repo, asks what you want, and writes the config files your assistant reads automatically. The result is an assistant that:

- Knows your stack, structure, and patterns from the first message
- Follows the rules and constraints you set, not generic defaults
- Can delegate to specialized roles (architect, reviewer, backend, frontend, embedded, security)
- Tracks tasks in a structured to-do system you define
- Keeps architecture docs as a source of truth instead of scattered tribal knowledge
- Stays token-light — pulls in only what the current prompt needs, not your whole project

### Token-light by design

Chisel deliberately avoids dumping your entire project into context on every message. The instruction file (`CLAUDE.md` / `copilot-instructions.md`) is a lean **index**, not a manual — a few pointers to where things live. Detailed context sits in `.docs/` and loads **on demand**: the assistant reads `architecture.md`, a data model, or a runbook only when the task at hand calls for it. Agents work the same way — a specialized role is spun up only when you delegate to it. The result is a setup that gives the assistant deep context without burning tokens carrying all of it around all the time.

---

## How it works

Chisel is **pure Markdown** — no CLI, no package to install, no runtime dependencies, nothing of its own to execute. It ships as a slash command / prompt plus a template library: you copy a few files into your repo, run it once, and it always self-cleans afterward — leaving behind only the generated config. The entire run happens inside your assistant, so your code goes only where Claude Code or Copilot already sends it.

```
/dev-chisel
```

*(installed as a **skill** in Claude Code, a **prompt** in GitHub Copilot)*

Under the hood, a single instruction file (`.chisel/body.md`) drives both the Claude Code skill and the Copilot prompt, so behavior is identical regardless of target tool.

### The run, phase by phase

1. **Explore** — reads your manifest (`package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, …), git history, directory layout, and test setup. Records only what it can confirm; nothing is assumed. Classifies the project as **greenfield** (little/no code — architecture to be defined) or **existing** (architecture to be discovered).
2. **Self-cleanup notice** — tells you up front that the scaffolder will remove its own source at the end, and asks you to acknowledge it. No acknowledgement, no run.
3. **Pick target tool** — Chisel detects which tool it's running in (Claude Code → `.claude/`, GitHub Copilot → `.github/`) and uses it without asking. It asks only if the project already has a setup for the other tool, and you can always ask for the other one.
4. **Ask what you want** — full setup, partial setup, add agents, or modify an existing config.
5. **Handle existing files** — if anything would be overwritten, Chisel asks first (skip / overwrite / back-up-and-overwrite). **It never overwrites without permission**, and an existing todo list keeps every entry.
6. **Configure** — you choose task types & priority scheme, which agents to add (`code-reviewer` is always included), project-specific rules, and whether the project is pre-production. The workflow itself is always included.
7. **Generate** — writes the files, adapting every template to your real stack, paths, and choices. Templates are starting points, not copied verbatim — the exceptions are the driver-mode toggles and the spec / plan / HTML-view formats, which are copied as-is.
8. **Self-cleanup** — removes the scaffolder source, leaving a clean workspace.
9. **Explain the system** — opens a short user manual (`.docs/manual.html`) and sums up how to work in a few lines.

---

## Works on any project — empty or existing

Chisel detects your project's state during Phase 1 and adapts what it writes:

- **Empty / greenfield repo** (no manifest, little or no code) — Chisel doesn't invent a project that isn't there. It generates the config *scaffold* with clearly-marked `{{TODO}}` placeholders for the things only you can decide: project purpose, run instructions, goals, key systems, tech choices, data model, and deployment. `.docs/architecture.md` becomes a fill-in-the-blanks form, and `.docs/todo.md` is seeded with a top-priority kickoff task (e.g. *"Define project scope and initial architecture — fill in the architecture doc"*) so your first session has a concrete starting point.
- **Existing project** — Chisel reads your manifest, README, source, and git history, and fills the same files with **real** discovered information: what the project does, how to run it, its structure, key systems, and patterns. No placeholders — the architecture doc documents what actually exists.

Either way you get the same file set; only the content differs — a form to fill in for a fresh repo, a documented map for an established one.

---

## What gets generated

### Claude Code target

```
.claude/
├── CLAUDE.md                  # Project overview + pointers to rules, agents, docs
├── rules/
│   └── rules.md               # Auto-loaded workflow rules, conventions, constraints
├── agents/                    # One file per chosen specialized agent
│   └── *.md
└── skills/
    ├── dev_feature/           # Workflow commands (see Workflow below)
    ├── dev_plan/
    ├── dev_execute/
    ├── dev_debug/
    ├── dev_tdd/
    ├── dev_explore/
    ├── dev_mainseat/          # Driver-mode toggle (see below)
    └── dev_backseat/

.docs/
├── manual.md / manual.html    # Short user manual for this setup
├── architecture.md            # Project structure, key systems, data flow
├── todo.md                    # Task tracker (your types + priority scheme)
├── todo_done.md               # Completed-task archive
├── tasks/                     # One folder per feature TASK (spec, plan, HTML views)
├── explore/                   # Saved /dev_explore reports (created on demand)
│   └── done/                  # Finished TASKs
└── templates/                 # Spec / plan / HTML-view formats used by the workflow
```

### GitHub Copilot target

```
.github/
├── copilot-instructions.md    # Project overview + conventions
├── instructions/
│   └── rules.instructions.md  # Workflow rules and constraints
├── agents/                    # One custom agent per chosen agent
│   └── *.agent.md
└── prompts/
    ├── dev_feature.prompt.md   # Workflow commands
    ├── dev_plan.prompt.md
    ├── dev_execute.prompt.md
    ├── dev_debug.prompt.md
    ├── dev_tdd.prompt.md
    ├── dev_explore.prompt.md
    ├── dev_mainseat.prompt.md  # Driver-mode toggles
    └── dev_backseat.prompt.md

.docs/                         # Same tool-agnostic docs as above
```

The `.docs/` folder is identical either way — architecture, todo, TASK folders, and the manual are tool-agnostic.

### `.docs/` is your living source of truth

`.docs/` isn't a one-time dump — it's the project's persistent memory that your assistant reads on demand and keeps up to date as the work evolves. Chisel seeds it based on the project it's installed on (real structure for an existing codebase, `{{TODO}}` scaffolds for a fresh one), but from there it's **yours to grow**:

- **Extensible** — add whatever the project needs alongside the defaults: `data-model.md`, `api.md`, `deployment.md`, `decisions/` (ADRs), domain glossaries, runbooks. Anything you drop in `.docs/` becomes context your assistant can pull in.
- **Source of truth** — every agent reads `.docs/architecture.md` (plus the project rules) before starting work, and the built-in workflow keeps `.docs/todo.md` / `todo_done.md` up to date as tasks complete — so the docs, not guesswork or stale chat history, drive decisions. Keep them accurate and every session starts from the same picture.
- **Tool-agnostic** — the same `.docs/` serves both the Claude Code and Copilot setups, so switching or sharing tools doesn't fragment your project knowledge.

---

## Specialized agents

Each agent becomes a Claude Code subagent or a Copilot custom agent. **The set below is extensible — these are simply the roles I use.** Pick any combination, drop the ones you don't need, or describe a custom one (*"a data-pipeline engineer"*, *"a technical writer"*) and Chisel generates it from the same template pattern. Nothing is gated by project type; you choose what fits your work.

| Agent                        | Role                                                          | Access    |
| ---------------------------- | ------------------------------------------------------------- | --------- |
| **architect**          | System design, project structure, technical decisions         | Read-only |
| **code-reviewer**      | Code quality, correctness, best practices — **always included** | Read-only |
| **security-expert**    | Threat modeling, vulnerability analysis, secure-coding review | Read-only |
| **frontend-developer** | UI implementation, styling, UX                                | Can edit  |
| **backend-developer**  | Server logic, database, APIs                                  | Can edit  |
| **embedded-developer** | Firmware, drivers, RTOS, hardware integration                 | Can edit  |

**Rolling your own:** agent templates live in `.chisel/templates/agents/` as `{{PLACEHOLDER}}`-driven Markdown. Add a new `.md.template` there before running `/dev-chisel` and it's offered alongside the defaults (after a run the scaffolder is removed, so reinstall Chisel to add more later); or just ask for a custom role during a run and Chisel writes it inline. Read-only agents get `Read, Grep, Glob, Bash`; code-modifying agents also get `Edit`/`Write` and honor driver mode.

---

## Workflow: Direct, Quick, Feature

Chisel's workflow scales with the size of the request. Before acting, the assistant triages every request with two questions:

1. **Does any feature signal fire?** — a new feature or capability; spans several subsystems or ~5+ files; open design decisions; naturally splits into independent pieces. If yes, it *proposes* the Feature flow in one line and waits for you.
2. **Do all direct conditions hold?** — a localized change, exactly specified, no design choice, no new dependency. If yes, Direct.

Otherwise it's Quick.

| Tier | For | What happens |
|---|---|---|
| **Direct** | *"Sort this list in `parse_rows`"* | Change → verify → one-line report. No plan, no approval, no todo entry. |
| **Quick** | Small tasks and bugs | Explore → plan in chat → approve → implement → test → housekeeping. |
| **Feature** | New features, multi-part work | `/dev_feature` → `/dev_plan` → `/dev_execute`, tracked as a TASK. |

When unsure, the assistant picks the heavier tier, and escalation only goes up. You can always override: *"just do it"* or `/dev_feature`.

### The Feature flow

- **`/dev_feature`** — brainstorms with you one question at a time, proposes approaches, and writes `spec.md` into `.docs/tasks/TASK-NNN-<slug>/`.
- **`/dev_plan`** — splits the spec into work items, each with an agent, dependencies, exact files, and a *done when* check, grouped so items in a group never touch the same file and can run in parallel.
- **`/dev_execute`** — dispatches each group to the agents (in parallel where the tool supports it), has `code-reviewer` check every item, tracks progress in `todo.md`, verifies the acceptance criteria, and closes the TASK. It's resumable from the first unfinished item.

Each step waits for your approval and offers the next. The TASK's line in `todo.md` shows its status (`brainstorming` → `specced` → `planned` → `in-progress n/m` → `done`) and links to its folder; small entries stay one-liners.

**Readable specs.** Markdown is the source of truth, but every spec and plan is also rendered to a self-contained HTML page (`spec.html`, `plan.html` — light/dark, table of contents, plan laid out by parallel group) and opened in your browser for review. No scripts or build step — the assistant renders it.

**Discipline commands**, usable in any tier: **`/dev_debug`** (root cause before fix, regression test, two-strike rule) and **`/dev_tdd`** (red → green → refactor; proposes setting up tests if the project has none).

**`/dev_explore`** — understand code before changing it. Give it a path, a feature, a TASK, or a question (or nothing, and it asks). It answers in chat with a summary, the flow step by step, key files with `path:line` references, connections, observations, and where `.docs/` has drifted from the code. Read-only; at the end it offers to save the answer as a report in `.docs/explore/` with an HTML view.

The workflow is generated into your project — it doesn't depend on any plugin, so Claude Code and Copilot get the same thing.

---

## Driver mode: mainseat vs. backseat

Chisel installs two toggle commands that control *who holds the keyboard* for a conversation:

- **`/dev_backseat`** *(default)* — the assistant and its code-modifying agents edit files directly; you review the results.
- **`/dev_mainseat`** — the assistant does **not** edit files. It walks you through changes and hands you ready-to-paste snippets, so you stay in control of every write.

Driver mode is **conversation-scoped** — every new conversation starts in backseat. Re-invoke `/dev_mainseat` whenever you want to drive. Read-only agents (architect, reviewer, security) ignore the setting since they can't edit anyway.

---

## Installation

**Requirements:** [Claude Code](https://claude.com/claude-code), or **VS Code with GitHub Copilot Chat** — the Copilot entry point and custom agents use VS Code's [prompt files](https://code.visualstudio.com/docs/copilot/customization/prompt-files) and [custom agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents), so Copilot in other IDEs (JetBrains, Xcode, Eclipse) won't pick them up. Chisel itself is plain Markdown — nothing to build or install.

Chisel installs *into the project you want to scaffold*. **Copy only these three folders — not the whole Chisel repo.** Chisel's own README, LICENSE, docs, and git history would make it think the project *is* Chisel (it stops and warns you if that happens). Copy the scaffolder source into your repo root:

```
your-project/
├── .chisel/
│   ├── body.md                # single source of truth
│   └── templates/             # template library
├── .claude/
│   └── skills/dev-chisel/     # Claude Code entry point
└── .github/
    └── prompts/dev-chisel.prompt.md   # GitHub Copilot entry point
```

You only need the entry point for the tool you're targeting, but copying both is harmless.

**Quick copy:** clone this repo and drop those three folders into your project — for example, from the root of the project you want to scaffold:

```bash
cp -r /path/to/chisel/{.chisel,.claude,.github} .
```

The brace expansion is bash/zsh syntax — on Windows, run it from Git Bash (bundled with Git for Windows), or copy the `.chisel`, `.claude`, and `.github` folders in manually.

Then open the project in your assistant and run `/dev-chisel`.

Because Chisel always self-cleans after a successful run (you acknowledge this at the start), the scaffolder source disappears from the target project once it's done its job — no clutter left behind. Re-running later just means re-installing these files.

---

## Usage

1. Install the scaffolder source into your project (above).
2. Run **`/dev-chisel`** in Claude Code or GitHub Copilot.
3. Acknowledge the self-cleanup notice, then answer the prompts (target tool, task types, agents, rules, pre-production status).
4. Read the manual that opens at the end (`.docs/manual.html`), and review the generated `CLAUDE.md` / `copilot-instructions.md`, `rules`, agents, and `.docs/`.
5. Start working — your assistant now has full project context.

Works the same on an empty repo or an established codebase — see [Works on any project](#works-on-any-project--empty-or-existing) for how the generated content differs between the two.

---

## Design principles

- **Single source of truth** — one instruction file drives both Claude Code and Copilot; the two never drift apart.
- **Confirm, don't assume** — Chisel records only what it can verify from your files, and never overwrites without asking.
- **Adapt, don't template-dump** — generated files are tailored to your real stack, paths, and choices.
- **Index, not manual** — `CLAUDE.md` / `copilot-instructions.md` stay lean pointers; detail lives in `.docs/`.
- **Clean up after itself** — always one-shot; you acknowledge it up front, and only the config you'll actually use is left behind.
- **Gates where they matter** — trivial changes just happen; features get a spec and a plan you approve.

---

## Repository layout

```
.chisel/
├── body.md                    # phase-by-phase scaffolder instructions (source of truth)
└── templates/                 # templates Chisel adapts during a run
    ├── CLAUDE.md.template
    ├── rules.md.template
    ├── architecture.md.template
    ├── todo.md.template
    ├── todo_done.md.template
    ├── manual.md.template     # user manual
    ├── agents/                # agent templates (Claude subagents / Copilot custom agents)
    ├── workflow/              # dev_feature / plan / execute / debug / tdd / explore + HTML view
    ├── task/                  # spec / plan formats for TASK folders
    └── driver-mode/           # static mainseat / backseat toggle files
.claude/
└── skills/dev-chisel/         # Claude Code skill entry point
.github/
└── prompts/dev-chisel.prompt.md   # GitHub Copilot prompt entry point
```

---

## Acknowledgements

The workflow commands (except `/dev_explore`, written in the same style) are adapted from [superpowers](https://github.com/obra/superpowers) by Jesse Vincent (MIT), rewritten to be token-light, integrated with Chisel's `.docs/` task system, and generated for both Claude Code and GitHub Copilot.

---

## License

MIT — see [LICENSE](LICENSE). Use it, copy it, adapt it freely — Chisel is meant to be copied into your projects.

---

*Chisel — carve the context your AI assistant needs, once.*
