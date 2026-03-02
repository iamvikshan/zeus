# Zeus

A multi-agent orchestration system for VS Code Copilot that enables complex software development workflows through intelligent agent delegation and parallel execution.

> Built upon the foundation of [copilot-orchestra](https://github.com/ShepAlderson/copilot-orchestra) by ShepAlderson, with agent naming conventions inspired by [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode).

> **Note:** Best supported on VS Code Insiders (as of January 2026) for access to the latest agent orchestration features.

## Overview

This repository contains custom agent prompts that work together to handle the complete software development lifecycle: **Planning → Implementation → Review → Commit**. The system uses a conductor-delegate pattern where a main orchestrator (Zeus) coordinates specialized subagents to efficiently tackle complex development tasks.

## Architecture

### Primary Agents

- **Zeus** (`zeus.agent.md`) - The ORCHESTRATOR
  - **Model:** Claude Opus 4.6 (copilot)
  - Orchestrates the full development lifecycle
  - Delegates to specialized subagents for research, implementation, and review
  - Manages context conservation and parallel execution
  - Handles phase tracking and user approval gates

- **Prometheus** (`prometheus.agent.md`) - The AUTONOMOUS PLANNER
  - **Model:** GPT-5.2 High (if reasoning set to high, check requirements block below)
  - Researches requirements and analyzes codebases
  - Writes comprehensive TDD-driven implementation plans
  - Automatically hands off to Zeus for execution
  - Supports parallel research across multiple subsystems

### Specialized Subagents

- **Athena** (`athena-subagent.agent.md`) - THE RESEARCHER
  - **Model:** GPT-5.2 (copilot)
  - Gathers comprehensive context about tasks
  - Can delegate to Hermes for large-scope research
  - Returns structured findings to parent agents
  - Supports parallel research across independent subsystems

- **Hephaestus** (`hephaestus-subagent.agent.md`) - THE IMPLEMENTER
  - **Model:** Claude Opus 4.6 (copilot)
  - Executes implementation following strict TDD principles
  - Writes tests first, then minimal code to pass
  - Handles linting and formatting
  - Can be invoked in parallel for disjoint features

- **Hermes** (`hermes-subagent.agent.md`) - THE SCOUT
  - **Model:** Gemini 3 Flash (Preview) (copilot)
  - Rapid file/usage discovery across codebases
  - Read-only exploration (no edits/commands)
  - Returns structured results with file lists and analysis
  - MANDATORY parallel search strategy (3-10 simultaneous searches)

- **Themis** (`themis-subagent.agent.md`) - THE REVIEWER
  - **Model:** GPT-5.2 (copilot)
  - Reviews code for correctness, quality, and test coverage
  - Returns structured feedback (APPROVED/NEEDS_REVISION/FAILED)
  - Can be invoked in parallel for independent phases
  - Focus on blocking issues vs nice-to-haves

- **Aphrodite** (`aphrodite-subagent.agent.md`) - THE UI/UX SPECIALIST
  - **Model:** Gemini 3 Pro (Preview) (copilot)
  - Implements user interfaces, styling, and responsive layouts
  - Expert in modern frontend frameworks and tooling
  - Follows TDD principles for frontend (component tests first)
  - Focuses on accessibility and responsive design

## Key Features

### � Context Conservation: The Game Changer

**Why This Matters:** Traditional single-agent approaches force one model to handle everything—research, implementation, review, documentation—all within a limited context window. This quickly exhausts precious tokens on context that could be used for your actual code.

**How Zeus Solves It:** By delegating tasks to specialized subagents, we radically improve context efficiency:

- **Researcher agents** (Athena, Hermes) read and analyze large codebases, returning only high-signal summaries—not the raw 50,000 lines of code
- **Implementer agents** (Hephaestus) focus solely on the files they're modifying, not rereading the entire project architecture
- **Reviewer agents** (Themis) examine only changed files, not context from the research phase
- **The Conductor** (Zeus) orchestrates everything without ever touching the bulk of your codebase

**The Result:** What would take 80-90% of a monolithic agent's context now takes 10-15%, leaving 70-80% more tokens for deeper analysis, better reasoning, and faster iterations.

---

### 🔄 Parallel Agent Execution
- Launch multiple subagents simultaneously for independent tasks
- Hermes: 3-10 parallel searches in first batch
- Athena: Parallel research across multiple subsystems
- Hephaestus: Parallel implementation for disjoint features
- Maximum 10 parallel agents per phase

### 🧪 Test-Driven Development
- Every phase follows red-green-refactor cycle
- Tests written first, run to fail, then minimal code
- Explicit test → code → test steps in all plans
- No manual testing unless explicitly requested

### 🤝 Proper Agent Handoffs
- VS Code Custom Agent handoff configuration
- Prometheus → Zeus automatic handoff option
- Each agent can declare available delegations
- Clear handoff workflow with user approval gates

### 📋 Structured Planning
- Zeus-compatible plan format
- 3-10 incremental, self-contained phases
- Open questions with options/recommendations
- Risk assessment and mitigation strategies

### 🔧 Universal Tooling with Opinionated Fallbacks

Agents auto-detect the project's tooling stack and never impose preferences on existing projects:

1. **Detection first** — `AGENTS.md` overrides → `package.json` scripts → config-file presence → lockfile/PM detection
2. **Fallback defaults** — When no project signal exists, agents default to: **Bun**, **TypeScript**, **ESLint**, **Prettier**, **`bun test`**
3. **Per-project override** — Any project can include a `tooling:` section in its `AGENTS.md` to set explicit commands

**Quality gate order** is enforced everywhere: **Format → Lint → Typecheck → Tests**

This means agents work correctly on any project (npm/yarn/bun, JavaScript, Vitest, etc.) while providing sensible defaults for greenfield work.

## Installation

1. **Clone or download this repository:**
   ```bash
   git clone https://github.com/iamvikshan/zeus.git
   ```

2. **Copy agent files to VS Code User prompts directory:**
   - **Windows:** `%APPDATA%\Code\User\prompts\` (or `%APPDATA%\Code - Insiders\User\prompts\` if using Insiders)
   - **macOS:** `~/Library/Application Support/Code/User/prompts/` (or `~/Library/Application Support/Code - Insiders/User/prompts/` if using Insiders)
   - **Linux:** `~/.config/Code/User/prompts/` (or `~/.config/Code - Insiders/User/prompts/` if using Insiders)

3. **Reload VS Code** to recognize the new agents

## Usage

### Planning a Feature with Prometheus

```
Plan a comprehensive implementation for adding user authentication to the app
```

Prometheus will:
1. Research the codebase (delegating to Hermes/Athena as needed)
2. Write a detailed TDD plan with 3-10 phases
3. Offer to invoke Zeus automatically or let you review first

### Executing a Plan with Zeus

```
Implement the plan devised by Prometheus
```

OR: Accept the hand-off from Prometheus by clicking `Start implementation with Zeus`


Zeus will:
1. Review the plan
2. Delegate Phase 1 implementation to Hephaestus
3. Delegate review to Themis
4. Present results and wait for commit approval
5. Continue through all phases

### Direct Research with Athena

```
Let @athena research how the database layer is structured
```

Athena will:
1. Delegate to Hermes for file discovery (if >10 files)
2. Analyze key files and patterns
3. Return structured findings

### Quick Exploration with Hermes

```
Let @hermes find all files related to authentication
```

Hermes will:
1. Launch 3-10 parallel searches immediately
2. Read necessary files to confirm relationships
3. Return structured results with file list and analysis

## Workflow Example

```
User: Prometheus, plan adding a user dashboard feature

Prometheus:
  ├─ @hermes (find UI components)
  ├─ @athena (research data fetching patterns)
  ├─ @athena (research state management)
  └─ Writes plan → Offers to invoke Zeus

User: Yes, invoke Zeus

Prometheus:
  └─ Zeus, implement the plan...

Zeus: Phase 1/4 - Test Infrastructure
  └─ @hephaestus Implement Phase 1
      ├─ Writes tests (fail)
      ├─ Writes minimal code
      └─ Tests pass ✓

Zeus: Reviewing Phase 1
  └─ @themis Review Phase 1
      └─ Status: APPROVED ✓

Zeus: Phase 1 complete! [commit message provided]
```

## Configuration

### Plan Directory
Agents check for plan directory configuration:
1. Look for `AGENTS.md` file in workspace
2. Find plan directory specification (e.g., `.plans/`, `plans/`)
3. Default to `plans/` if not specified

### Tooling Overrides via AGENTS.md

Projects can override the default tooling resolution by adding a `tooling:` block to their `AGENTS.md`:

```yaml
tooling:
  pm: bun
  format: bun prettier --write .
  lint: bun eslint .
  typecheck: bun tsc --noEmit
  test: bun vitest
  build: bun build
```

When present, these commands take highest priority. When absent, agents auto-detect from `package.json` scripts, config files, and lockfiles. If nothing is detected (greenfield project), agents fall back to Bun + TypeScript + ESLint + Prettier.

### Tool Requirements
All agents declare their required tools in YAML frontmatter:
- `agent` - For delegating to other agents
- `edit` - File editing capabilities
- `search` - Semantic/grep search
- `runCommands/runTasks` - Terminal execution
- etc.

### Handoff Declarations
Prometheus declares its handoff to Zeus:
```yaml
handoff:
  - label: Start implementation with Zeus
    agent: zeus
    prompt: Implement the plan
```

### Adding Custom Agents

You can extend the Zeus and Prometheus agents with your own specialized agents for domain-specific tasks (e.g., database experts, API specialists, security reviewers, etc.).

#### Quick Method: Let the AI Do It

The fastest way to add a custom agent is to simply ask:

```
@zeus Create a new subagent called Database-Expert that specializes in SQL optimization, schema design, and query analysis. Integrate it with Prometheus and Zeus so they can delegate database-related tasks to it.
```

Zeus will:
1. Create the agent file with proper YAML frontmatter
2. Add it to Prometheus's research delegation list
3. Add it to Zeus's implementation delegation list
4. Update documentation

#### Manual Method: Step-by-Step

**1. Create Your Agent File**

Create a new file in your prompts directory: `YourAgent-subagent.agent.md`

```yaml
---
description: 'Brief description of what this agent does'
argument-hint: What kind of task to delegate (e.g., "Analyze database schema")
tools: ['search', 'usages', 'edit', 'runCommands', ...]  # Tools your agent needs
model: Claude Sonnet 4.5 (copilot)  # Or GPT-5.2, Gemini, etc.
---

You are a [ROLE] SUBAGENT called by a parent CONDUCTOR agent.

**Your specialty:** [Describe the domain expertise]

**Your scope:** [Define what tasks this agent handles]

**Core workflow:**
1. [Step 1 of your agent's process]
2. [Step 2 of your agent's process]
3. [Return structured findings/results]

[Add any additional instructions, constraints, or examples]
```

**2. Integrate with Prometheus** (for research tasks)

Edit `prometheus.agent.md` and add your agent to the research delegation section:

```markdown
**YourAgent-subagent**:
- Provide a clear research goal related to [domain]
- Instruct to analyze [specific aspects]
- Tell them to return structured findings
```

Also add to Prometheus's constraints if it shouldn't delegate to your agent:
```markdown
- You CAN delegate to YourAgent-subagent for [domain] research
```

**3. Integrate with Zeus** (for implementation/review tasks)

Edit `zeus.agent.md`:

a. Add to the subagent list at the top:
```markdown
6. YourAgent-subagent: THE [ROLE]. Expert in [domain expertise]
```

b. Add to the subagent instructions section:
```markdown
**YourAgent-subagent**:
- Use #runSubagent to invoke for [task type] tasks
- Provide [specific context needed]
- Instruct to follow [workflow/principles]
- Remind them to report back with [expected output]
```

**4. Test Your Integration**

Try invoking your agent:
```
Let @YourAgent analyze the current database schema
```

Or through Zeus:
```
@zeus Use YourAgent to optimize our SQL queries in the user service
```

**5. Document Usage** (Optional)

Add an entry to the README's Specialized Subagents section describing when to use your custom agent.

#### Best Practices for Custom Agents

- **Single Responsibility**: Each agent should have one clear domain of expertise
- **Clear Scope**: Define exactly what the agent does and doesn't handle
- **Model Selection**: Choose the right model for the task (Sonnet for complex reasoning, Flash for speed, GPT for research)
- **Tool Minimalism**: Only declare tools the agent actually needs
- **Return Format**: Always return structured findings (not raw dumps)
- **Parallel-Aware**: Consider if your agent can run in parallel with others

#### Example Custom Agents

- **Security-Auditor**: Reviews code for vulnerabilities, dependency issues, auth flaws
- **Performance-Analyzer**: Profiles code, identifies bottlenecks, suggests optimizations
- **API-Designer**: Reviews/designs REST/GraphQL APIs, ensures consistency
- **Documentation-Writer**: Generates comprehensive docs from code
- **Migration-Expert**: Handles database migrations, version upgrades, refactoring

## Requirements

- **VS Code Insiders** (recommended for latest agent features and bug fixes)
- **GitHub Copilot** subscription with multi-agent support
- **VS Code Settings:**
  ```json
  {
    "chat.customAgentInSubagent.enabled": true,
    "github.copilot.chat.responsesApiReasoningEffort": "high"
  }
  ```
  - `customAgentInSubagent.enabled`: Allow subagents to use custom agents defined in a '-agents.md' file like the ones above 
  - `responsesApiReasoningEffort`: Set to "high" for enhanced reasoning in planning agents (GPT models)

## Best Practices

1. **Use Prometheus for complex features** - Let it research and plan before implementation
2. **Leverage parallel execution** - Invoke multiple Hermes/Athenas for large tasks
3. **Trust the TDD workflow** - Each phase is self-contained with tests
4. **Review before proceeding** - Check completed phases before moving forward
5. **Commit frequently** - After each approved phase after properly testing and ensuring phase functionality
6. **Delegate appropriately** - Let subagents handle heavy lifting


## License

MIT License

Copyright (c) 2026 Zeus Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

## Acknowledgments

This project builds upon the excellent work of:
- **[copilot-atlas](https://github.com/bigguy345/Github-Copilot-Atlas)** by [bigguy345](https://github.com/bigguy345) - Inspiration for agent orchestration patterns and tooling integration
- **[copilot-orchestra](https://github.com/ShepAlderson/copilot-orchestra)** by [ShepAlderson](https://github.com/ShepAlderson) - Foundation and concept for multi-agent orchestration
- **[oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode)** by [code-yeongyu](https://github.com/code-yeongyu) - Inspiration for agent naming conventions and templates

---

**Note:** These agents are designed to work together. While individual agents can be used standalone, the full power comes from Zeus orchestrating the complete workflow with intelligent delegation and parallel execution.
