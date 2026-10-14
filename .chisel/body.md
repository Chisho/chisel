You are executing `/dev-chisel` — a project scaffolding tool that sets up configuration files tailored to the current project and the user's chosen AI coding assistant (Claude Code or GitHub Copilot). Templates live in `.chisel/templates/` (relative to the project root). The scaffolder is one-shot: after successful setup it always removes its own source so it doesn't pollute the target workspace.

## Phase 1: Explore the Project

Before asking anything, silently gather context. Do not assume what the project is — only record what you can actually confirm from the files.

1. Read whatever project manifest exists (examples: `package.json`, `Cargo.toml`, `project.godot`, `pyproject.toml`, `go.mod`, `Gemfile`, `composer.json`, `CMakeLists.txt`, etc.) to identify the tech stack
2. Run `git log --oneline -10` to understand recent activity
3. Scan the top-level directory structure
4. Check if `.claude/`, `.github/`, and `.docs/` already exist and what's inside them. Note which tool an existing setup belongs to — Phase 3 uses it to spot a conflict:
   - `.claude/` has content beyond the `dev-chisel` skill itself → existing **Claude Code** setup
   - `.github/copilot-instructions.md`, `.github/instructions/`, `.github/agents/`, or `.github/chatmodes/` exists → existing **Copilot** setup
5. **Check for a whole-repo copy.** If the project's README, LICENSE, `docs/superpowers/`, or git history belong to Chisel itself (e.g. a README titled "Chisel", commits about the scaffolder), the whole Chisel repo was copied instead of only `.chisel/`, `.claude/`, `.github/`. Stop before Phase 2 and tell the user: Chisel would describe itself as the project. Show the correct install (`cp -r /path/to/chisel/{.chisel,.claude,.github} .` into the real project) and ask whether to stop so they can clean up, or continue anyway. Never treat Chisel's own files as project content.
6. Classify project maturity based on what you found:
   - **Greenfield** — no project manifest and little to no source code (empty repo, or only README/LICENSE). Architecture docs will need to be defined rather than discovered.
   - **Existing** — has a manifest and/or meaningful source code. Architecture docs can be filled from exploration.
7. Detect the test setup (test directories, test scripts in the manifest, test framework config). Record the test command if found — it fills `{{TEST_INSTRUCTIONS}}` in the rules and the manual. If there is none, `{{TEST_INSTRUCTIONS}}` says so and points to `/dev_tdd`, which proposes setting tests up.

## Phase 2: Self-Cleanup Notice

Before any other questions, show this notice and wait for acknowledgement:

> **Heads-up — this scaffolder is one-shot.**
>
> When setup finishes, `/dev-chisel` **will** remove its own source from this project:
> - `.chisel/` — the scaffolder instructions and template library
> - `.claude/skills/dev-chisel/` — the Claude Code skill entry
> - `.github/prompts/dev-chisel.prompt.md` — the Copilot prompt entry
>
> Everything generated stays: the instruction index, rules, agents, the workflow and driver-mode commands, and `.docs/`. You will **not** be able to run `/dev-chisel` again in this project without reinstalling it.
>
> **Type "ok" to acknowledge and continue.**

If the user does not acknowledge, stop the run without creating any files. Cleanup is not optional — it runs in Phase 8.

## Phase 3: Pick Target Tool

Don't ask. Use the target the entry point gave you — the `dev-chisel` skill says Claude Code, the `dev-chisel` prompt says GitHub Copilot; only the matching tool can load each one.

- **Claude Code** — uses `.claude/` (CLAUDE.md, rules, agents)
- **GitHub Copilot** — uses `.github/` (copilot-instructions.md, instructions, custom agents)

**Conflict:** if Phase 1 found an existing setup for the *other* tool only (e.g. running in Copilot, but `.claude/` has a setup and `.github/` doesn't), ask one question: generate for the tool you're running in, or for the existing one?

**Override:** if the user explicitly asks for the other tool, use it.

This choice determines the target directories and file formats for the rest of the run. The `.docs/` folder is tool-agnostic and used either way.

## Phase 4: Ask What They Want

Present what you found:
> **Project detected:** [only what you actually confirmed — tech stack, language, framework]
> **Project state:** [greenfield / existing]
> **Target tool:** [Claude Code / Copilot] (detected)
> **Existing setup:** [none / partial / full — list what exists in the relevant dirs]
> **Self-cleanup at end:** yes (acknowledged)

Then ask:

> **What would you like to do?**
> 1. **Full setup** — Create the complete configuration (instructions, rules, agents, workflow commands, .docs)
> 2. **Partial setup** — Choose which pieces to create
> 3. **Add agents** — Add specific ones to an existing setup
> 4. **Modify existing** — Update or extend files that already exist

## Phase 5: Handle Existing Files

If ANY target file already exists:

1. List which files would be affected
2. Ask: *"These files already exist. What should I do?"*
   - **Skip** — Don't touch existing files, only create missing ones
   - **Overwrite** — Replace with new versions
   - **Backup + Overwrite** — Rename existing files with `.old` extension, then create new ones (e.g., `CLAUDE.md` → `CLAUDE.md.old`)

NEVER overwrite without explicit permission. This is a hard rule.

**Copilot, older setups:** if the `.github/chatmodes/` folder with `.chatmode.md` files exists (VS Code's former format), offer to migrate them: rename each to `<name>.agent.md`, move it to `.github/agents/`, and update its frontmatter as above. Current VS Code reads custom agents from `.github/agents/`.

`.docs/todo.md` and `.docs/todo_done.md` with existing entries are never replaced or reset, in every mode and whatever the user picks above: merge — add the new Convention lines (and a `TASK` section if missing) and keep every entry and tally as-is.

**The workflow bundle is all-or-nothing, in every mode** (including Partial setup and Modify existing): the six workflow commands, `.docs/templates/`, `.docs/tasks/`, the manual, and the rules sections `Workflow — Triage / Direct / Quick / Feature`, `Verify Before Claiming Done`, `Review Feedback`, plus the Driver-Mode bookkeeping line. If the user picks **Skip** for an existing rules file, ask once whether to append the missing sections to it; if yes, append the missing sections without touching the rest; if no, warn in Phase 9 that the workflow commands rely on those sections.

## Phase 6: Configure

### Task Types & Priorities

Ask the user what task types and priority scale they want for their todo system. Offer possibilities without prescribing — don't bucket them by project type. Examples of task categories they might pick from or combine: `TASK`, `BUG`, `CR`, `SEC`, `POLISH`, `DOCS`, `RESEARCH`, `CHORE`. Examples of priority schemes: `P0–P5`, `High / Medium / Low`, `Now / Next / Later`, or anything custom.

Let the user define what fits their project. A `TASK` type is always included — feature TASKs (`/dev_feature`) are numbered from its tally — so add it if the user's list doesn't have it, and say so.

### Agents

List every `*.md.template` in `.chisel/templates/agents/` and present each as a possibility — do not gate any of them by project type. The default set is:

- **architect** — system design, project structure, technical decisions
- **code-reviewer** *(read-only)* — code quality, correctness, best practices
- **frontend-developer** — UI implementation, styling, UX
- **backend-developer** — server logic, database, APIs
- **embedded-developer** — firmware, drivers, RTOS, hardware integration
- **security-expert** *(read-only)* — threat modeling, vulnerability analysis, secure-coding review

**`code-reviewer` is always included** — the Feature flow's `/dev_execute` uses it to review every work item. Show it as *included*, not as an option.

Ask: *"`code-reviewer` is always included. Which others would you like? You can also describe a custom one and I'll create it."*

Agents fall into two groups:
- **Code-modifying** (frontend-developer, backend-developer, embedded-developer) — tools include `Edit` and `Write`; the agent can make changes directly. These agents honor driver-mode by checking their task prompt for a *"mainseat"* directive: in **backseat** (default — no directive) they edit files directly; if the main session passes a mainseat directive, they walk the user through ready-to-paste snippets instead. Toggle the conversation's mode with `/dev_mainseat` and `/dev_backseat`.
- **Read-only** (architect, code-reviewer, security-expert) — tools are `Read, Grep, Glob, Bash` only. The agent plans, reviews, or recommends; the user applies changes. Driver-mode does not apply (these agents can't edit anyway).

Present the two groups as separate questions when your question tool limits the number of options (e.g. 4 per question): read-only agents in one, code-modifying agents in the other — never mixing the two groups in one question. Without such a limit, one list grouped under two headings is fine.

For user-added templates not named above, infer the group from the template's `tools:` frontmatter — templates whose tools include `Edit`/`Write` are code-modifying (and get the driver-mode paragraph); otherwise treat them as read-only.

### Workflow

No question — the Direct / Quick / Feature workflow and all six workflow commands (`/dev_feature`, `/dev_plan`, `/dev_execute`, `/dev_debug`, `/dev_tdd`, `/dev_explore`) are always generated.

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
- The user's chosen agents, task types, and rules
- Real file paths and system names from the codebase
- The target tool's file format and location conventions

**Exception — copied verbatim:** files under `.chisel/templates/driver-mode/`, and the three runtime templates `.chisel/templates/workflow/task-view.html.template`, `.chisel/templates/task/spec.md.template`, `.chisel/templates/task/plan.md.template`. Their `{{…}}` placeholders are filled later by the workflow commands, **not** now.

### Workflow conditionals

Templates may contain `{{IF_CLAUDE}}…{{/IF_CLAUDE}}`, `{{IF_COPILOT}}…{{/IF_COPILOT}}`, and `{{IF_GREENFIELD}}…{{/IF_GREENFIELD}}` blocks. Keep the content of blocks that apply to this run (target tool, project state), delete the others entirely, and always delete the markers themselves. A marker that sits on its own line is removed together with its line — generated workflow commands must start with `---` on line 1, or their frontmatter won't be recognized.

Workflow command templates (`.chisel/templates/workflow/dev_*.md.template`) are **not** adapted beyond that: resolve only the `{{IF_…}}` blocks and copy the rest as-is. Their other `{{…}}` text (`{{TITLE}}`, `{{SOURCE_FILE}}`, `{{TOC}}`, `{{CONTENT}}`, `{{TODO: …}}`) is literal instruction text used when the commands run.

**Important — `.chisel/` is scaffolder-internal, not project content.** Never describe `.chisel/`, `body.md`, the `dev-chisel` skill, or the templates folder as if they were the *project* in `CLAUDE.md` / `copilot-instructions.md`. They belong to the scaffolder, not the target project. When filling `{{PROJECT_DESCRIPTION}}` and `{{RUN_INSTRUCTIONS}}`:

- **Greenfield project** → both fields MUST be `{{TODO}}` placeholders pointing the user to `.docs/architecture.md`. Do not invent content. Do not infer the project's identity from the presence of `.chisel/` (which only indicates the scaffolder ran here).
- **Existing project** → read the manifest, README, and source to describe what the project actually does and how to run it. Use real entries from those files, not from the scaffolder source.
- The driver-mode toggles section stays in either case — it's project tooling, not scaffolder source.

### Files to generate — Claude Code target

- **`.claude/CLAUDE.md`** — Brief project overview + pointers to rules, agents, and docs. Based on `CLAUDE.md.template`.
- **`.claude/rules/rules.md`** — Auto-loaded rules with workflow, conventions, and project-specific constraints. Based on `rules.md.template`. Must include the Driver-Mode section from the template — that's how the main session knows to honor mainseat / backseat for the conversation.
- **`.claude/agents/*.md`** — One file per chosen agent, adapted from templates in `.chisel/templates/agents/`.
- **`.claude/skills/dev_mainseat/SKILL.md`** + **`.claude/skills/dev_backseat/SKILL.md`** — Driver-mode toggle skill entry points. Copy verbatim from `.chisel/templates/driver-mode/dev_mainseat.SKILL.md` and `.chisel/templates/driver-mode/dev_backseat.SKILL.md`.
- **`.claude/skills/<name>/SKILL.md`** for each of `dev_feature`, `dev_plan`, `dev_execute`, `dev_debug`, `dev_tdd`, `dev_explore` — workflow commands, from `.chisel/templates/workflow/<name>.md.template` (see *Workflow conditionals*).

### Files to generate — GitHub Copilot target

- **`.github/copilot-instructions.md`** — Brief project overview + conventions. Adapt from `CLAUDE.md.template`.
- **`.github/instructions/rules.instructions.md`** — Instructions file. Adapt from `rules.md.template`. Must include the Driver-Mode section from the template.
- **`.github/agents/<name>.agent.md`** — One VS Code custom agent per chosen agent, adapted from `.chisel/templates/agents/`: keep `name` and `description` in the frontmatter, map `tools` to Copilot tool names (read-only agents get only read/search tools; code-modifying agents also get edit tools), drop Claude-only fields (`model: inherit`, `color`), and replace `.claude/…` paths in the body with their Copilot equivalents.
- **`.github/prompts/dev_mainseat.prompt.md`** + **`.github/prompts/dev_backseat.prompt.md`** — Driver-mode toggle prompt entry points. Copy verbatim from `.chisel/templates/driver-mode/dev_mainseat.prompt.md` and `.chisel/templates/driver-mode/dev_backseat.prompt.md`.
- **`.github/prompts/<name>.prompt.md`** for each of `dev_feature`, `dev_plan`, `dev_execute`, `dev_debug`, `dev_tdd`, `dev_explore` — workflow commands, from `.chisel/templates/workflow/<name>.md.template` (see *Workflow conditionals*).

### Files to generate — shared (either target)

- **`.docs/architecture.md`** — Based on `architecture.md.template`, but the content depends on project state:
  - **Existing project** — Document the project structure, key systems, and patterns you discovered during exploration. Fill with REAL information from the codebase.
  - **Greenfield project** — Generate a scaffold with clearly-marked `{{TODO}}` placeholders for purpose, goals, key systems, tech choices, data model, and deployment. Do NOT invent content — the document is a form to fill in later, not fiction to write now.
- **`.docs/todo.md`** — Task tracker with the user's chosen types and priority scale. Based on `todo.md.template`.
  - **Greenfield projects only:** seed the tracker with a top-priority kickoff task, for example: *"Define project scope and initial architecture — fill in `.docs/architecture.md`"*. This gives the next working session a concrete starting point; `/dev_feature` promotes it to a feature TASK.
- **`.docs/todo_done.md`** — Empty completed task archive. Based on `todo_done.md.template`.
- **`.docs/tasks/.gitkeep`** + **`.docs/tasks/done/.gitkeep`** — empty files so the TASK folders exist in git.
- **`.docs/templates/task-view.html`**, **`.docs/templates/spec.md`**, **`.docs/templates/plan.md`** — copied verbatim from `.chisel/templates/workflow/task-view.html.template`, `.chisel/templates/task/spec.md.template`, `.chisel/templates/task/plan.md.template`. The workflow commands read them at runtime (`.chisel/` is gone by then).
- **`.docs/manual.md`** — the user manual, from `manual.md.template`. Fill `{{PROJECT_NAME}}`, `{{AGENTS_TABLE}}` (one row per generated agent: name · what it's for · edits code?), `{{TEST_INSTRUCTIONS}}` (same value as in the rules), and resolve its `{{IF_…}}` blocks (`{{IF_GREENFIELD}}` is kept only for greenfield projects).
- **`.docs/manual.html`** — using `.docs/templates/task-view.html` as the shell, write `.docs/manual.html` from `.docs/manual.md` (leave the template unchanged) the same way the workflow commands do (see the *Render & open* section of `.chisel/templates/workflow/dev_feature.md.template`): fill `{{TITLE}}`, `{{SOURCE_FILE}}`, `{{TOC}}`, `{{CONTENT}}`. Render only — it is opened in Phase 9.

## Phase 8: Self-Cleanup

Always runs (acknowledged in Phase 2), but only after Phase 7 completed. If generation failed partway, report what was and wasn't created and stop — without cleanup, so the user can re-run. Use a single platform-appropriate command (e.g. `rm -rf` on Unix-like shells) to delete:

- `.chisel/` (the entire directory — `body.md` and `templates/`)
- `.claude/skills/dev-chisel/` (the entire directory)
- `.github/prompts/dev-chisel.prompt.md`

After deletion, verify the listed paths no longer exist.

**Do NOT delete:**
- `.claude/skills/dev_mainseat/`, `.claude/skills/dev_backseat/` and `.claude/skills/dev_{feature,plan,execute,debug,tdd,explore}/` (Claude Code target)
- `.github/prompts/dev_mainseat.prompt.md`, `.github/prompts/dev_backseat.prompt.md` and `.github/prompts/dev_{feature,plan,execute,debug,tdd,explore}.prompt.md` (Copilot target)
- `.docs/` — including `.docs/templates/`, `.docs/tasks/`, `.docs/explore/` (if present), `.docs/manual.md`, `.docs/manual.html`
- Any other generated artifact (`CLAUDE.md`, agents, rules, etc.)

## Phase 9: Explain the System

1. Open `.docs/manual.html`: Windows `cmd /c start "" "<path>"` (works from PowerShell, cmd, and Git Bash), macOS `open "<path>"`, Linux `xdg-open "<path>"`. If that fails or there is no desktop, print the absolute path.
2. In chat, keep it short and loose — a few lines, no file-by-file walkthrough:
   - What was set up, one line per area: instructions, rules, agents, workflow commands, `.docs/`. Mention any files backed up (`.old`).
   - How to work: *"Just ask — I'll pick Direct, Quick, or Feature. For a feature I'll ask first."*
   - **Greenfield:** the kickoff TASK in `.docs/todo.md` is the first step — run `/dev_feature TASK-<number>` (name the actual number) to define the project.
   - Any files skipped in Phase 5, and whether the workflow sections are missing from a skipped rules file.
   - *"Everything else is in the manual that just opened (`.docs/manual.md`)."*
3. Confirm the scaffolder source was removed (*"Scaffolder source removed — `/dev-chisel` is gone; everything generated stays."*).

## Important Rules

- **Phase 2 acknowledgement is required before anything else** — no acknowledgement, no run. Cleanup always runs in Phase 8.
- **Read templates from `.chisel/templates/`** — don't hallucinate template content, actually read the files
- **Adapt templates to the target tool's conventions** — Copilot and Claude Code have different file formats and directory conventions; don't mix them
- **Never overwrite without permission** — always ask first
- **Adapt, don't copy** — templates are starting points, everything must be tailored. **Exceptions:** driver-mode files and the three `.docs/templates/` files are copied verbatim; workflow command templates only get their `{{IF_…}}` blocks resolved.
- **Don't prescribe project type** — offer possibilities, let the user define what the project is
- **Architecture doc must contain real information** — explore the codebase and document what you actually find, don't leave placeholders
- **Keep the index concise** — `CLAUDE.md` / `copilot-instructions.md` are indexes, not manuals. Detailed info goes in `.docs/`
- **Generated `rules.md` / `rules.instructions.md` MUST include the Driver-Mode section and the four `Workflow —` sections** (Triage, Direct, Quick, Feature) from the template — without them the main session won't know the driver-mode or how to triage requests.
- **Never leave `{{IF_…}}` markers in generated files.**
