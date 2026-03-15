# **atlas**

A multi-agent orchestration system for VS Code Copilot that drives the complete software development lifecycle -- **Planning -> Implementation -> Review -> Commit** -- through intelligent agent delegation and parallel execution.

> Built upon the foundation of [copilot-orchestra](https://github.com/ShepAlderson/copilot-orchestra) by ShepAlderson.
> Aligned with [Oh My OpenAgent (OmO)](https://github.com/nicholascpark/oh-my-open-agent) v3.11.x principles.

---

## Core Philosophy

These principles govern every agent in the system. They are not guidelines -- they are invariants.

- **Human intervention is a failure signal.** The system should resolve ambiguity, make decisions, and complete work without asking the user for help. Every question asked is a failure to infer. Every approval sought is a failure to verify. Minimize interaction rounds -- needing to ask means the system failed.

- **Indistinguishable Code.** Output must be indistinguishable from a senior engineer's work. Follow existing project conventions exactly. Use proper error handling without being asked. No over-engineering, no unnecessary abstractions, no AI-generated commentary that restates what code obviously does.

- **Zero-trust.** No agent trusts its own work. Every change is reviewed by **sentry**. Every plan is validated by **metis**. Completion claims are verified against the plan. Zero findings after review means look harder.

- **Minimize cognitive load.** Users provide intent; agents provide everything else -- research, alternatives, implementation, tests, review, and commit messages. When user input is needed, present structured choices via `vscode/askQuestions`, not open-ended questions.

- **Token cost is acceptable when it increases productivity.** Parallel searches, redundant verification, deep research -- all are justified when they produce better outcomes. Speed and cost matter less than correctness and completeness.

---

## Architecture

The system uses a flat conductor-delegate pattern. Two user-facing agents handle interaction and planning. **atlas** directly manages execution by delegating to specialized workers. Specialized subagents perform the actual work.

```
User
  |
  v
atlas (conductor) <--handoff--> prometheus (planner)
  |
  +---> ekko (backend)     -- writes server/logic code
  +---> aurora (frontend)   -- writes UI code
  +---> forge (infra)       -- writes CI/CD, containers, cloud, monitoring
  +---> sentry (reviewer)   -- reviews all changes (adversarial)
  +---> oracle (researcher) -- gathers context
  +---> killua (scout)      -- fast file discovery
  +---> metis (validator)   -- validates plans (dual-mode)
```

### User-Facing Agents

| Agent          | File                  | Model                     | Role                                                                                                                                                                                                                                       |
| -------------- | --------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **atlas**      | `atlas.agent.md`      | Claude Opus 4.6 (copilot) | Conductor. Routes tasks via IntentGate, delegates to workers directly, manages review loops, spot-checking, and todos. Presents results. May apply trivial single-file quick fixes (reviewed by **sentry**). Never writes multi-file code. |
| **prometheus** | `prometheus.agent.md` | Claude Opus 4.6 (copilot) | Planner. Researches requirements, consults **metis** PRE_PLAN, drafts phased plans, validates iteratively with **metis** VALIDATE, hands off to **atlas**. Never writes implementation code.                                               |

**atlas** and prometheus hand off to each other via VS Code's agent handoff system. **atlas** hands complex tasks to prometheus for planning; prometheus hands approved plans back to **atlas** for execution.

### Subagents

| Agent      | File              | Model                       | Role                                                                                                                       |
| ---------- | ----------------- | --------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **ekko**   | `ekko.agent.md`   | Claude Opus 4.6 (copilot)   | Backend/core implementer. Strict TDD, Write-Guard, Comment Discipline, Indistinguishable Code.                             |
| **aurora** | `aurora.agent.md` | GPT-5.4 (copilot)           | Frontend/UI implementer. TDD, accessibility-first, visual verification, stitch-mcp scaffolding.                            |
| **forge**  | `forge.agent.md`  | Claude Opus 4.6 (copilot)   | DevOps/infra implementer. CI/CD, containers, cloud, monitoring, deployment automation. Security-first.                     |
| **sentry** | `sentry.agent.md` | GPT-5.4 (copilot)           | Code reviewer. Adversarial analysis, security, correctness, requirement validation. Optional CodeRabbit. Never edits code. |
| **oracle** | `oracle.agent.md` | Claude Sonnet 4.6 (copilot) | Researcher. Structured findings, convention discovery, external docs, skills recommendations.                              |
| **killua** | `killua.agent.md` | Claude Haiku 4.5 (copilot)  | Scout. Ultra-fast file discovery, dependency mapping, read-only exploration.                                               |
| **metis**  | `metis.agent.md`  | Claude Sonnet 4.6 (copilot) | Plan validator (dual-mode). PRE_PLAN for pre-planning analysis, VALIDATE for plan validation. Default: VALIDATE.           |

### Delegation Rules

- **atlas** delegates to: **ekko**, **aurora**, **forge**, **sentry**, **oracle**, **killua**, **metis**
- **prometheus** delegates to: **oracle**, **killua**, **metis** (PRE_PLAN + VALIDATE)
- Planned multi-file implementation is handled exclusively by **ekko**, **aurora**, and **forge**. **atlas** may apply trivial single-file quick fixes directly (always reviewed by **sentry**).
- **sentry** reviews ALL changes in ALL modes -- never skipped.
- Category routing is STRICT. **atlas** enforces routing rules directly based on the task category.

---

## IntentGate

Before routing any task, **atlas** runs the IntentGate:

1. **Weigh intent.** What is the user actually asking for? Is the request clear, or does it contain hidden assumptions?
2. **Research alternatives.** Do established packages, libraries, or VS Code extensions already solve this problem?
3. **Challenge assumptions.** Is the user's proposed approach the best one, or is there a simpler/more maintainable alternative?
4. **Present findings.** If alternatives exist, present them via `vscode/askQuestions` with structured choices. Let the user decide.
5. **Proceed.** Only after intent is validated does the task enter the routing pipeline.

IntentGate prevents the system from blindly implementing whatever is asked. It acts as a forcing function for thoughtful engineering.

---

## Dual-Mode **metis**

**metis** operates in two modes, determined by the `MODE:` field in the delegation prompt:

### PRE_PLAN Mode

Used by prometheus before drafting a plan. Analyzes the user's request and interview results to surface:

- **Hidden Intentions** -- what the user didn't think of
- **Ambiguities** -- what could cause implementation failure
- **AI Failure Points** -- where agents are likely to hallucinate
- **Missing Context** -- what should be researched before planning
- **Scope Assessment** -- is this too big for one plan?
- **Package Alternatives** -- existing solutions worth considering

### VALIDATE Mode (Default)

Used by both prometheus and **atlas** to validate plans. Runs a checklist covering:

- Feasibility and scope assessment
- File and function references (do they exist?)
- Logical phase ordering and dependency chains
- Test strategy completeness
- Quality gate coverage
- **OmO Alignment** -- hooks/skills recommendations, package alternatives, Indistinguishable Code, minimal human interaction

If validation fails, the plan enters a revision loop (max 3 cycles). If it fails 3 times, issues are presented to the user via `vscode/askQuestions`.

---

## Strict Category Routing

**atlas** enforces strict routing based on the task category:

| Category                     | Worker Routing                                                                                                                                  |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `visual/UI/frontend/styling` | **aurora** ONLY.\*\* **ekko** excluded unless backend API work is also needed.                                                                  |
| `backend/API/database/logic` | **ekko** ONLY.\*\* **aurora** excluded unless frontend integration is also needed.                                                              |
| `infra/devops/deployment`    | **forge** ONLY. CI/CD, containers, cloud infrastructure, monitoring, deployment automation.                                                     |
| `full-stack/mixed`           | **Sequential: **ekko** first, then **aurora**.** If **ekko** passes review but **aurora** fails, re-run only **aurora**. Non-overlapping files. |
| `architecture/design`        | **oracle** for analysis, then route to appropriate worker.                                                                                      |
| `documentation/writing`      | **ekko** for backend docs, **aurora** for frontend docs.                                                                                        |

If no category is provided, **atlas** infers from file types (`.tsx`/`.css` -> **aurora**, `.ts`/`.py` server files -> **ekko**, `Dockerfile`/`.yml` CI configs/`.tf`/Helm -> **forge**, mixed -> both sequential).

---

## Worker Zero-Trust Hardening

All implementation workers (**ekko**, **aurora**, **forge**) follow these rules:

- **Write-Guard.** Never edit a file without reading it first. In the prompts workspace, workspace hooks enforce this (PreToolUse). In other workspaces, workers must follow proactively.
- **Comment Discipline.** Comments must add value. No restating what code obviously does. In the prompts workspace, workspace hooks flag files exceeding 30% comment density (PostToolUse). Workers must avoid AI slop proactively.
- **Indistinguishable Code.** Code must look like a senior engineer wrote it. Follow existing project conventions. No AI-generated boilerplate.
- **No Scope Creep.** Do exactly what was asked. No bonus features, no "while I'm here" refactors, no unnecessary abstractions.

---

## Completion Enforcement

### Completion Verification (**atlas** Phase Verification)

Before marking a phase as complete, **atlas** verifies:

- Every file listed in the plan was actually modified
- Every test specified in the plan was actually written
- Quality gates were actually run
- No `TODO`, `FIXME`, or `HACK` comments were left in modified files

### **atlas** Phase Implementation Loop

In Autopilot mode, **atlas** runs a phase implementation loop (max 5 iterations):

1. Delegates phase work to the appropriate worker(s)
2. **sentry** reviews the worker's output
3. If **sentry** finds issues, **atlas** re-delegates to the worker with **sentry**'s findings
4. Loop continues until **sentry** approves or max 5 iterations are reached
5. Only marks phase COMPLETE when **sentry** approves

This prevents the "declare victory early" failure mode common in autonomous agents.

### Completion Honesty

**atlas** will NEVER report COMPLETE if tests are failing or quality gates were skipped. It will report BLOCKED instead.

---

## Adversarial Review (**sentry**)

**sentry** reviews ALL changes in ALL modes -- normal and Autopilot. It is never skipped.

Beyond standard code review (correctness, security, quality), **sentry** performs adversarial analysis:

- **Assumptions Challenged.** For every major decision, identify assumptions relied on. What breaks if they're wrong?
- **Failure Modes.** Scenarios that could cause the code to fail in production. Edge cases not covered by tests.
- **Zero findings = look harder.** Absence of findings is a red flag, not a sign of perfection. Every review must surface at least observations.
- **False Claims.** Worker claims are verified against actual code. Discrepancies are flagged as MAJOR issues.

**sentry** can also recommend creating hooks (`/create-hook`) when it identifies recurring quality issues.

---

## Hooks System

Hooks provide automated quality enforcement at key lifecycle events. Hooks are shipped via `.github/hooks/quality.json` inside the plugin bundle and fire for all agents in the workspace.

### Hook Scripts

| Hook            | Lifecycle Event  | Script             | Purpose                                                        |
| --------------- | ---------------- | ------------------ | -------------------------------------------------------------- |
| session-start   | SessionStart     | session-start.sh   | Injects repo conventions, git branch, project metadata         |
| prompt-submit   | UserPromptSubmit | prompt-submit.sh   | Detects Autopilot keywords (ULW/YOLO), quality anti-patterns   |
| write-guard     | PreToolUse       | write-guard.sh     | Warns when editing files not read in the current session       |
| comment-checker | PostToolUse      | comment-checker.sh | Flags files exceeding 30% comment density                      |
| pre-compact     | PreCompact       | pre-compact.sh     | Captures session state snapshot before context compaction      |
| subagent-start  | SubagentStart    | subagent-start.sh  | Injects Core Philosophy and role-specific rules into subagents |
| session-stop    | Stop             | session-stop.sh    | Warns on uncommitted changes and temp artifacts                |

**Scope:** PreToolUse/PostToolUse hooks fire for the active agent's own tool calls. Subagent (**ekko**/**aurora**/**forge**) edits are not covered by hooks -- workers enforce Write-Guard and Comment Discipline proactively.

### Portability

- **Plugin install** (`copilot plugin install`): Hooks load from `.github/hooks/quality.json` inside the plugin directory. Relative paths resolve automatically. No additional setup needed.
- **Per-project**: Copy `.github/hooks/quality.json` and `scripts/hooks/` into any repo for project-level hook coverage.

### Creating New Hooks

Use the `/create-hook` VS Code command to create project-specific hooks. **sentry** recommends hooks when it identifies recurring quality issues across reviews.

Hook scripts live in `scripts/hooks/` and follow consistent patterns: `set -euo pipefail`, jq-based JSON parsing, graceful degradation if jq is missing.

---

## Skills & Plugins

### Plugin Distribution

This repo is packaged as a Copilot plugin via `plugin.json`. Install with:

```bash
copilot plugin install iamvikshan/atlas
```

The plugin bundles agents and hooks into a single installable unit. Skills are auto-discovered from the `.github/skills/` directory as they are added. Works in VS Code (local, Codespaces, and SSH remote).

### Skills

Reusable workflow patterns placed in `.github/skills/` and auto-discovered by the plugin system.

**Shipped skills:**

| Skill                  | Directory                            | Description                                                                                                                                                                                                                                                                                                             |
| ---------------------- | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **frontend-design**    | `.github/skills/frontend-design/`    | Design quality enforcement for UI work. Guides **aurora** and **sentry** to produce distinctive, production-grade interfaces. Includes 7 deep-dive reference docs (typography, color, spatial, motion, interaction, responsive, UX writing). Based on [impeccable](https://github.com/pbakaus/impeccable) (Apache 2.0). |
| **design-audit**       | `.github/skills/design-audit/`       | Systematic UI quality audit across accessibility, performance, theming, responsive design, and AI slop detection. Generates severity-rated reports with fix recommendations. Based on impeccable (Apache 2.0).                                                                                                          |
| **design-polish**      | `.github/skills/design-polish/`      | Final quality pass before shipping. Systematically fixes alignment, spacing, consistency, interaction states, and micro-details across 12 dimensions. Based on impeccable (Apache 2.0).                                                                                                                                 |
| **design-normalize**   | `.github/skills/design-normalize/`   | Normalize design to match the project's design system. Replaces one-off implementations with system equivalents, enforces consistent tokens and patterns. Based on impeccable (Apache 2.0).                                                                                                                             |
| **design-harden**      | `.github/skills/design-harden/`      | Harden interfaces against edge cases, i18n issues, error handling, text overflow, and real-world usage. Makes UIs production-ready. Based on impeccable (Apache 2.0).                                                                                                                                                   |
| **design-critique**    | `.github/skills/design-critique/`    | Evaluate design effectiveness from a UX perspective. Assesses visual hierarchy, information architecture, emotional resonance, and overall design quality. Based on impeccable (Apache 2.0).                                                                                                                            |
| **design-clarify**     | `.github/skills/design-clarify/`     | Improve unclear UX copy, error messages, microcopy, labels, and instructions. Makes interfaces easier to understand and use. Based on impeccable (Apache 2.0).                                                                                                                                                          |
| **design-adapt**       | `.github/skills/design-adapt/`       | Adapt designs to work across different screen sizes, devices, contexts, or platforms. Ensures consistent experience across varied environments. Based on impeccable (Apache 2.0).                                                                                                                                       |
| **design-optimize**    | `.github/skills/design-optimize/`    | Improve interface performance across loading speed, rendering, animations, images, and bundle size. Makes experiences faster and smoother. Based on impeccable (Apache 2.0).                                                                                                                                            |
| **design-animate**     | `.github/skills/design-animate/`     | Enhance features with purposeful animations, micro-interactions, and motion effects that improve usability and delight. Based on impeccable (Apache 2.0).                                                                                                                                                               |
| **design-extract**     | `.github/skills/design-extract/`     | Extract and consolidate reusable components, design tokens, and patterns into your design system. Identifies opportunities for systematic reuse. Based on impeccable (Apache 2.0).                                                                                                                                      |
| **design-onboard**     | `.github/skills/design-onboard/`     | Design or improve onboarding flows, empty states, and first-time user experiences. Helps users get started successfully and understand value. Based on impeccable (Apache 2.0).                                                                                                                                         |
| **design-colorize**    | `.github/skills/design-colorize/`    | Add strategic color to features that are too monochromatic or lack visual interest. Makes interfaces more engaging and expressive. Based on impeccable (Apache 2.0).                                                                                                                                                    |
| **design-bolder**      | `.github/skills/design-bolder/`      | Amplify safe or boring designs to make them more visually interesting and stimulating. Increases impact while maintaining usability. Based on impeccable (Apache 2.0).                                                                                                                                                  |
| **design-quieter**     | `.github/skills/design-quieter/`     | Tone down overly bold or visually aggressive designs. Reduces intensity while maintaining design quality and impact. Based on impeccable (Apache 2.0).                                                                                                                                                                  |
| **security-review**    | `.github/skills/security-review/`    | Systematic, OWASP-informed security code review that identifies exploitable vulnerabilities with confidence-based reporting. Inspired by [getsentry/skills](https://github.com/getsentry/skills) (Apache 2.0).                                                                                                          |
| **terraform-patterns** | `.github/skills/terraform-patterns/` | Terraform best practices for file structure, naming, variables, outputs, version pinning, remote state, secrets, and module composition. Inspired by [antonbabenko/terraform-best-practices](https://github.com/antonbabenko/terraform-best-practices) (Apache 2.0).                                                    |
| **postgres-patterns**  | `.github/skills/postgres-patterns/`  | PostgreSQL best practices for query performance, connection management, security, schema design, concurrency, data access patterns, and monitoring. Inspired by [supabase/agent-skills](https://github.com/supabase/agent-skills) (MIT).                                                                                |
| **github-triage**      | `.github/skills/github-triage/`      | Read-only GitHub issue/PR triage. Classifies items, performs evidence-backed analysis with permalink citations, and produces structured reports. Zero mutations. Inspired by [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent).                                                                        |

**Creating skills:**

- Use `/create-skill` to package a reusable workflow (e.g., TDD setup, migration checklist, deployment procedure)
- **oracle** recommends skills when it discovers reusable patterns during research
- Workers note skill-worthy patterns in their deviation reports

---

## Integrations

### impeccable (Design Quality)

The bundled `frontend-design` skill (in `.github/skills/frontend-design/`) is adapted from [impeccable](https://github.com/pbakaus/impeccable) (Apache 2.0). It provides design guidance, aesthetic rules, and 7 reference documents that **aurora** and **sentry** use during UI work.

- **aurora** applies the skill's design direction and aesthetic guidelines when building UI components
- **sentry** checks for common AI-generated design anti-patterns (Inter font defaults, purple gradients, excessive border-radius, etc.) during UI phase reviews

In addition to the core design knowledge, atlas bundles 14 action-oriented design skills adapted from impeccable's skill commands:

- `/design-audit` -- Comprehensive UI quality audit with severity-rated report
- `/design-polish` -- Final quality pass across 12 polish dimensions
- `/design-normalize` -- Align feature with project's design system
- `/design-harden` -- Harden UI against edge cases, i18n, and error scenarios
- `/design-critique` -- Evaluate design effectiveness from a UX perspective
- `/design-clarify` -- Improve unclear UX copy, error messages, and microcopy
- `/design-adapt` -- Adapt designs across screen sizes, devices, and platforms
- `/design-optimize` -- Improve interface performance (loading, rendering, bundle size)
- `/design-animate` -- Enhance features with purposeful animations and micro-interactions
- `/design-extract` -- Extract reusable components and design tokens into the design system
- `/design-onboard` -- Design or improve onboarding flows and first-time experiences
- `/design-colorize` -- Add strategic color to monochromatic or flat interfaces
- `/design-bolder` -- Amplify safe designs to be more visually interesting
- `/design-quieter` -- Tone down overly bold or visually aggressive designs

These are invocable as slash commands in Copilot Chat and auto-load when relevant. No separate impeccable install needed.

### forge (DevOps/Infrastructure Agent)

New worker agent derived from the [agency-agents](https://github.com/msitarzewski/agency-agents) DevOps Automator pattern. Handles CI/CD pipelines, containerization, cloud infrastructure, monitoring, and deployment automation.

- Routes via `infra/devops/deployment` category
- Security-first: secrets management, pinned versions, container scanning
- Reviewed by **sentry** with infrastructure-specific security patterns

### Ecosystem Skills

Three additional skills authored from established open-source methodologies:

- **`/security-review`** -- OWASP-informed security code review with confidence-based reporting. Used by **sentry** during all phase reviews. Inspired by [getsentry/skills](https://github.com/getsentry/skills) (Apache 2.0).
- **`/terraform-patterns`** -- Terraform best practices for file structure, naming, state, secrets, and module composition. Used by **forge** during IaC tasks. Inspired by [antonbabenko/terraform-best-practices](https://github.com/antonbabenko/terraform-best-practices) (Apache 2.0).
- **`/postgres-patterns`** -- PostgreSQL best practices for query performance, security, schema design, and monitoring. Used by **ekko** during database tasks. Inspired by [supabase/agent-skills](https://github.com/supabase/agent-skills) (MIT).

---

## Modes

### Normal Mode (Default)

After each phase passes **sentry** review, **atlas** presents the phase summary and commit message via `vscode/askQuestions` carousel. The user can accept (**atlas** commits), pause (review first), or revise (submit feedback). All phases stay within a single chat interaction.

### Autopilot Mode

Triggered by explicit chat keywords only: `ULW`, `YOLO`. VS Code's `/yolo`, `/autoApprove`, and Autopilot permission level do NOT trigger this mode. Fully autonomous:

- Auto-commits after **sentry** approval
- No user stops between phases
- Phase Implementation Loop enforces **sentry** review on every phase (max 5 iterations)
- Uses `task_complete` tool to signal completion

---

## Research Tool Priority

All agents with research capabilities follow this priority order:

1. **`context7/*`** -- Primary Documentation. Fastest/most authoritative for library APIs.
2. **`search`** -- Local Context. Find internal patterns and conventions.
3. **`exa/*` and `tavily/*`** -- Reliable Web Search. External troubleshooting/comparison.
4. **`web`** -- Fallback Crawler. Use only if 1-3 fail.
5. **killua** -- File Discovery. "Where is X?"
6. **oracle** -- Deep Analysis. "How does X work?"

---

## Reasoning Scaffolds

**Prerequisite:** The `sequential-thinking` MCP server must be registered.

```json
		"sequential-thinking": {
			"url": "https://remote.mcpservers.org/sequentialthinking/mcp",
			"type": "http"
		}
```

### Supported-Agent Matrix

| Agent          | Included | Rationale                                                                    |
| -------------- | -------- | ---------------------------------------------------------------------------- |
| **atlas**      | Yes      | Planning and architectural routing benefit from structured steps             |
| **prometheus** | Yes      | Multi-phase plan design requires multi-step reasoning                        |
| **metis**      | Yes      | Multi-constraint validation and analysis                                     |
| **oracle**     | Yes      | Deep research involves multi-constraint evaluation                           |
| **sentry**     | Yes      | Adversarial review requires systematic reasoning                             |
| **ekko**       | Yes      | Backend logic involves reasoning-intensive architecture                      |
| **aurora**     | No       | GPT-5.4 model; UI work is convention-matching, not reasoning                 |
| **killua**     | No       | Claude Haiku 4.5 speed-first design; reasoning overhead contradicts the role |

**Hook scripts are intentionally unchanged for this rollout.** Sequential thinking is opt-in per agent file; no hook modifications required.

---

## Browser & Terminal Discipline

Agents that interact with terminals and browsers follow strict lifecycle rules:

### Dev Server Management

- Dev servers are detected before launching (no duplicate servers)
- Background processes are cleaned up before returning reports
- Pre-existing servers are never killed

### Built-in Browser Tools

When `workbench.browser.enableChatTools` is enabled (VS Code v1.109+), agents use built-in browser tools for verification:

- `runPlaywrightCode` -- Run custom Playwright scripts
- `clickElement` -- Click UI elements
- `readPage` -- Read page content and DOM
- `screenshotPage` -- Capture visual state
- `openBrowserPage` / `navigatePage` -- Open/navigate to URLs

**aurora** retains `stitch-mcp/*` separately for UI scaffolding (component generation, design tokens). Scaffolding and browser verification are separate concerns.

### Optional Review Enhancements

**sentry** supports optional background tooling:

- **CodeRabbit** (`coderabbit review --plain`) -- launched in background if available, findings supplement manual review
- **Browser verification** -- for UI phases, **sentry** loads the app and checks for console errors and visual regressions

---

## Memory System

### Session Memory (`/memories/session/`)

- **atlas** writes `<task>-atlas.md` for orchestration state recovery after context compaction
- Subagents use private scratchpad files during execution. If the scratchpad has no transfer value, the subagent deletes it before returning. If it has relevant context, **atlas** reads it, extracts what it needs, and deletes it immediately.
- Exception: **metis** and **sentry** session files persist until their respective review loops complete, then **atlas** deletes them.
- **atlas**'s own `<task>-atlas.md` is deleted last, during the final archive/completion flow.

### Repository Memory (`/memories/repo/`)

- Agents write `.json` files for discovered conventions, verified commands, and architecture decisions
- Format: `{"subject": "...", "fact": "...", "citations": [...], "reason": "...", "category": "...", "last_updated": "<time>", "by": "<agent>"}`
- Categories: `convention`, `tooling`, `architecture`, `anti-pattern`
- Persists across conversations within the same repository

---

## Universal Tooling Detection

Agents auto-detect the project's tooling stack:

1. `AGENTS.md` `tooling:` block (highest priority)
2. `package.json` scripts
3. Config files (`vitest.config.*`, `eslint.config.*`, etc.)
4. Lockfile detection (`bun.lock` -> Bun, `yarn.lock` -> Yarn, `package-lock.json` -> npm)
5. Fallback for greenfield projects: Bun + TypeScript + ESLint + Prettier

---

## Installation

### Option A: Plugin Install (recommended, portable)

Install as a Copilot plugin from the GitHub repo. Works in any environment (local, Codespaces, SSH remote):

```bash
copilot plugin install iamvikshan/atlas
```

The plugin system handles path resolution for agents, hooks, and skills automatically. No additional setup required.

Alternatively, in VS Code Insiders: open Copilot Chat and use **Install Plugin from Source** pointing to the repo.

### Option B: Global Prompts Directory (manual)

1. Clone or copy to your VS Code prompts directory:
   - **macOS**: `~/Library/Application Support/Code - Insiders/User/prompts/`
   - **Linux**: `~/.config/Code - Insiders/User/prompts/`
   - Or configure via `chat.promptFilesLocations`

This path provides the agents and prompt files, but it does **not** bundle hook wiring automatically. For hook coverage, either install Atlas as a plugin or add `.github/hooks/quality.json` plus `scripts/hooks/` to the target workspace.

### Post-Install (both options)

Enable required VS Code settings:

```json
{
  "chat.useCustomizationsInParentRepositories": true
}
```

Optional enhancements:

```json
{ "workbench.browser.enableChatTools": true }
```

```bash
# CodeRabbit CLI (enhanced reviews)
curl -fsSL https://cli.coderabbit.ai/install.sh | sh
```

Start a conversation with `@atlas` in VS Code Copilot Chat.

---

## File Structure

```
prompts/
  plugin.json                  -- Copilot plugin manifest (portable install)
  atlas.agent.md              -- Conductor (user-facing)
  prometheus.agent.md          -- Planner (user-facing)
  ekko.agent.md                -- Backend implementer
  aurora.agent.md              -- Frontend implementer
  forge.agent.md               -- DevOps/infra implementer
  sentry.agent.md              -- Code reviewer
  oracle.agent.md              -- Researcher
  killua.agent.md              -- Scout
  metis.agent.md               -- Plan validator (dual-mode)
  README.md                    -- This file
  .github/skills/
    frontend-design/
      SKILL.md                 -- Design quality skill (from impeccable, Apache 2.0)
      NOTICE.md                -- Attribution notice
      reference/               -- 7 deep-dive reference docs
        typography.md
        color-and-contrast.md
        spatial-design.md
        motion-design.md
        interaction-design.md
        responsive-design.md
        ux-writing.md
    design-audit/
      SKILL.md                 -- UI quality audit skill (from impeccable, Apache 2.0)
      NOTICE.md                -- Attribution notice
    design-polish/
      SKILL.md                 -- Final polish pass skill (from impeccable, Apache 2.0)
      NOTICE.md                -- Attribution notice
    design-normalize/
      SKILL.md                 -- Design system normalization skill (from impeccable, Apache 2.0)
      NOTICE.md                -- Attribution notice
    design-harden/
      SKILL.md                 -- UI hardening skill (from impeccable, Apache 2.0)
      NOTICE.md                -- Attribution notice
    github-triage/
      SKILL.md                 -- Read-only GitHub triage skill
  scripts/
    git.sh                     -- Git utilities
    hooks/
      comment-checker.sh       -- PostToolUse: comment density check
      pre-compact.sh           -- PreCompact: session state snapshot
      prompt-submit.sh         -- UserPromptSubmit: keyword and anti-pattern detection
      session-start.sh         -- SessionStart: repo conventions and metadata injection
      session-stop.sh          -- Stop: uncommitted changes warning
      subagent-start.sh        -- SubagentStart: Core Philosophy and role rules injection
      write-guard.sh           -- PreToolUse: read-before-edit enforcement
  .github/
    hooks/
      quality.json             -- VS Code workspace/plugin hooks (the 7 configured events, relative paths)
```
