You are executing `/dev-chisel` — a project scaffolding tool that sets up configuration files tailored to the current project and the user's chosen AI coding assistant (Claude Code or GitHub Copilot). Templates live in `.chisel/templates/` (relative to the project root). The scaffolder is one-shot: by default, after successful setup it removes its own source so it doesn't pollute the target workspace.

## Phase 1: Explore the Project

Before asking anything, silently gather context. Do not assume what the project is — only record what you can actually confirm from the files.

1. Read whatever project manifest exists (examples: `package.json`, `Cargo.toml`, `project.godot`, `pyproject.toml`, `go.mod`, `Gemfile`, `composer.json`, `CMakeLists.txt`, etc.) to identify the tech stack
2. Run `git log --oneline -10` to understand recent activity
3. Scan the top-level directory structure
4. Check if `.claude/`, `.github/`, and `.docs/` already exist and what's inside them. Use this to suggest a likely target in Phase 3:
   - If `.claude/` already has content beyond the `dev-chisel` skill itself → likely **Claude Code**
   - If `.github/copilot-instructions.md` or `.github/instructions/` exists → likely **Copilot**
   - If both or neither → no default, ask the user
5. Classify project maturity based on what you found:
   - **Greenfield** — no project manifest and little to no source code (empty repo, or only README/LICENSE). Architecture docs will need to be defined rather than discovered.
   - **Existing** — has a manifest and/or meaningful source code. Architecture docs can be filled from exploration.

## Phase 2: Self-Cleanup Notice

Before any other questions, surface this notice and record the user's answer for use in Phase 9:

> **Heads-up — this scaffolder is one-shot.**
>
> By default, after successful setup `/dev-chisel` removes its own source from this project so it doesn't pollute the workspace:
> - `.chisel/body.md` — the scaffolder instructions
> - `.chisel/templates/` — the template library
> - `.claude/skills/dev-chisel/` — the Claude Code skill entry
> - `.github/prompts/dev-chisel.prompt.md` — the Copilot prompt entry
>
> Everything generated this run stays: `CLAUDE.md` / `copilot-instructions.md`, rules, agents / chat modes, `.docs/`, and the driver-mode toggle skills (`/dev_mainseat`, `/dev_backseat`).
>
> Re-running `/dev-chisel` later would require reinstalling the scaffolder source.
>
> **Run self-cleanup at the end?** (Y / n — answer "n" to keep the scaffolder source, e.g. for iterative testing or future re-runs in this project)

Default is yes. Record the answer and continue to Phase 3 regardless of which they pick. The recorded preference is honored in Phase 9.

## Phase 3: Pick Target Tool

Ask the user which AI coding assistant this setup is for:

> **Target tool?**
> 1. **Claude Code** — uses `.claude/` (CLAUDE.md, rules, agents)
> 2. **GitHub Copilot** — uses `.github/` (copilot-instructions.md, instructions, chat modes)

If Phase 1 detected an existing setup, suggest that as the default but let the user override.

This choice determines the target directories and file formats for the rest of the run. The `.docs/` folder is tool-agnostic and used either way.

## Phase 4: Ask What They Want

Present what you found:
> **Project detected:** [only what you actually confirmed — tech stack, language, framework]
> **Project state:** [greenfield / existing]
> **Target tool:** [Claude Code / Copilot]
> **Existing setup:** [none / partial / full — list what exists in the relevant dirs]
> **Self-cleanup at end:** [yes / no]

Then ask:

> **What would you like to do?**
> 1. **Full setup** — Create the complete configuration (instructions, rules, agents/chat modes, .docs)
> 2. **Partial setup** — Choose which pieces to create
> 3. **Add agents / chat modes** — Add specific ones to an existing setup
> 4. **Modify existing** — Update or extend files that already exist

## Phase 5: Handle Existing Files

If ANY target file already exists:

1. List which files would be affected
2. Ask: *"These files already exist. What should I do?"*
   - **Skip** — Don't touch existing files, only create missing ones
   - **Overwrite** — Replace with new versions
   - **Backup + Overwrite** — Rename existing files with `.old` extension, then create new ones (e.g., `CLAUDE.md` → `CLAUDE.md.old`)

NEVER overwrite without explicit permission. This is a hard rule.

## Phase 6: Configure

### Task Types & Priorities

Ask the user what task types and priority scale they want for their todo system. Offer possibilities without prescribing — don't bucket them by project type. Examples of task categories they might pick from or combine: `TASK`, `BUG`, `CR`, `SEC`, `POLISH`, `DOCS`, `RESEARCH`, `CHORE`. Examples of priority schemes: `P0–P5`, `High / Medium / Low`, `Now / Next / Later`, or anything custom.

Let the user define what fits their project.

### Agents / Chat Modes

List every `*.md.template` in `.chisel/templates/agents/` and present each as a possibility — do not gate any of them by project type. The default set is:

- **architect** — system design, project structure, technical decisions
- **code-reviewer** *(read-only)* — code quality, correctness, best practices
- **frontend-developer** — UI implementation, styling, UX
- **backend-developer** — server logic, database, APIs
- **embedded-developer** — firmware, drivers, RTOS, hardware integration
- **security-expert** *(read-only)* — threat modeling, vulnerability analysis, secure-coding review

Ask: *"Which would you like? You can also describe a custom one and I'll create it."*

Agents fall into two groups:
- **Code-modifying** (frontend-developer, backend-developer, embedded-developer) — tools include `Edit` and `Write`; the agent can make changes directly. These agents honor driver-mode by checking their task prompt for a *"mainseat"* directive: in **backseat** (default — no directive) they edit files directly; if the main session passes a mainseat directive, they walk the user through ready-to-paste snippets instead. Toggle the conversation's mode with `/dev_mainseat` and `/dev_backseat`.
- **Read-only** (architect, code-reviewer, security-expert) — tools are `Read, Grep, Glob, Bash` only. The agent plans, reviews, or recommends; the user applies changes. Driver-mode does not apply (these agents can't edit anyway).

For user-added templates not named above, infer the group from the template's `tools:` frontmatter — templates whose tools include `Edit`/`Write` are code-modifying (and get the driver-mode paragraph); otherwise treat them as read-only.

### Project-Specific Rules

Ask: *"Are there any project-specific rules or conventions I should know about?"* Examples:
- Architectural patterns to follow
- Things to avoid
- Deployment constraints
- Domain-specific considerations

### Pre-Production Status

Ask: *"Is this project pre-production (no users, no data to migrate) or does it have production deployments?"*

This determines whether the "No Legacy" rules section is included.

## Phase 7: Generate

Using the templates from `.chisel/templates/` as starting points, generate all files. **Do not copy templates verbatim** — adapt every `{{PLACEHOLDER}}` and section to match:

- The project's actual tech stack, language, and framework (only what you confirmed)
- The project's directory structure and patterns
- The user's chosen agents / chat modes, task types, and rules
- Real file paths and system names from the codebase
- The target tool's file format and location conventions

**Exception:** files under `.chisel/templates/driver-mode/` are static (no placeholders) — copy them verbatim to their destinations.

**Important — `.chisel/` is scaffolder-internal, not project content.** Never describe `.chisel/`, `body.md`, the `dev-chisel` skill, or the templates folder as if they were the *project* in `CLAUDE.md` / `copilot-instructions.md`. They belong to the scaffolder, not the target project. When filling `{{PROJECT_DESCRIPTION}}` and `{{RUN_INSTRUCTIONS}}`:

- **Greenfield project** → both fields MUST be `{{TODO}}` placeholders pointing the user to `.docs/architecture.md`. Do not invent content. Do not infer the project's identity from the presence of `.chisel/` (which only indicates the scaffolder ran here).
- **Existing project** → read the manifest, README, and source to describe what the project actually does and how to run it. Use real entries from those files, not from the scaffolder source.
- The driver-mode toggles section stays in either case — it's project tooling, not scaffolder source.

### Files to generate — Claude Code target

- **`.claude/CLAUDE.md`** — Brief project overview + pointers to rules, agents, and docs. Based on `CLAUDE.md.template`.
- **`.claude/rules/rules.md`** — Auto-loaded rules with workflow, conventions, and project-specific constraints. Based on `rules.md.template`. Must include the Driver-Mode section from the template — that's how the main session knows to honor mainseat / backseat for the conversation.
- **`.claude/agents/*.md`** — One file per chosen agent, adapted from templates in `.chisel/templates/agents/`.
- **`.claude/skills/dev_mainseat/SKILL.md`** + **`.claude/skills/dev_backseat/SKILL.md`** — Driver-mode toggle skill entry points. Copy verbatim from `.chisel/templates/driver-mode/dev_mainseat.SKILL.md` and `.chisel/templates/driver-mode/dev_backseat.SKILL.md`.

### Files to generate — GitHub Copilot target

- **`.github/copilot-instructions.md`** — Brief project overview + conventions. Adapt from `CLAUDE.md.template`.
- **`.github/instructions/rules.instructions.md`** — Instructions file. Adapt from `rules.md.template`. Must include the Driver-Mode section from the template.
- **`.github/chatmodes/*.chatmode.md`** — One chat mode per chosen agent, adapted from `.chisel/templates/agents/` to Copilot's chat-mode format.
- **`.github/prompts/dev_mainseat.prompt.md`** + **`.github/prompts/dev_backseat.prompt.md`** — Driver-mode toggle prompt entry points. Copy verbatim from `.chisel/templates/driver-mode/dev_mainseat.prompt.md` and `.chisel/templates/driver-mode/dev_backseat.prompt.md`.

### Files to generate — shared (either target)

- **`.docs/architecture.md`** — Based on `architecture.md.template`, but the content depends on project state:
  - **Existing project** — Document the project structure, key systems, and patterns you discovered during exploration. Fill with REAL information from the codebase.
  - **Greenfield project** — Generate a scaffold with clearly-marked `{{TODO}}` placeholders for purpose, goals, key systems, tech choices, data model, and deployment. Do NOT invent content — the document is a form to fill in later, not fiction to write now.
- **`.docs/todo.md`** — Task tracker with the user's chosen types and priority scale. Based on `todo.md.template`.
  - **Greenfield projects only:** seed the tracker with a top-priority kickoff task, for example: *"Define project scope and initial architecture — fill in `.docs/architecture.md`"*. This gives the next working session a concrete starting point.
- **`.docs/todo_done.md`** — Empty completed task archive. Based on `todo_done.md.template`.

## Phase 8: Summary

After generating, show the user:
- List of all files created (with paths)
- List of all files backed up (if any)
- Quick explanation of what each file does
- If the project is greenfield, point out the seeded kickoff task in `.docs/todo.md` as the concrete next step — defining the project is the first real piece of work
- Surface the driver-mode toggles: `/dev_mainseat` flips the conversation to "user drives, agent provides snippets" mode; `/dev_backseat` reverts to the default "agent edits, user reviews". Note that mode is conversation-scoped — every new conversation starts in backseat.
- State whether self-cleanup is about to run (per the Phase 2 answer):
  - **Yes** → "About to remove the scaffolder source. Re-running `/dev-chisel` later requires reinstalling it."
  - **No** → "Self-cleanup skipped — scaffolder source is preserved for re-runs. Run `/dev-chisel` again later to add agents / chat modes or modify the setup."

## Phase 9: Self-Cleanup

If the user answered **yes** in Phase 2, perform the cleanup now. Use a single platform-appropriate command (e.g. `rm -rf` on Unix-like shells) to delete:

- `.chisel/body.md`
- `.chisel/templates/` (the entire directory and its contents)
- `.claude/skills/dev-chisel/` (the entire directory)
- `.github/prompts/dev-chisel.prompt.md`

Since `.chisel/` contains nothing else after this run (no state file, no body files), `.chisel/` itself becomes empty and may be removed too.

After deletion, verify the listed paths no longer exist and report briefly to the user (e.g. *"Scaffolder source removed. The toggle skills, generated config, and `.docs/` remain."*).

**Do NOT delete:**
- `.claude/skills/dev_mainseat/`, `.claude/skills/dev_backseat/` — toggle skill entry points (Claude Code target)
- `.github/prompts/dev_mainseat.prompt.md`, `.github/prompts/dev_backseat.prompt.md` — toggle prompt entry points (Copilot target)
- Any other generated artifact (`CLAUDE.md`, agents, rules, `.docs/`, etc.)

If the user answered **no** in Phase 2, this phase is skipped — leave the scaffolder source intact.

## Important Rules

- **Phase 2 confirmation runs before any generation** — record the user's self-cleanup preference, but proceed with the rest of the scaffolder regardless of which they pick.
- **Read templates from `.chisel/templates/`** — don't hallucinate template content, actually read the files
- **Adapt templates to the target tool's conventions** — Copilot and Claude Code have different file formats and directory conventions; don't mix them
- **Never overwrite without permission** — always ask first
- **Adapt, don't copy** — templates are starting points, everything must be tailored. **Exception:** files under `.chisel/templates/driver-mode/` are static — copy verbatim.
- **Don't prescribe project type** — offer possibilities, let the user define what the project is
- **Architecture doc must contain real information** — explore the codebase and document what you actually find, don't leave placeholders
- **Keep the index concise** — `CLAUDE.md` / `copilot-instructions.md` are indexes, not manuals. Detailed info goes in `.docs/`
- **Generated `rules.md` / `rules.instructions.md` MUST include the Driver-Mode section** from the template — without it the main session won't know to honor mainseat / backseat.
