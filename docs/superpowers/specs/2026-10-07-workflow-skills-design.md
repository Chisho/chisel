# Workflow Skills for Chisel — Design

**Date:** 2026-10-07
**Branch:** `feature/workflow-skills`
**Status:** Awaiting user review

## 1. Goal

Make Chisel generate its own structured development workflow — brainstorm → spec → plan → parallel execution, plus debugging and TDD discipline — as project-local skills (Claude Code) and prompts (GitHub Copilot). The generated setup must not depend on the superpowers plugin, so Copilot users get the same workflow Claude users get.

The workflow is integrated with Chisel's `.docs/` todo system: feature-sized TASKs are the unit of planned work, each broken into work items that subagents can execute in parallel.

## 2. Context

- Chisel today: pure-Markdown scaffolder. `.chisel/body.md` (single source of truth) drives the `dev-chisel` Claude skill and Copilot prompt. Templates in `.chisel/templates/` are adapted to the target project. Generated output: instruction index, `rules.md`, agents/chat modes, driver-mode toggles, `.docs/` (architecture, todo, todo_done).
- Chisel's current workflow (in `rules.md.template`) is a single flow: Explore → Plan → Implement → Test → Housekeeping, suited to small iterative todo entries.
- The superpowers plugin (MIT, © 2025 Jesse Vincent) provides the inspiration. Its skills are Claude-specific (Skill/Agent tools, session-start hook) and long (200–570 lines). Chisel will **rewrite** them in its own token-light style, not copy them.

## 3. Decisions

| # | Decision | Rationale |
|---|---|---|
| D1 | Rewrite skills Chisel-native, credited "adapted from superpowers (MIT)" | Token-light, integrated with `.docs/`, one source for both targets |
| D2 | Two-tier task model: todo entry + `.docs/tasks/TASK-NNN-<slug>/` folder | Keeps `todo.md` lean; gives specs/plans/progress a home |
| D3 | Three workflow tiers: Direct / Quick / Feature | Gates only where they add value; avoids rubber-stamp approvals |
| D4 | Triage is rule-based and *proposes* the Feature flow; user confirms or overrides | Avoids superpowers-style over-triggering; works on Copilot (no hooks) because the rule lives in the always-loaded rules file |
| D5 | Markdown is the source of truth; HTML is a generated, auto-opened view (spec, plan, manual) | Cheap for agents, readable for humans, no drift |
| D6 | Everything is enforced except the agent choice: all workflow skills (incl. `dev_tdd`) are always generated, and `code-reviewer` is always included. The user only chooses which other agents to add | One consistent system; the feature flow depends on the reviewer |
| D7 | Self-cleanup is mandatory; Phase 2 becomes an acknowledgement, not a yes/no | One-shot scaffolder; user explicitly acknowledges losing `/dev-chisel` |
| D8 | New final phase explains the system loosely and opens a generated user manual | Onboarding the user, not just the assistant |
| D9 | One template per skill with `{{IF_CLAUDE}}` / `{{IF_COPILOT}}` blocks | Skills are long; two static copies (driver-mode style) would drift |

## 4. Design

### 4.1 Scaffolder layout (ships with Chisel)

```
.chisel/templates/
├── workflow/                       # NEW
│   ├── dev_feature.md.template
│   ├── dev_plan.md.template
│   ├── dev_execute.md.template
│   ├── dev_debug.md.template
│   ├── dev_tdd.md.template
│   └── task-view.html.template     # HTML shell for spec / plan / manual views
├── task/                           # NEW — TASK folder skeleton
│   ├── spec.md.template
│   └── plan.md.template
├── manual.md.template              # NEW — user manual
├── rules.md.template               # CHANGED — triage + three tiers + folded principles
├── todo.md.template                # CHANGED — feature-TASK entry format
├── CLAUDE.md.template              # CHANGED — pointers to workflow skills, tasks/, manual
└── (other templates unchanged)
```

Workflow templates contain `{{IF_CLAUDE}}…{{/IF_CLAUDE}}` and `{{IF_COPILOT}}…{{/IF_COPILOT}}` blocks for tool names, subagent launching, and frontmatter. Phase 7 keeps the block for the chosen target and drops the other. Each template carries a header comment: `<!-- Adapted from superpowers by Jesse Vincent (MIT) -->`.

### 4.2 Generated layout (in the target project)

```
Claude:   .claude/skills/{dev_feature,dev_plan,dev_execute,dev_debug,dev_tdd}/SKILL.md
Copilot:  .github/prompts/{dev_feature,dev_plan,dev_execute,dev_debug,dev_tdd}.prompt.md

.docs/
├── manual.md  manual.html          # user manual
├── templates/                     # spec.md, plan.md, task-view.html — used by the skills at runtime
│                                   # (copied here because .chisel/ is deleted by cleanup)
├── todo.md  todo_done.md
└── tasks/
    ├── TASK-NNN-<slug>/
    │   ├── spec.md  spec.html
    │   └── plan.md  plan.html
    └── done/                       # completed TASK folders
```

Existing generated files (instruction index, rules, agents/chat modes, driver-mode toggles, architecture) remain.

### 4.3 body.md phase changes

| # | Phase | Change |
|---|---|---|
| 1 | Explore | Also detect test setup (test dirs, test scripts in manifest, test frameworks) — used to fill `{{TEST_INSTRUCTIONS}}` and the manual |
| 2 | Cleanup notice | Becomes a mandatory acknowledgement: cleanup **will** run; user loses `/dev-chisel` unless reinstalled. No acknowledgement → stop the run |
| 3–5 | Target, scope, existing files | Unchanged |
| 6 | Configure | No workflow or skill questions — the workflow is always generated. Agent selection: `code-reviewer` is always included (shown as included, not offered); the user chooses any additional agents. Task types, project rules, and pre-production questions are unchanged |
| 7 | Generate | + workflow skills, `.docs/tasks/` (with `done/`), `manual.md` + `manual.html`; updated rules/todo/index |
| 8 | Self-cleanup | Moved here (was 9); always runs. Must not delete generated workflow skills/prompts |
| 9 | Explain the system | NEW. Open `manual.html`; give a short in-chat summary (what was set up, how to start, where the manual is) |

`.gitkeep` files keep `.docs/tasks/` and `.docs/tasks/done/` in git when empty.

The existing "Summary" content (files created, backups) is folded into Phase 9's summary.

### 4.4 Workflow tiers and triage (rules.md)

Triage runs before every request, in order:

1. **Feature signals** — any of: new feature or capability; spans several subsystems or ~5+ files; unresolved design decisions; naturally splits into independent pieces. If any fires → propose the Feature flow in one line and wait.
2. **Direct conditions** — all of: localized change (one function / a few lines / one file); request states exactly what to do; no design choice; no new dependency. If all hold → Direct.
3. Otherwise → Quick.

| Tier | Behavior |
|---|---|
| **Direct** | Do it → verify (relevant test, or build/lint) → report in 1–2 lines. No plan, no approval, no todo entry. Escalate to Quick immediately if it turns out bigger. |
| **Quick** | Existing flow: Explore → Plan (in chat) → approve → Implement → Test → Housekeeping. Used for any todo entry that isn't a feature TASK. |
| **Feature** | `dev_feature` → `dev_plan` → `dev_execute` (see 4.6). |

Rules:
- When unsure between tiers, pick the heavier one. Escalation is one-way within a task.
- User overrides both ways: "just do it" keeps it lighter; `/dev_feature` forces Feature.
- `BUG` entries use `dev_debug` in Quick or Feature.
- Driver mode applies to all tiers (mainseat → snippets, including Direct).

Folded principles added to rules.md:
- **Verify before claiming done** — run the check and show the output; no "should work".
- **Review feedback is verified, not obeyed** — check a suggestion against the code before applying it; push back with reasons if it's wrong.

### 4.5 TASK model

**todo.md entry (feature TASK):**
```
- **TASK-012** [P1] User authentication — `in-progress 3/7` → .docs/tasks/TASK-012-user-auth/
```
Quick entries keep today's format. TASK numbering uses the existing tally convention.

**Status progression:** `brainstorming` → `specced` → `planned` → `in-progress n/m` → `done`.

**spec.md sections:** Goal · Context · Decisions · Design · Out of scope · Acceptance criteria · Open questions.

**plan.md work item:**
```
### WI-3 — Session middleware   [group B · agent: backend-developer]
Depends on: WI-1
Files: src/auth/session.ts, src/middleware/index.ts
Done when: <concrete, checkable condition>
- [ ] steps…
```
Constraints: items in the same group touch disjoint files; groups execute in order (A → B → C); every acceptance criterion maps to ≥1 work item.

**Lifecycle:**

| Step | Skill | Writes | User gate |
|---|---|---|---|
| Start | triage or `/dev_feature` | todo entry (`brainstorming`), folder | confirm Feature |
| Brainstorm | `dev_feature` | spec.md + spec.html → `specced` | approve spec |
| Plan | `dev_plan` | plan.md + plan.html → `planned` | approve plan |
| Execute | `dev_execute` | code; ticks WIs; `in-progress n/m` | none per item; stops on blockers |
| Close | `dev_execute` | entry → `todo_done.md`; folder → `tasks/done/`; `.docs/` updated | confirm done |

Each skill offers the next one after approval.

### 4.6 Skill behavior

Target length: ~80–150 lines each. Every skill first reads `.docs/architecture.md` and the rules file.

**`dev_feature`** — create todo entry + folder → explore relevant code → clarifying questions one at a time (prefer multiple choice; purpose, constraints, success criteria) → propose 2–3 approaches with a recommendation → write spec.md → self-review (placeholders, contradictions, ambiguity, scope; suggest splitting into multiple TASKs if too large) → render + open spec.html → wait for approval → status `specced` → offer `dev_plan`.

**`dev_plan`** — read spec → derive work items with agent, deps, files, done-when → assign parallel groups and check file overlap within each group → check every acceptance criterion is covered → render + open plan.html → wait for approval → status `planned` → offer `dev_execute`.

**`dev_execute`** —
- For each group in order: dispatch each WI to its agent with the WI text, spec, and driver-mode directive and the instruction to follow `dev_tdd` for code changes. Parallel where the tool supports it; sequential otherwise.
- After each WI: `code-reviewer` checks it against "Done when". On fail: one rework round; second fail → stop and report.
- On pass: tick WI in plan.md, update `n/m` in todo.md, regenerate plan.html.
- After all groups: verify acceptance criteria with real output; update `.docs/`; ask user to confirm; close (see lifecycle).
- Resumable: on re-run, continue from the first unticked WI.
- If the user's chosen agents don't include a suitable one for a WI, the main session executes it itself.

**`dev_debug`** — reproduce → gather evidence → hypothesis → smallest test of hypothesis → fix root cause → regression test where possible → verify. After 2 failed fix attempts, stop and re-examine assumptions with the user instead of stacking fixes.

**`dev_tdd`** (always generated) — red → green → refactor. A test must be seen failing before the implementation is written. If the project has no test setup yet, the skill stops and proposes setting one up (as a Quick task, or a Feature TASK if substantial) before continuing.

### 4.7 HTML views

- `task-view.html.template`: one self-contained page — inline CSS, light/dark via `prefers-color-scheme`, readable typography, table of contents, banner "Generated from `<file>.md` — edit the Markdown, not this file".
- The assistant renders Markdown → HTML into the template itself. No scripts, libraries, or build step.
- `plan.html` additionally shows work items arranged by parallel group with dependencies.
- Regenerated whenever the source `.md` changes.
- Opened via the platform command (`start ""` on Windows, `open` on macOS, `xdg-open` on Linux); if opening fails, print the path.

### 4.8 Target differences

| | Claude Code | GitHub Copilot |
|---|---|---|
| Skill entry | `.claude/skills/<name>/SKILL.md` | `.github/prompts/<name>.prompt.md` (`mode: agent`) |
| Subagents | Agent tool, parallel dispatch | subagent tool where available, else sequential in-session |
| Reviewer | `code-reviewer` agent | `code-reviewer` chat-mode instructions applied in-session |
| Triage | rules.md (auto-loaded) | rules.instructions.md (auto-loaded) |

### 4.9 User manual

`manual.md` (+ `manual.html`), generated per project, plain language, short:
- Day-to-day usage: just ask; how triage picks Direct / Quick / Feature and when it will ask you
- Commands: `/dev_feature`, `/dev_plan`, `/dev_execute`, `/dev_debug`, `/dev_tdd`, `/dev_mainseat`, `/dev_backseat`
- The chosen agents and what they're for
- Where things live in `.docs/` and how TASK folders work
- Greenfield: point to the seeded kickoff task

Lives in `.docs/` so the assistant can also read it on demand.

### 4.10 Docs and attribution

- README: describe the three tiers, the feature flow, new generated files, mandatory cleanup, the manual; update phase list and layouts; add Acknowledgements crediting superpowers (MIT, Jesse Vincent).
- Workflow templates carry the attribution header comment.

## 5. Out of scope (v1)

- Git worktrees and branch-finishing skills
- Automatic sync with upstream superpowers
- An HTML renderer script (the assistant renders)
- Copilot's `.github/skills/` format (prompts used, consistent with driver-mode)

## 6. Testing and rollout

**Rollout:** all work on `feature/workflow-skills`; commit; push to `origin`; test only after push. **Never run `/dev-chisel` inside the Chisel repo** — mandatory cleanup would delete its source. Tests use scratch copies in separate directories. `main` is the fallback until the user approves merge.

**Scenarios:**

| Scenario | Target | Checks |
|---|---|---|
| Empty repo | Claude | All files generated; `{{TODO}}` in architecture/manual; kickoff TASK seeded; cleanup runs; manual opens |
| Small existing project with tests | Claude | triage → Direct ("sort this list"), Quick (small bug), Feature proposal ("add auth") |
| Same project | Copilot | Prompts generated; feature flow runs end-to-end (sequential fallback OK) |
| Existing project without tests | either | `dev_tdd` present; invoking it proposes setting up tests first |
| Interrupted `dev_execute` | Claude | Re-run resumes at first unticked WI |

**Cross-cutting checks:** no leftover `{{…}}` placeholders except intentional `{{TODO: …}}` in greenfield docs (grep); HTML views render in light and dark mode; todo statuses, `todo_done.md` move, and `tasks/done/` move are correct.

## 7. Acceptance criteria

1. A fresh run on either target generates all five workflow skills, the `code-reviewer` agent plus the user-chosen agents, `.docs/tasks/`, and `manual.md`/`manual.html`, without depending on any plugin.
2. Generated rules contain the triage rule, the three tiers, and the folded principles; the existing Driver-Mode section remains.
3. A feature TASK can be taken from triage proposal through spec, plan, execution, and closure, with the todo entry and folder updated at each step.
4. Spec, plan, and manual HTML views are generated from Markdown and opened automatically (path printed on failure).
5. Phase 2 requires acknowledgement; cleanup always runs and leaves all generated workflow files intact.
6. The final phase opens the manual and gives a short summary.
7. README documents the new behavior and credits superpowers.

## 8. Open questions

None.

## 9. Amendment (2026-10-14) — `/dev_explore`

A sixth workflow command, always generated, written in the same style (not adapted from superpowers, which has no equivalent).

- **Invocation:** no argument → asks what to explore (key systems from `architecture.md`, a path, or a question); or takes a path/glob, feature name, TASK, or question directly.
- **Process:** read `.docs/` → resolve scope (one clarifying question only if ambiguous) → read code from entry points inward (Claude may delegate to `architect` or the built-in Explore agent).
- **Answer in chat:** Summary · How it works (`path:line`) · Key files · Connections · Observations · Docs drift. Follow-ups invited.
- **Optional report:** asked once at the end; on yes → `.docs/explore/<slug>.md` + `.html`, rendered and opened like specs and plans. Slug as TASK folders; collisions get the date, then `-2`.
- **Rules:** read-only in any driver-mode; only writes the report on yes; suggests (never adds) todo entries; code questions aren't tiered by triage.
