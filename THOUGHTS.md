# AI Philosophy

> This document captures our principles for AI-driven development. It is designed to be dropped into any project repository to give agents and collaborators context on how we work.

---

## 1. The Power of N

We believe in generating **multiple variations** early and cheaply rather than perfecting a single plan upfront.

- Generate 20-25 variations of a plan or spec across different models and prompt styles.
- Compare variations to find **common gaps, weaknesses, and blind spots** that no single version reveals.
- Feed those findings back in to improve the plan — this is where the real leverage is.
- A better plan can jump a project from 50% to 80%+ complete before a line of implementation code is written. For scripts, this can approach 100%.
- Run **3 rounds of iterative feedback** before executing at scale. Each round should tighten the spec and reduce surprises.

### Why This Works
- Language models are **non-deterministic** — this is a feature, not a bug.
- Varying prompts, models, and framing produces genuinely different perspectives.
- You discover things you didn't know you wanted.
- You get different levels of finish and creativity depending on model selection and prompt nuance.
- The variations themselves become early test cases.

### Model Selection
- We are **model-agnostic** — no bias toward any single model or provider.
- Different models bring different strengths. Vary model selection deliberately to get diverse output.
- The right model depends on the task, the required creativity, and the level of finish needed.

---

## 2. Custom AI Tooling

Every project should have **purpose-built, AI-friendly tools** — not just prompts.

### JSON for Models, Markdown for Humans
- Tools built for AI consumption should output **JSON** — models parse it cleanly, and we process it with `jq`.
- When humans need to interact with AI output (review, approve, edit), use **Markdown** — readable in dev containers, IDEs, and GitHub.
- The choice between JSON and Markdown depends on **who is consuming the output**:
  - Model-to-model workflows: JSON task structures
  - Human-in-the-loop workflows: Markdown files, readable docs

### Project-Specific Harnesses
- Each project gets its own customised agent harness — there is no one-size-fits-all runner.
- Harness design depends on: task size, task volume, human involvement level, and loop duration.
- We maintain a library of agent runners and adapt them per project.

### Tool Development Lifecycle
- Tools start rough — **ship early, refine through use**.
- AI agents are first-class testers of our tools. During any interactive session, we regularly talk to agents about our tool collection: what's working, what's awkward, what's missing.
- Agent feedback gets fed into the next patch. Because we control our own tools, we can iterate fast — we can even replace existing standard tools with our own if it serves the workflow better.

### Tool Telemetry and Adoption Research
We embed lightweight telemetry directly into our custom tools — this is non-negotiable for understanding real-world usage.

- Every custom tool should include a **telemetry hook**: a simple function call that logs when the tool was invoked, with what command, parameters, and options.
- Telemetry writes to a temporary coded file (or a known high-level port/endpoint) — something reliable and independent of any agent harness or dashboard.
- This solves a real problem: agent harnesses, dashboards, and model-reported tool use are **unreliable for accounting**. They may be prompt-driven, misconfigured, or scattered across machines. The only trustworthy signal that a tool ran is **the tool itself reporting that it ran**.
- Telemetry captures: tool name, command/subcommand, parameters, options, timestamp, environment context.

### Adoption Experiments
We actively research how AI agents discover and prefer tools:

- **Do agents find our tools naturally**, or do we need to advertise them via prompts and system instructions?
- **Can we replace standard binaries** with our tools and see seamless adoption, or do agents default to known commands?
- **Do we need to simulate standard command interfaces** (same flags, same output format) to get agents to use our tools over built-in alternatives?
- We track tool call rates against different prompt styles and environment configurations to understand what drives adoption.
- The goal: understand the relationship between **environment setup, prompt design, and tool call frequency** — and use that data to make our tools the path of least resistance for agents.

---

## 3. Environments Over Prompts

Prompts matter, but **environments are greater leverage**.

A well-configured environment lets a model do more with less instruction. A poorly configured environment means fighting the setup instead of doing the work.

### Environment Tiers

| Use Case | Platform | When To Use |
|---|---|---|
| Human-interactive dev | **GitHub Codespaces** | Day-to-day development, pair programming with AI, human review needed |
| Headless model loops | **Docker Desktop / Dev Containers** | Local autonomous agent loops, no human interaction needed |
| Remote headless loops | **Azure Container Instances (ACI)** | Cloud-based autonomous loops, scalable, no local resources needed |
| Parallel projects | **Ona** (formerly Gitpod) | Many projects running simultaneously, integrated AI agent, CLI-driven |

### CLI-First Development
- Our primary interface is **VS Code + Claude CLI / Codex CLI**.
- We prefer CLI tools over GUI-based AI extensions (Cursor, Windsurf, sidebar extensions).
- CLI tools are improving rapidly and sit closer to both code and operations — this matters because much of what we do is ops, not just code.
- We have used GUI tools in the past; our philosophy going forward is CLI-first.

### Container Discipline
- Use **dedicated VS Code profiles** for container work — never the default profile.
- **Do not sync profiles** — each container gets a fresh start.
- Operations agents must have **safe, secure knowledge** of their environment without leaking secrets.
- Use the **devcontainer CLI** as a standard tool for spinning up and managing dev containers.

---

## 4. Environment as Instruction

We believe that a well-prepared environment **instructs the model better than any prompt can**. What's already installed, what's locked down, what's available on PATH — these are signals that shape agent behaviour more reliably than written instructions.

### Three Layers of Environment Control

1. **Hard controls** — chroot jails, filesystem restrictions, network policies, sandboxed runtimes. The agent physically cannot do the wrong thing.
2. **Soft controls** — intent-driven scripts that shape the environment through code: building dev containers, provisioning Azure infrastructure, managing keys and credentials, ensuring prerequisites are in place.
3. **Indicators** — the presence or absence of tools, configs, auth tokens, and environment variables. These are passive signals that tell the model what's possible without explicit instruction.

### Defensive Scripting: Assume Nothing

Every script should imagine it could be run at **any point in the chain** — from a fully provisioned machine to a bare system with nothing installed. Scripts must:

- Check if the environment is ready (tools installed, commands on PATH, users authenticated)
- Check for prerequisites at multiple levels: command exists, command has expected options, user is logged in, config files are present
- Handle the full spectrum gracefully — from "everything is already done" to "we need to start from scratch"

### Multi-Stage Intent Scripts

Scripts do not have to accomplish everything in a single prompt. A script can be **multi-stage**:

1. **Stage 1: Reconnaissance** — Send an initial prompt to explore the environment. Have the agent check installations, prerequisites, and auth state. Return a structured JSON response with a decision.
2. **Stage 2: Decision gate** — Parse the JSON response. Use a simple schema: `{ "ready": true/false, "reason": "..." }`. If not ready, either stop execution with a clear message or branch to a different intent.
3. **Stage 3: Execution** — If the environment passes the gate, proceed with the main intent. Fork the session ID if needed to run parallel checks or installations.
4. **Stage 4: Verification** — Confirm the outcome, report back.

### Structured Decision Gates

- Use **JSON schema objects** to get clean, parseable decisions from agents: a boolean (`ready`, `installed`, `authenticated`) plus a `reason` or `details` field.
- These responses can be passed back to the user, used to branch script logic, or fed into the next stage.
- The user sitting at the prompt should always see **progress** — use echo/Write-Host statements inside conditionals so the script feels responsive, not silent.

### Session Continuity

- The initial prompt can return a **session ID** that subsequent stages fork from.
- This allows multiple iterations over the same session: check installation, check prerequisites, make a go/no-go decision — all with shared context.
- This is cheaper and more coherent than starting fresh prompts for each stage.

---

## 5. Iterative Refinement as a First-Class Process

Refinement is not an afterthought — it is the process.

1. **Draft** — Write the initial intent/spec/prompt.
2. **Generate N variations** — Run it through multiple models and framings.
3. **Compare and critique** — Identify gaps across variations.
4. **Feed back** — Incorporate findings into the next draft.
5. **Repeat 3 times** — Three rounds of feedback is the sweet spot before scaling.
6. **Execute** — Run the refined version at scale.
7. **Observe** — Capture edge cases from real environments.
8. **Incorporate** — Feed real-world lessons back into the prompt for the next iteration.

This loop never truly ends — scripts and specs get more robust with every pass through real environments.

---

## 6. The Productivity Multiplier

### Historical Context
In 2008, 400 lines of clean, committed code in a single day was a victory. One commit a day, one pull request a week — that was the reality. Getting developers to document their work, constrain commits to singular features, and maintain good commit nomenclature was a constant battle.

Today, it is not unusual to wake up to **25 pull requests across different projects** — because the environment does the work while you sleep.

### It's As Easy to Create As It Is to Delete
This is a core belief. The cost of creation has collapsed. What matters now is **controlling the environment** so that creation happens reliably, repeatedly, and in parallel.

### Parallel Autonomous Workloads
We operate across multiple concurrent execution surfaces:

- **Git worktrees** — multiple branches worked on simultaneously in the same local repo
- **Remote Codespaces** — cloud-hosted dev environments with full IDE access
- **Deployed dev containers** — headless containers running agent loops (local Docker or remote ACI)
- **Local dev containers** — lightweight local environments for faster iteration
- **Claude Code for Web** — browser-based agent sessions for quick parallel tasks
- **Ona** — managed parallel workspaces with integrated AI agents

In practice, this means **multiple threads working for you** — locally, remotely, and across platforms — simultaneously. Extended workshop agents can run overnight, producing pull requests and results while you sleep.

### The Environment Standardisation Goal
The long-term vision is a **tool that controls and standardises environments** across all experiments and projects. This tool would:
- Ensure consistent base configurations across local, remote, and container environments
- Manage the deployment process for spinning up new workloads
- Reduce the friction of going from zero to a productive agent loop in any environment
- Make it trivial to replicate an environment across machines, clouds, and projects

---

## 7. The Operator Pattern

Every working environment has a **consistent operator** — one for Windows, one for Linux. The operator is the meta-system that controls the other systems.

### What the Operator Does
- Understands the target environment and its current state
- Sets up environments and enforces core skills
- Ensures prerequisites are installed and configured
- Understands how we work and how we manipulate environments
- Acts as the persistent, knowledgeable session that other agents defer to

### Default Working Setup
The common default is:
- **One Claude session** in the operator role — managing the environment, running intent scripts, enforcing standards
- **One Codex session** in the codebase — writing and modifying code within the controlled environment

The goal is to make the environment **so good that we can leave it running while we sleep**.

---

## 8. Human Attention Is the Sacred Resource

### The Shift
We've gone from an era where **writing code was hard** to one where writing code is easy and agents produce it in volume. But volume brings risk — bad quality code, slop, and unreviewed changes can flood through the system.

The scarce resource is no longer code production. **The scarce resource is human attention.** We must protect it.

### Guarding Attention
Every system we build should minimise the demand on human attention by maximising **automated validation**:

- **Self-validation** — give agents ways to check their own work before surfacing it to a human. The more ways an agent can mark its own homework objectively, the less noise reaches the user.
- **Test-Driven Development** — a core philosophy. Tests are the objective validation layer. If an agent's work passes the test suite, it has earned the right to request human review.
- **Controlled guard rails** — structured environments, quality gates, and gated branches all reduce the chance of bad work reaching a human.
- **The goal**: by the time work reaches a human, it should already have been validated, tested, and reviewed by automated systems. Human attention should be spent on judgement, not verification.

### Self-Improving Systems
We are committed to systems that **get better without human intervention**:
- Agents iterate, test, and refine in loops
- Edge cases discovered in one run feed back into the next
- Environments become more robust with each deployment
- The human steps in to set direction, not to babysit execution

---

## 9. Agent-Reviews-Agent

### The Principle
Agents should review the work of other agents. Never rely on a single agent to both produce and validate.

### Why This Works
- A **focused auditing agent** is naturally critical — it has no attachment to the code it's reviewing.
- By separating the builder from the reviewer, you get tighter feedback loops in autonomous cycles.
- Review agents can be given narrow, specific mandates: check for security issues, check for test coverage, check for adherence to commit conventions, check for slop.
- This mirrors human code review but runs at machine speed and machine cost.

### In Practice
- In loop-based workflows, the review agent runs after each iteration before work is promoted.
- Review agents can reject work and send it back for another iteration — no human needed.
- The combination of TDD + agent review creates a **double gate**: objective tests plus subjective review, both automated.

---

## 10. Documentation That Drives Agents

We've had significant success with structured documentation as agent input:

| Document Type | Purpose |
|---|---|
| **User stories** | Define intent and acceptance criteria in agent-friendly format |
| **ADRs** (Architecture Decision Records) | Capture decisions and constraints so agents don't revisit settled questions |
| **Q&A knowledge docs** | Pre-answer common questions agents will have about the project |
| **Anti-requirements** | Explicitly state what NOT to build, what NOT to do — constraints are as valuable as requirements |
| **Constraints docs** | Hard boundaries: tech choices, performance budgets, security rules, scope limits |

### Why Anti-Requirements Matter
Agents are eager builders. Without explicit "do not build" instructions, they will over-engineer, add features, and expand scope. Documenting what is **out of scope** is as important as documenting what is in scope.

---

## 11. The Overall Process: From Idea to Autonomous Execution

### Phase 1: Ideation and Documentation
It starts with a question: **"What do I want to build?"**

Documentation can enter the system in many ways:
- **Voice brainstorming** — talk into Windows Recorder, transcribe, then feed into Claude or Codex in plan mode to structure the ideas.
- **Agent-assisted planning** — use agents to critique and refine plans iteratively.
- **External critique** — take generated plans into tools like NotebookLM and have the hosts critique them, looking for holes and gaps.
- **Agent scoring** — use review agents to generate **non-arbitrary, structured scoring** against plans and implementations. With attention being scarce, we need objective signals we can trust without reading every line.

### Phase 2: Environment Lockdown
Before agents start building, the environment must be controlled. Trust comes from constraints:

#### File System Controls
- **chroot jails** — the ultimate sandbox. Agents can only operate within their designated filesystem.
- **Controlled PATH** — replace or restrict system binaries with our own personalised tools. Remove access to binaries agents don't need.
- **Immutable dependency files** — use `chattr +i` (Linux) or equivalent to make files like `package-lock.json`, `requirements.txt`, and lock files read-only. Agents cannot change the dependency tree without human approval.

#### Role Separation (Separation of Concerns)
This is critical for autonomous loops:
- **Code agents** can write implementation code but **cannot modify tests**.
- **Test agents** can write and maintain tests but **cannot modify implementation code**.
- Neither agent can do the other's job. This prevents the most common failure mode: **an agent changing the tests to make bad code pass**.
- If a code agent believes a test is wrong, it must submit a formal objection to the test agent — and that objection will be scrutinised.
- This enforced separation creates genuine adversarial validation.

#### Remote Environment Discipline
- Prefer remote environments (Codespaces, ACI, Ona) for autonomous loops — better isolation, better controls.
- Ensure good binaries are in place, structured commit messages are enforced, and documentation conventions are established **before** agents start work.

### Phase 3: Parallel Execution
With the environment locked down and the plan refined:
- Spin up multiple agents across worktrees, containers, and platforms.
- Let them run — overnight if needed.
- Trust the gates: TDD, agent review, quality pipelines, structured commits.
- Human attention is reserved for direction-setting and judgement calls, not babysitting.

### Phase 4: Review and Integration
- Review what the agents produced — guided by test results, agent review scores, and structured commit history.
- Merge what passes the gates. Reject and re-queue what doesn't.
- Feed lessons back into the environment and documentation for the next cycle.

---

## 12. You Are an Architect of Constraints, Not a Writer of Code

### The Mindset Shift
Your job is not writing code. **Code is cheap.** Your job is designing the constraints, environments, and validation systems that let agents produce reliable work without your constant supervision.

The goal with every project is the same: get the environment secure, get the plan defined, get the gates in place — and then **get it out of your immediate inbox**. Even if it's just 10-15 iterations of a task running autonomously, that frees you to move to the next project.

### The Container Trap
A common mistake is holding onto interactive sessions too long. When you're sat in a dev container interacting with a model through VS Code or Codespaces, that model is consuming your **active attention** — the most scarce resource you have.

The progression should be:
1. **Interactive** — set up the environment, define the plan, establish the constraints.
2. **Semi-autonomous** — validate that the agent can make progress on its own, intervene only when blocked.
3. **Fully autonomous** — the environment is good enough that the agent runs unattended. Move on.

The faster you move from stage 1 to stage 3, the more projects you can run in parallel.

### The Power of a Good Environment
A truly well-controlled environment means your prompt can be as simple as:

> "Complete my task list."

Four words. The agent finds the work, pulls in the required context, has the right tools on PATH, knows the constraints from the documentation, and has auto-memory feeding it information at the right times. **That is the power of environment over prompt.**

Prompts matter — but they should be the last mile, not the load-bearing structure.

### The Personal Prompting Style
- Programming models is non-deterministic. What works for one developer will not work for another.
- Every developer carries **biases in how they speak to models** — phrasing habits, assumptions, levels of specificity — and these biases get baked in over long durations.
- You have to do the work to understand your own prompting style: what gets results for you, what leads to confusion, what models respond to your framing best.
- General tips and pointers help, but the real skill is **self-awareness about your own patterns**.
- Refactoring your prompts to remove unhelpful biases takes deliberate effort — it doesn't happen passively.

### Practical Minimum
- Run **at least 3 variations** of any meaningful plan or task — different models, different context, different framing.
- Try planning beforehand vs letting the model discover. Try terse prompts vs detailed ones. Try different model families.
- Find what works for you **most of the time** — then build your environments and scripts around that style.

---

## 13. Branch Naming and Git Workflow

### Branch Naming Convention
Branch names mirror the commit message type system. Always branch off `development`:

```bash
# Features
git checkout development
git checkout -b feat/user-auth-flow

# Bug fixes
git checkout development
git checkout -b bug/null-ref-on-login

# Refactors
git checkout development
git checkout -b refactor/split-api-handlers
```

The branch name prefix matches the commit type: `feat/`, `bug/`, `fix/`, `refactor/`, `chore/`, `ops/`, `docs/`, etc. This keeps branch names and commit messages aligned — an agent can infer the commit type from the branch it's on.

### Variation Branches
When running multiple variations (Power of N):
- Keep a **base branch** for the task: `feat/user-auth-flow`
- Append variation numbers: `feat/user-auth-flow-v1`, `feat/user-auth-flow-v2`, `feat/user-auth-flow-v3`
- This makes it trivial to diff variations against each other

> **Note:** In environments like Claude Code for Web or Codex, you may not have full control over branch naming yet. Work within the platform's constraints and document the mapping.

### The Git Steward Agent
We recommend having **one dedicated agent per repository** that owns all git operations for the early lifecycle of a project:

- This agent handles **all commit messages, branch creation, and merges** — at least for the first ~25 commits.
- Having one consistent agent doing the initial commit nomenclature **embeds a truth** in the repository. The structured history becomes self-reinforcing.
- This agent runs as an interactive session — you sit in its shell and let it manage the git ceremony while you (or other agents) focus on the actual work.
- After the initial structure is embedded, the `smart-commit/SKILL.md` takes over — other agents can now follow the established pattern because it's visible in the history and codified in the skill.

### Why Consistency in the First 25 Commits Matters
- Early commits set the tone. Agents reading git history will pattern-match against what they see.
- If the first 25 commits are clean, structured, and consistent, agents will naturally continue that pattern.
- If the first 25 commits are messy, every future agent will inherit that mess — and prompting them out of it costs attention.
- Front-load the discipline. It pays compound interest.

---

## 14. Context Window Philosophy

### Fresh Context on Every Loop
When running autonomous loops (for/while loops over tasks), each iteration should start with a **fresh context window**. By the time you're looping, the environment should be locked down enough that the agent doesn't need conversational memory from prior iterations.

A fresh context means:
- No accumulated confusion or hallucinated state from previous iterations
- The agent starts clean and pulls in exactly what it needs from the environment
- Each iteration is independently valid — if one fails, it doesn't poison the next

### The Dream State
The ideal is an environment so well-constructed that a fresh agent with an empty context window can:
- Read the structured commit history and understand what's been done
- Read the documentation (AGENTS.md, AI_PHILOSOPHY.md, task lists) and understand what to do
- Discover the available tools, their JSON outputs, and their constraints
- Pull in auto-memory and context at the right moments
- Start producing valuable work — without a single line of conversational preamble

### AI-First Environment Design
We are building toward an **AI-first operating system** — not restricting agents, but guiding them through environment design:

- **Tools are designed for AI consumption first** — JSON output, schema-driven, parseable
- **Restrictions guide, not block** — controlled PATH, curated binaries, immutable lock files. The agent can do everything it needs to; it just can't do things it shouldn't.
- **The environment tells the story** — documentation, commit history, task files, and tool output collectively give the agent everything it needs. The prompt becomes minimal.
- **Design for the agent, not for the human** — the human sets up the environment; the agent lives in it. Optimise the environment for agent productivity, not human readability (though human readability is a useful side effect).

The core principle: **don't restrict the agent, guide it through the environment.** Give it what it needs to be productive. Let the constraints be structural, not conversational.

### Interactive Context Management (Pre-Loop Phase)
When you're still in the early days — building the environment, not yet looping — you work interactively inside the operator prompt:

- Work the context window down to about **30% remaining**, then compact.
- Compact with an intentional message: *"Take our philosophies as a filter and distil our knowledge base through them."* This resets the context while preserving the philosophical frame.
- Then start again — continue building up the environment with the compacted context as your foundation.

### Building the Environment in Layers
The environment is built iteratively, not deployed once and forgotten:

1. **Layer 1: The Operator** — the persistent agent session you sit inside. Present on every system during bootstrapping, building, and deployment. Lives in its own project with its own setup. May be removed when transitioning to a cloud environment.
2. **Layer 2: Documentation** — AGENTS.md, AI_PHILOSOPHY.md, skills, rules, hooks. Each iteration adds more context around the agent, making the environment stickier and more self-explanatory.
3. **Layer 3: Validation** — test the environment itself. Use TDD not just for code but for the environment: are prerequisites installed? Do cross-platform commands work? Are tools discoverable? Use focused agents trained to test one specific thing.

### Making Environments Sticky
An environment is "sticky" when an agent dropped into it naturally does the right thing. We achieve this through multiple reinforcing layers:

| Mechanism | Framework | Purpose |
|---|---|---|
| **Intent-driven scripts** | Any | Give agents specific tasks with clear outcomes |
| **Skills** | Claude CLI / Codex | Reusable capabilities agents can invoke (e.g. smart-commit) |
| **Rules** | AGENTS.md / CLAUDE.md | Project-level constraints and conventions |
| **Hooks** | Agent framework hooks | Automated triggers on events (pre-commit, post-tool-call) |

Each project should customise all four layers. The more layers that align, the less the agent can drift — and the less prompt you need to write.

### Templates Are Starting Points, Not Sacred
We work from **template scripts and prompts** — they give us a consistent starting point across projects. But templates are never final. Every project should customise, iterate, and evolve its templates without hesitation.

- Start from the template. Get it working.
- Customise it for your project's specific needs.
- Iterate over the customisation as you learn more about the project.
- Never be scared to diverge from the template — the template exists to save you the first 70%, not to constrain the last 30%.

### Init Scripts: The Onboarding Entry Point
Every project starts with:
- **`init-claude.sh`** — bootstraps Claude CLI with project-specific skills, hooks, rules, and restrictions.
- **`init-codex.sh`** — same for Codex CLI.

These are the **first thing that runs** when an agent enters the project. They are the bridge between a generic agent and a project-ready agent. Customise them aggressively for each project — they are the single highest-leverage file for rapid onboarding.

---

## 15. Git Strategy and Quality Gates

### Branching Model
- Every repo has a **`main`** branch and a **`development`** branch.
- `main` is protected — code only enters `main` after passing a full gate of tests.
- All active work happens on `development` or feature branches off `development`.

### Quality Gates
- **CodeRabbit** is configured on repositories we deem important — it provides automated AI code review on pull requests.
- **GitHub Actions / Workflows** (`pipeline.yaml`) run the full test suite on every PR targeting `main`.
- Nothing merges to `main` without passing both automated review and CI pipeline.

### Test-Driven AI Development
- AI models are **more responsive to test-driven development** than to spec documents or prose descriptions.
- We invest upfront time in **generating comprehensive test cases first**, before any implementation.
- Once tests are in place, we let multiple model variations attempt to build the project against those test cases.
- This combines naturally with the **Power of N**: different prompts and models will get further into the test suite than others — we can measure progress by how many tests pass.
- The test suite becomes the **objective scoring function** for comparing prompt variations, model choices, and implementation approaches.
- Spending time on better tests has the same leverage effect as spending time on better plans — it compounds across every variation that runs against them.

---

## 7. Structured Commit Messages (Golden Rule)

This is a **non-negotiable rule** across all repositories.

### The Problem
AI agents frequently need to parse, search, and reason over git history. Freeform commit messages are hostile to programmatic parsing. If an agent can't reliably extract intent from commit history, it loses a critical context source.

### The Format
Every commit message follows:

```
type(locator): <description>
```

- **`type`** — the category of change: `feat`, `fix`, `bug`, `design`, `refactor`, `test`, `docs`, `chore`, `ci`, `perf`, `ops`, etc.
- **`locator`** — a project-specific identifier scoped to the area of change: a component, module, script name, or subsystem. Each project defines its own nomenclature.
- **`description`** — concise, imperative description of what changed.

### Examples
```
feat(gh-cli): add authentication check before install
fix(system-recon): handle missing git repos in Downloads
refactor(orchestrator): split prerequisite checks into stages
docs(philosophy): add structured commit message rule
chore(deps): bump claude cli to latest
ops(debug-logs): add timestamped log output to all scripts
```

### Why This Matters
- Agents can **programmatically filter** commits by type or locator (`git log --grep="^fix("`) — they can locate what they need and discard what they don't.
- Agents can **scope context** to relevant areas when investigating bugs or planning changes.
- Agents can **generate changelogs and summaries** by parsing structured history.
- It makes the Power of N work for git history — structured data is parseable data.

### Commit History Is Part of the Environment
This connects directly to **Environment as Instruction** (Section 4). Git history is not just a record — it is an environment signal. When commit messages are structured and parseable:
- Agents can **self-discover** what the project does, what has changed, and what the intent was — without being told explicitly in a prompt.
- Agents are **less likely to misunderstand** the project's purpose or make destructive changes, because the history tells a coherent story.
- You reduce prompt burden — instead of explaining the project in every prompt, you let the agent read the structured history and orient itself.
- A well-structured git history is a **passive guardrail**: agents that understand what was built and why are far less likely to tear it down.

### Implementation
- Every repository should have a **`smart-commit/SKILL.md`** that teaches the Claude CLI / Codex agent the project's specific commit nomenclature.
- This skill should be created early in the project lifecycle — before the first meaningful commit.
- The skill defines the valid types and locators for that project so agents produce consistent messages without being reminded each time.
