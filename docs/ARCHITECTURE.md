# atlas Architecture Reference

This document is the detailed architecture and contributor reference for the `atlas` plugin.

Use `README.md` for installation, first-run setup, bundled capabilities, and day-to-day usage. Use this document for the system model, routing rules, review loops, memory behavior, and implementation guarantees.

---

## System Model

The system uses a flat conductor-delegate pattern. Two user-facing agents handle interaction and planning. **atlas** directly manages execution by delegating to specialized workers.

```text
User
  |
  v
atlas (conductor) <--handoff--> prometheus (planner)
  |
  +---> ekko (backend)      -- writes server/logic code
  +---> aurora (frontend)   -- writes UI code
  +---> forge (infra)       -- writes CI/CD, containers, cloud, monitoring
  +---> sentry (reviewer)   -- reviews all changes (adversarial)
  +---> oracle (researcher) -- gathers context
  +---> killua (scout)      -- fast file discovery
  +---> metis (validator)   -- validates plans (dual-mode)
```

### User-Facing Agents

| Agent          | File                                 | Model                     | Role                                                                                                                                                                                                                       |
| -------------- | ------------------------------------ | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **atlas**      | `plugins/agents/atlas.agent.md`      | Claude Opus 4.6 (copilot) | Conductor. Routes tasks via IntentGate, delegates to workers directly, manages review loops, spot-checking, and todos. May apply trivial single-file quick fixes after review by **sentry**. Never writes multi-file code. |
| **prometheus** | `plugins/agents/prometheus.agent.md` | Claude Opus 4.6 (copilot) | Planner. Researches requirements, consults **metis** PRE_PLAN, drafts phased plans, validates iteratively with **metis** VALIDATE, then hands approved plans back to **atlas**. Never writes implementation code.          |

### Subagents

| Agent      | File                             | Model                       | Role                                                                                                      |
| ---------- | -------------------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------- |
| **ekko**   | `plugins/agents/ekko.agent.md`   | Claude Opus 4.6 (copilot)   | Backend/core implementer. Strict TDD, Write-Guard, Comment Discipline, Indistinguishable Code.            |
| **aurora** | `plugins/agents/aurora.agent.md` | GPT-5.4 (copilot)           | Frontend/UI implementer. TDD, accessibility-first, visual verification, stitch-mcp scaffolding.           |
| **forge**  | `plugins/agents/forge.agent.md`  | Claude Opus 4.6 (copilot)   | DevOps/infra implementer. CI/CD, containers, cloud, monitoring, deployment automation. Security-first.    |
| **sentry** | `plugins/agents/sentry.agent.md` | GPT-5.4 (copilot)           | Code reviewer. Adversarial analysis, security, correctness, and requirement validation. Never edits code. |
| **oracle** | `plugins/agents/oracle.agent.md` | Claude Sonnet 4.6 (copilot) | Researcher. Structured findings, convention discovery, external docs, skills recommendations.             |
| **killua** | `plugins/agents/killua.agent.md` | Claude Haiku 4.5 (copilot)  | Scout. Ultra-fast file discovery, dependency mapping, read-only exploration.                              |
| **metis**  | `plugins/agents/metis.agent.md`  | Claude Sonnet 4.6 (copilot) | Plan validator. PRE_PLAN for pre-planning analysis, VALIDATE for plan validation.                         |

### Delegation Rules

- **atlas** delegates to: **ekko**, **aurora**, **forge**, **sentry**, **oracle**, **killua**, **metis**
- **prometheus** delegates to: **oracle**, **killua**, **metis**
- Planned multi-file implementation is handled exclusively by **ekko**, **aurora**, and **forge**
- **sentry** reviews all changes in all modes
- Category routing is strict and enforced by **atlas**

---

## Core Philosophy

These principles govern every agent in the system. They are invariants, not soft guidelines.

- **Human intervention is a failure signal.** Resolve ambiguity, make decisions, and complete work without asking for avoidable user input.
- **Indistinguishable Code.** Output should read like senior-engineer work, matching project conventions without AI boilerplate.
- **Zero-trust.** No agent trusts its own work. Plans go through **metis** and code goes through **sentry**.
- **Minimize cognitive load.** Users provide intent; the system handles research, implementation, tests, review, and commit guidance.
- **Token cost is acceptable when it increases productivity.** Parallel search, redundant verification, and deeper research are all acceptable when they raise quality.

---

## IntentGate

Before routing a task, **atlas** evaluates intent.

1. **Weigh intent.** What is the user actually asking for?
2. **Research alternatives.** Do packages, built-ins, or existing repo patterns already solve it?
3. **Challenge assumptions.** Is the requested approach the best one?
4. **Present findings.** If materially different interpretations exist, use structured choices.
5. **Proceed.** Only after intent is clear does routing begin.

IntentGate exists to prevent the system from blindly implementing the first plausible interpretation.

---

## Planning and Validation

### Dual-Mode metis

**metis** operates in two modes, selected by the `MODE:` field in the delegation prompt.

#### PRE_PLAN Mode

Used by **prometheus** before drafting a plan. It surfaces:

- Hidden intentions
- Ambiguities
- AI failure points
- Missing context
- Scope assessment
- Package alternatives

#### VALIDATE Mode

Used by both **prometheus** and **atlas** to validate plans. It checks:

- Feasibility and scope
- File and function references
- Logical phase ordering
- Test strategy completeness
- Quality gate coverage
- Alignment with Atlas operating rules

If validation fails, the plan enters a revision loop. If it fails repeatedly, the issue is surfaced back to the user through structured choices.

---

## Task Routing

### Strict Category Routing

| Category                     | Worker Routing                                  |
| ---------------------------- | ----------------------------------------------- |
| `visual/UI/frontend/styling` | **aurora** only                                 |
| `backend/API/database/logic` | **ekko** only                                   |
| `infra/devops/deployment`    | **forge** only                                  |
| `full-stack/mixed`           | Sequential: **ekko** first, then **aurora**     |
| `architecture/design`        | **oracle** for analysis, then route to a worker |
| `documentation/writing`      | **atlas** may handle agent/system docs directly |

If the category is not explicit, **atlas** infers from file types and task wording.

---

## Worker Guarantees

All implementation workers (**ekko**, **aurora**, **forge**) follow the same baseline guarantees:

- **Write-Guard.** Never edit a file without reading it first.
- **Comment Discipline.** Comments must add value; avoid narrating obvious code.
- **Indistinguishable Code.** Match repository conventions and avoid AI-shaped code patterns.
- **No Scope Creep.** Do exactly what the task requires. No bonus refactors or opportunistic rewrites.

---

## Completion and Review

### Completion Verification

Before marking a phase complete, **atlas** verifies:

- Every planned file was changed as expected
- Every planned test was written or updated
- Quality gates actually ran
- No `TODO`, `FIXME`, or `HACK` comments remain in modified files

### Atlas Phase Implementation Loop

In Autopilot mode, **atlas** runs a bounded review loop:

1. Delegate phase work to the appropriate worker
2. Send the result to **sentry** for review
3. If **sentry** finds issues, re-delegate to the same worker with targeted feedback
4. Repeat until approved or the iteration limit is reached

### Adversarial Review by sentry

**sentry** reviews all changes in all modes. In addition to correctness, it looks for:

- Wrong assumptions
- Missing edge cases
- Security gaps
- False claims in worker reports
- Situations where “zero findings” likely means the review was too shallow

**sentry** can also suggest new hooks when it sees repeated quality issues.

---

## Hooks Details

Hooks are shipped via `hooks/hooks.json` and cover the agent lifecycle.

### Hook Scripts

| Hook            | Lifecycle Event  | Script               | Purpose                                                  |
| --------------- | ---------------- | -------------------- | -------------------------------------------------------- |
| session-start   | SessionStart     | `session-start.sh`   | Injects repo conventions, branch, and project metadata   |
| prompt-submit   | UserPromptSubmit | `prompt-submit.sh`   | Detects Autopilot keywords and anti-patterns             |
| write-guard     | PreToolUse       | `write-guard.sh`     | Warns when editing files not read in the current session |
| comment-checker | PostToolUse      | `comment-checker.sh` | Flags high comment density                               |
| pre-compact     | PreCompact       | `pre-compact.sh`     | Captures session state before context compaction         |
| subagent-start  | SubagentStart    | `subagent-start.sh`  | Injects role-specific rules into subagents               |
| session-stop    | Stop             | `session-stop.sh`    | Warns on uncommitted changes and temp artifacts          |

### Scope and Portability

- PreToolUse/PostToolUse hooks fire for the active agent's own tool calls
- Subagent edits are not covered the same way; workers enforce these rules proactively
- On plugin install, relative paths resolve automatically from `hooks/hooks.json`
- For per-project use, copy the hook config into `.github/hooks/quality.json` and copy `scripts/hooks/` into the target repo

### Creating New Hooks

Use `/create-hook` to add project-specific lifecycle checks. Hook scripts in `scripts/hooks/` follow a consistent `set -euo pipefail` shell pattern with jq-based parsing and graceful degradation if jq is unavailable.

---

## Skills and Integrations

### Skills Mechanics

Skills live under `plugins/skills/` and are auto-discovered by the plugin system.

To package a reusable workflow:

- Use `/create-skill`
- Let **oracle** recommend new skills when reusable patterns appear during research
- Let workers note skill-worthy patterns in their deviation reports

### Integrations and Provenance

#### impeccable

The bundled `frontend-design` skill is adapted from [impeccable](https://github.com/pbakaus/impeccable) (Apache 2.0). It provides design guidance and the reference material used by **aurora** and **sentry** during UI work.

The design skill family adapted from impeccable includes:

- `/design-audit`
- `/design-polish`
- `/design-normalize`
- `/design-harden`
- `/design-critique`
- `/design-clarify`
- `/design-adapt`
- `/design-optimize`
- `/design-animate`
- `/design-extract`
- `/design-onboard`
- `/design-colorize`
- `/design-bolder`
- `/design-quieter`

#### forge origin

**forge** is derived from the [agency-agents](https://github.com/msitarzewski/agency-agents) DevOps Automator pattern and handles CI/CD, containerization, cloud infrastructure, monitoring, and deployment automation.

#### Ecosystem skills

Additional skills are inspired by established open-source methodologies:

- **`/security-review`** -- inspired by [getsentry/skills](https://github.com/getsentry/skills) (Apache 2.0)
- **`/terraform-patterns`** -- inspired by [antonbabenko/terraform-best-practices](https://github.com/antonbabenko/terraform-best-practices) (Apache 2.0)
- **`/postgres-patterns`** -- inspired by [supabase/agent-skills](https://github.com/supabase/agent-skills) (MIT)

---

## MCP and Research Workflow

### Research Tool Priority

Agents with research capabilities follow this priority order:

1. **`context7/*`** -- primary documentation
2. **`search`** -- local repo context
3. **`exa/*`** and **`tavily/*`** -- external search and research
4. **`web`** -- fallback crawler
5. **killua** -- fast file discovery
6. **oracle** -- deeper analysis

### MCP Server Details

**context7** -- up-to-date library documentation for LLMs. The bundled plugin config does not require or prompt for an API key.

**sequential-thinking** -- structured multi-step reasoning scaffold used by reasoning-heavy agents for architectural decisions and multi-constraint analysis.

**exa** -- AI-native semantic search with bundled web search, advanced search, code context, crawl, company research, people search, and deep research tools.

**tavily** -- AI-optimized search and research suited to synthesized answers and current-web context.

**stitch-mcp** -- Google Stitch for rapid UI scaffolding, component generation, and layout templates. **aurora** treats this as a scaffold, not final output.

### sequential-thinking Supported-Agent Matrix

| Agent          | Included | Rationale                                                        |
| -------------- | -------- | ---------------------------------------------------------------- |
| **atlas**      | Yes      | Planning and architectural routing benefit from structured steps |
| **prometheus** | Yes      | Multi-phase planning needs explicit reasoning scaffolds          |
| **metis**      | Yes      | Validation is a multi-constraint task                            |
| **oracle**     | Yes      | Research often requires tradeoff analysis                        |
| **sentry**     | Yes      | Adversarial review benefits from systematic reasoning            |
| **ekko**       | Yes      | Backend architecture and logic often require deeper reasoning    |
| **aurora**     | No       | UI work is convention-matching rather than reasoning-heavy       |
| **killua**     | No       | Speed-first scouting would be slowed down by reasoning overhead  |

Hook scripts are intentionally unchanged for this rollout. Sequential thinking is opt-in per agent file.

---

## Browser and Terminal Discipline

### Dev Server Management

- Detect existing dev servers before launching new ones
- Clean up background processes before returning reports
- Never kill pre-existing servers

### Built-in Browser Tools

When `workbench.browser.enableChatTools` is enabled, agents use built-in browser tools for verification:

- `runPlaywrightCode`
- `clickElement`
- `readPage`
- `screenshotPage`
- `openBrowserPage` / `navigatePage`

**aurora** keeps `stitch-mcp/*` as a separate concern for scaffolding, while browser tools are used for verification.

### Optional Review Enhancements

**sentry** can optionally use:

- **CodeRabbit** (`coderabbit review --plain`) for supplemental findings
- Browser verification for UI phases to check console errors and visible regressions

---

## Memory System

### Session Memory

- **atlas** writes `<task>-atlas.md` for orchestration state recovery after context compaction
- Subagents may use private scratchpad files during execution
- If a scratchpad has no transfer value, the subagent deletes it before returning
- If it has relevant context, **atlas** reads it, extracts what it needs, and deletes it immediately
- **metis** and **sentry** session files can persist until their review loops complete

### Repository Memory

Agents can write `.json` files under repository memory for:

- discovered conventions
- verified commands
- architecture decisions
- anti-patterns worth retaining across sessions

---

## Universal Tooling Detection

Agents auto-detect the tooling stack in this order:

1. `AGENTS.md` `tooling:` block
2. `package.json` scripts
3. config files like `vitest.config.*` and `eslint.config.*`
4. lockfile detection (`bun.lock`, `yarn.lock`, `package-lock.json`)
5. fallback defaults for greenfield work
