---
title: Environment Engineering Guide
filename: ENVIRONMENT_ENGINEERING_GUIDE.md
status: draft-v1
audience:
  - Environment builders
  - Operator authors
  - Toolsmiths
  - Senior implementers
authority:
  - Operational
  - Normative where explicitly stated
  - Experimental sections clearly marked
read_time: 25-35 minutes
related:
  - AI_PHILOSOPHY.md
  - AI_OPERATING_MODEL.md
  - REPO_STANDARDS.md
  - AGENTS.md
---

# ENVIRONMENT_ENGINEERING_GUIDE.md

> This guide defines how we engineer environments for AI-driven development. It is about implementation: tools, containers, controls, bootstrap scripts, discoverability, validation, and autonomous execution surfaces.
>
> It is intentionally technical. It should help a builder create an environment in which a capable agent can do the right work with minimal prompting.
>
> This document does **not** replace:
> - **`AI_PHILOSOPHY.md`** for the worldview and rationale
> - **`AI_OPERATING_MODEL.md`** for the workflow from idea to execution
> - **`REPO_STANDARDS.md`** for mandatory repo conventions such as branch naming, commit structure, and quality gates

---

## 1. Scope

This document covers the engineering of AI-ready environments:

- execution surfaces
- tool design
- environment bootstrap
- hard and soft controls
- discoverability signals
- structured decision gates
- session continuity
- context management
- environment validation
- documentation layers that make environments "sticky"

The core idea is simple:

> A strong environment reduces prompt burden, reduces drift, and increases reliable autonomous output.

An environment should carry as much intent as possible through its structure, tools, files, and controls. Prompts still matter, but they should not be load-bearing.

---

## 2. Design Principles

### 2.1 Environment Over Prompt

A well-prepared environment is a stronger instruction source than a long prompt.

The agent should be able to infer what is possible and what is expected from:

- what tools are installed
- what tools are missing
- what appears on `PATH`
- what files are present
- what files are immutable
- what documentation exists
- what auth is available
- what constraints are encoded in scripts and hooks

The goal is not to explain the project from scratch in every session. The goal is to make the environment itself tell the story.

### 2.2 AI-First, Human-Supportive

These environments are designed primarily for agent productivity, while still remaining inspectable by humans.

That means:

- outputs intended for model consumption should be structured and parseable
- outputs intended for human review should be legible in terminals, editors, and pull requests
- the system should expose the minimum ambiguity necessary for reliable automation

Human readability remains important, but machine readability must be treated as a first-class engineering requirement.

### 2.3 Defensive Scripting: Assume Nothing

Every script should behave as though it may be run at any point in the chain:

- on a fully provisioned machine
- on a partially configured environment
- on a new container
- on a machine where auth expired
- on a system missing dependencies
- on a repo that has not yet been initialized correctly

Scripts must not assume readiness. They must detect readiness.

### 2.4 Idempotency by Default

Bootstrap and intent scripts should be safe to run repeatedly.

A good environment script should be able to answer one of three questions cleanly:

1. **Already done** — no action required
2. **Not ready** — prerequisites missing; stop or branch
3. **Proceeding** — requirements satisfied; continue

Idempotency matters because autonomous systems retry, re-enter, and resume far more often than manual workflows do.

### 2.5 Visibility Beats Silence

Silent automation creates uncertainty.

Any script that takes real action should emit meaningful progress messages:

- what it is checking
- what it found
- whether it is branching
- whether it is stopping
- what the next step is

A responsive script is easier for both humans and agents to trust.

---

## 3. Output Design: JSON for Models, Markdown for Humans

### 3.1 JSON for Machine Decisions

When output is intended to be consumed by another model, parser, or orchestrator, prefer JSON.

JSON should be used for:

- readiness checks
- install checks
- auth checks
- structured tool responses
- branch decisions
- telemetry events
- machine-generated scorecards
- inter-stage handoffs

JSON makes branching, validation, and post-processing straightforward.

Example decision shape:

```json
{
  "ready": true,
  "reason": "git, node, and docker are installed; auth is present"
}
```

Recommended properties for decision objects:

- a primary boolean such as `ready`, `installed`, `authenticated`, or `passed`
- a concise `reason`
- optional `details`
- optional `next_action`

### 3.2 Markdown for Human Review

When humans need to inspect, approve, edit, or discuss output, prefer Markdown.

Markdown should be used for:

- task lists
- review notes
- ADRs
- user stories
- anti-requirements
- environment summaries
- implementation plans
- operator notes

Markdown works well in:

- VS Code
- GitHub
- dev containers
- terminal editors
- pull request discussions

### 3.3 Choose by Consumer, Not by Habit

The deciding question is not "What format do we like?" It is:

> Who consumes this output next?

- **Model-to-model** or **script-to-script**: use JSON
- **Human-in-the-loop**: use Markdown
- **Mixed workflows**: generate both or generate JSON with a Markdown render layer

---

## 4. Execution Surfaces and Environment Tiers

Different workloads require different execution surfaces. There is no single universal environment.

| Use case | Platform | When to use | Notes |
|---|---|---|---|
| Human-interactive development | **GitHub Codespaces** | Day-to-day paired work where human review is frequent | Strong for interactive development and repository-local setup |
| Local headless loops | **Docker Desktop / Dev Containers** | Autonomous or semi-autonomous loops on local infrastructure | Good for fast iteration and repeatable isolated setup |
| Remote headless loops | **Azure Container Instances (ACI)** | Cloud-based unattended loops and scalable execution | Strong isolation and no dependency on local resources |
| Parallel projects | **Ona** (formerly Gitpod) | Many concurrent environments with integrated agent workflows | Useful when running several projects or experiments in parallel |

### 4.1 Selection Guidance

Prefer the environment that matches the work mode:

- If a human is actively steering, use an interactive surface.
- If the loop is autonomous, prefer a headless containerized surface.
- If isolation matters more than convenience, prefer remote execution.
- If parallelism is the goal, prefer platforms designed for many simultaneous workspaces.

### 4.2 CLI-First Development

The default interaction model is CLI-first.

Preferred characteristics:

- tools close to the code
- tools close to operations
- tools scriptable from shells and pipelines
- tools composable with other tooling

CLI-first matters because much of the real work is not only code generation; it is environment manipulation, inspection, provisioning, validation, and orchestration.

### 4.3 VS Code Profile Discipline

For container-based work:

- use dedicated VS Code profiles
- do not use the default profile
- do not sync container-specific profiles automatically
- keep container tooling isolated from general desktop configuration

Fresh, purpose-specific profiles reduce cross-project drift and extension noise.

---

## 5. Control Model: Hard Controls, Soft Controls, Indicators

Environment control happens across three layers.

### 5.1 Hard Controls

Hard controls physically prevent classes of failure.

Examples:

- chroot jails
- sandboxed runtimes
- filesystem restrictions
- network policy restrictions
- permission boundaries
- immutable files
- read-only mounts

Use hard controls where the consequence of an error is unacceptable.

### 5.2 Soft Controls

Soft controls shape the environment through code and automation.

Examples:

- bootstrap scripts
- devcontainer setup
- provisioning scripts
- login checks
- prerequisite installers
- path assembly
- intent-driven wrappers
- staged execution runners

Soft controls are not absolute barriers. They are repeatable guidance mechanisms.

### 5.3 Indicators

Indicators are passive signals that influence agent behavior.

Examples:

- which binaries appear on `PATH`
- whether credentials are available
- whether task files exist
- whether documentation is present
- whether tools are discoverable
- whether hooks are installed
- whether a lock file is immutable

Indicators matter because agents infer capability from the shape of the environment.

### 5.4 Use the Three Layers Together

The most reliable environments combine all three:

- **Hard controls** stop dangerous actions.
- **Soft controls** make the right path easier.
- **Indicators** make the intended path obvious.

Do not try to solve everything with prompts when the environment can solve it structurally.

---

## 6. Tool Design for AI-Friendly Environments

### 6.1 Every Project Gets Project-Specific Harnesses

There is no one-size-fits-all agent runner.

Harness design should vary based on:

- task size
- task volume
- duration of loops
- human involvement
- repository conventions
- platform constraints
- validation requirements

A good harness fits the project, not the other way around.

### 6.2 Tool Development Lifecycle

Custom tools should start simple and improve through use.

Recommended lifecycle:

1. Ship an early version
2. Use it in live workflows
3. Let agents encounter the rough edges
4. Capture friction points
5. Patch the tool
6. Repeat

Agents are not just users of the toolchain. They are also effective testers of the toolchain.

### 6.3 Replace Standard Tools When It Serves the Workflow

If a custom tool is better suited to the environment, it is acceptable to replace or front existing standard binaries.

Valid reasons include:

- better machine-readable output
- safer defaults
- clearer environment integration
- telemetry support
- reduced ambiguity for agent users

Replacement is justified when it makes the intended workflow more reliable.

### 6.4 Tool Contracts Should Be Explicit

Every custom tool should define:

- purpose
- inputs
- outputs
- exit codes
- whether it is safe to retry
- whether it mutates state
- whether it requires auth
- whether it emits JSON, Markdown, or both

Ambiguity at the tool boundary causes unnecessary drift in downstream agents.

---

## 7. Tool Telemetry and Adoption Research

### 7.1 Telemetry is Normative

Every custom tool should include a lightweight telemetry hook.

The only fully trustworthy evidence that a tool ran is the tool reporting that it ran.

Do not rely exclusively on:

- dashboard summaries
- harness-reported tool use
- prompt transcripts
- model claims about what they did

Those signals may be incomplete, prompt-shaped, or fragmented across systems.

### 7.2 Minimum Telemetry Payload

A telemetry event should capture at least:

- tool name
- command or subcommand
- parameters
- options
- timestamp
- environment context
- caller context where available
- success or failure status
- duration if cheaply measurable

Example:

```json
{
  "tool": "system-recon",
  "command": "check",
  "params": {
    "target": "devcontainer"
  },
  "options": ["--json"],
  "timestamp": "2026-04-03T09:15:00Z",
  "environment": "codespaces",
  "status": "success",
  "duration_ms": 412
}
```

### 7.3 Telemetry Destination

Telemetry should write to a destination that is:

- simple
- reliable
- independent of the conversational agent layer
- available even when dashboards are down

Examples:

- a coded temp file
- a local append-only log
- a lightweight port or endpoint
- a structured event stream

Choose the simplest destination that is stable in the target environment.

### 7.4 Adoption Research is Experimental

The following questions are worth deliberate experimentation:

- Do agents discover our tools naturally?
- Do we need to advertise tools explicitly?
- Will agents adopt replacements for standard binaries?
- Do agents need familiar flags and interface shapes?
- Which prompt styles increase tool usage?
- Which environment cues make a tool the default path?

These are research questions, not settled truths. Measure them.

### 7.5 Run Adoption Experiments Deliberately

When running experiments, vary one thing at a time:

- prompt framing
- tool discoverability
- aliasing strategy
- PATH ordering
- output format
- interface familiarity
- environment documentation

Track tool-call rates and success outcomes across conditions. The goal is not vanity metrics. The goal is to learn what makes the correct tooling the path of least resistance for agents.

---

## 8. Bootstrap and Entry Points

### 8.1 Init Scripts are the Onboarding Bridge

Every project should define explicit onboarding entry points for agents.

Recommended files:

- `init-claude.sh`
- `init-codex.sh`

These scripts bridge the gap between a generic agent and a project-ready agent.

They should:

- install or verify required prerequisites
- expose the right tools
- load project skills
- install or verify hooks
- point to project rules and constraints
- verify authentication where needed
- emit a clear readiness result

### 8.2 Init Scripts Should Be Aggressively Customized

Do not treat init scripts as sacred templates.

Start from a working template, then customize hard for the project:

- repo layout
- stack
- test commands
- deployment target
- security boundaries
- local vs remote constraints
- operator expectations

Templates should save the first 70%, not dictate the final 30%.

### 8.3 Bootstrap Must Verify, Not Just Install

A bootstrap script is incomplete if it only installs binaries.

It should verify:

- command exists
- command is on `PATH`
- command supports required flags or subcommands
- user is authenticated when necessary
- config files exist
- project files exist
- permissions are correct
- required documentation layers are present

Verification is more important than installation.

---

## 9. Readiness Checks and Defensive Scripting

### 9.1 Readiness Checks Are Mandatory

Before meaningful execution, scripts should check readiness at several levels:

| Check | Example question |
|---|---|
| Command presence | Is `docker` installed? |
| Capability | Does this version support the required flag? |
| Auth | Is the user logged into GitHub, Azure, or the target service? |
| Config | Are required config files present and valid? |
| Permissions | Can the script read or write the required locations? |
| Environment mode | Are we in Codespaces, local Docker, or ACI? |
| Repo state | Are we in the right repository and branch context? |
| Restrictions | Are required files immutable or locked as intended? |

### 9.2 Check Capabilities, Not Just Presence

A command existing is not the same as it being usable.

Examples:

- `az` may exist but not be logged in
- `docker` may exist but the daemon may not be running
- `git` may exist but the repo may not be initialized
- `node` may exist but be the wrong major version

Capability checks reduce false positives.

### 9.3 Failure Modes Must Be Explicit

A readiness script should clearly distinguish between:

- not installed
- installed but unusable
- installed but unauthenticated
- installed and ready

That distinction matters because the remediation path is different in each case.

### 9.4 Every Check Should Have a Next Action

Do not stop at "not ready."

Return something actionable, such as:

- install missing dependency
- run login flow
- open the correct container
- switch to the intended path
- re-run bootstrap
- contact operator if hard controls prevent progress

---

## 10. Multi-Stage Intent Scripts

Scripts do not need to do everything in one pass. In many cases they should not.

### 10.1 Recommended Staging Model

#### Stage 1: Reconnaissance

Inspect the current environment.

Tasks may include:

- checking installed tools
- checking auth state
- checking repo structure
- checking available skills and hooks
- checking network or filesystem restrictions

Output should be structured.

#### Stage 2: Decision Gate

Turn reconnaissance into a clean branching decision.

Recommended minimal schema:

```json
{
  "ready": false,
  "reason": "docker is installed but the daemon is not running",
  "next_action": "start docker and rerun readiness check"
}
```

#### Stage 3: Execution

Proceed only if the gate passes.

Examples:

- run the main agent task
- provision the container
- execute the repair script
- launch autonomous loop
- invoke build or test workflow

#### Stage 4: Verification

Confirm that the intended outcome occurred.

Examples:

- expected files created
- services reachable
- tests pass
- container healthy
- outputs emitted in the expected format

### 10.2 Why Staging Matters

Staging improves:

- debuggability
- branchability
- observability
- reusability
- cost control
- failure isolation

It also avoids forcing a single prompt or script to carry too many responsibilities.

### 10.3 Branch Early, Not Late

If the environment is not ready, stop before expensive work begins.

The cheapest failure is the earliest one.

---

## 11. Structured Decision Gates

Structured decision gates let scripts and agents cooperate without ambiguity.

### 11.1 Recommended Shape

For simple binary gates, use:

```json
{
  "ready": true,
  "reason": "all prerequisites satisfied"
}
```

For richer gates, use:

```json
{
  "ready": false,
  "reason": "GitHub authentication missing",
  "details": {
    "gh_installed": true,
    "gh_authenticated": false
  },
  "next_action": "run gh auth login"
}
```

### 11.2 Gate Properties

A useful gate object should be:

- simple
- parseable
- stable across iterations
- easy to branch on in shell scripts
- understandable to humans when printed

### 11.3 Show Progress to the User

Even when the branch decision is machine-readable, the user should still see progress.

Examples:

- "Checking GitHub authentication..."
- "Docker detected; verifying daemon..."
- "Environment not ready: Azure login missing."
- "Proceeding to execution stage."

The JSON is for the system. The progress messages are for trust.

---

## 12. Session Continuity and State Management

### 12.1 Reuse Context Where It Helps

In staged workflows, an initial interaction can return a session identifier or reusable context handle.

That handle can then be used to:

- continue the same line of investigation
- fork parallel checks
- avoid restating context
- reduce redundant setup cost
- preserve coherence across related stages

### 12.2 Fresh Context for Autonomous Loops

Once the environment is mature enough for real looping, each iteration should start from a fresh context window.

Benefits:

- less accumulated confusion
- fewer hallucinated dependencies on earlier conversation
- each iteration stands on the environment, not memory
- one bad iteration does not poison the next

Long-lived conversational memory is most useful in the bootstrap phase, not the steady-state loop phase.

### 12.3 Externalize State into the Environment

Stable context should live in files, tools, docs, and structured history, not only in chat memory.

Preferred external state carriers:

- task files
- ADRs
- AGENTS.md
- skills
- hook configuration
- test suites
- structured commit history
- telemetry logs
- machine-readable run summaries

The more context that lives outside the conversation, the more reproducible the workflow becomes.

---

## 13. Isolation, Lockdown, and Safe Execution

### 13.1 Filesystem Controls

Use filesystem restrictions to narrow the agent’s valid action surface.

Patterns may include:

- chroot jails
- designated working directories
- read-only mounts
- limited writable areas
- workspace-specific users
- temporary scratch locations

The environment should make the correct filesystem boundary obvious.

### 13.2 Controlled PATH

`PATH` is an instruction surface.

Use it deliberately:

- include tools the agent should use
- exclude tools the agent should not rely on
- front preferred wrappers or safer replacements
- make custom tooling easy to discover

Do not think of `PATH` as neutral plumbing. It is part of the teaching surface.

### 13.3 Immutable Dependency Files

Where appropriate, make dependency lockfiles or equivalent artifacts immutable until a human explicitly permits change.

Examples include:

- `package-lock.json`
- `requirements.txt`
- language-specific lockfiles
- version pinning manifests

The purpose is not obstruction for its own sake. It is to prevent silent dependency drift during autonomous work.

### 13.4 Role Separation

Where autonomous loops are mature enough, separate roles by capability.

Examples:

- implementation agents write code but not tests
- test agents maintain tests but not implementation
- review agents audit output before promotion

This creates genuine validation pressure instead of self-certification.

Detailed repo policy belongs in `REPO_STANDARDS.md`, but the environment should support the separation technically wherever possible.

### 13.5 Prefer Structural Protection Over Reminder Text

If a dangerous action can be blocked or constrained in the environment, prefer that over relying on an instruction buried in a prompt.

---

## 14. Documentation as Part of the Environment

Documentation is not separate from the environment. It is part of the environment.

### 14.1 Documentation Types That Help Agents

Useful document types include:

| Document type | Environment role |
|---|---|
| `AGENTS.md` / `CLAUDE.md` | Project-level rules and conventions |
| User stories | Intent and acceptance criteria |
| ADRs | Settled decisions and constraints |
| Q&A docs | Pre-answered project questions |
| Anti-requirements | Clear out-of-scope boundaries |
| Constraints docs | Security, performance, platform, and scope limits |
| Skills (`SKILL.md`) | Reusable task capabilities |
| Task lists | Concrete work intake |

### 14.2 Documentation Should Reduce Prompt Burden

The purpose of environment documentation is not decoration.

It should help an agent answer:

- What am I allowed to do?
- What should I use?
- What is already decided?
- What is out of scope?
- What is the expected output shape?
- What must be validated before review?

### 14.3 Use Stable Names and Locations

Keep important docs in predictable places and with predictable names.

Stability improves discoverability for both humans and agents.

---

## 15. Making Environments Sticky

An environment is "sticky" when a capable agent dropped into it naturally does the right thing.

### 15.1 Four Reinforcing Layers

| Mechanism | Framework | Purpose |
|---|---|---|
| Intent-driven scripts | Any | Give specific tasks with clear outcomes |
| Skills | Claude CLI / Codex | Reusable capabilities agents can invoke |
| Rules | `AGENTS.md`, `CLAUDE.md`, project docs | Project-level constraints and conventions |
| Hooks | Agent framework hooks | Automatic reactions to events |

The more these layers align, the less prompting is needed.

### 15.2 Stickiness Comes from Reinforcement

A sticky environment usually has:

- clear entry points
- discoverable tools
- obvious task files
- stable documentation
- meaningful hooks
- constrained action surfaces
- validation feedback
- consistent history and structure

### 15.3 The Goal

The end state is not maximum restriction. It is guided productivity.

A good sticky environment lets the agent do everything it needs to do, while making the intended path more obvious than the unintended one.

---

## 16. Validating the Environment Itself

The environment should be tested just as code is tested.

### 16.1 Treat the Environment as a System Under Test

Environment validation can include:

- prerequisite tests
- path discovery tests
- auth readiness tests
- init script tests
- cross-platform command tests
- hook installation tests
- JSON schema validation tests
- smoke tests for autonomous loops

### 16.2 Use TDD Thinking for the Environment

Before relying on an environment feature, ask:

- How do we know it is present?
- How do we know an agent can discover it?
- How do we know the output shape is stable?
- How do we know it works on the intended surfaces?

### 16.3 Focused Validation Agents Are Useful

A focused agent that tests one narrow property can be highly effective.

Examples:

- "Verify all required commands exist and support the required flags."
- "Verify the init script produces the correct readiness JSON."
- "Verify the preferred tools are discoverable on PATH."
- "Verify the lock files are immutable."

Validation agents should test the environment itself, not just the application code inside it.

---

## 17. Recommended Repository Layout for Environment Engineering

A typical project may expose the environment through a structure like:

```text
/
├── AGENTS.md
├── init-claude.sh
├── init-codex.sh
├── docs/
│   ├── AI_PHILOSOPHY.md
│   ├── AI_OPERATING_MODEL.md
│   ├── ENVIRONMENT_ENGINEERING_GUIDE.md
│   ├── REPO_STANDARDS.md
│   ├── constraints.md
│   ├── anti-requirements.md
│   └── adr/
├── skills/
│   ├── smart-commit/
│   │   └── SKILL.md
│   └── ...
├── hooks/
│   └── ...
├── scripts/
│   ├── bootstrap/
│   ├── readiness/
│   ├── telemetry/
│   └── intents/
├── tasks/
│   └── ...
└── tests/
    └── environment/
```

The exact structure can vary, but the principles should remain stable:

- obvious entry points
- obvious rules
- obvious tasks
- obvious validation
- obvious tooling

---

## 18. Practical Checklists

### 18.1 Environment Bootstrap Checklist

Before declaring an environment ready, verify:

- [ ] Required commands exist
- [ ] Required commands support needed flags or subcommands
- [ ] Required auth is present
- [ ] Preferred tools are on `PATH`
- [ ] Unsafe or unwanted tools are removed, wrapped, or deprioritized as needed
- [ ] Project docs are present in expected locations
- [ ] Skills and hooks are installed or discoverable
- [ ] Lockfiles or protected files are constrained as intended
- [ ] Readiness script returns structured output
- [ ] A human can understand the bootstrap log
- [ ] An agent can parse the bootstrap result

### 18.2 Tool Contract Checklist

For each custom tool, define:

- [ ] input contract
- [ ] output contract
- [ ] exit codes
- [ ] retry safety
- [ ] mutability behavior
- [ ] auth requirements
- [ ] telemetry behavior
- [ ] JSON support where machine consumption is expected

### 18.3 Autonomous Loop Readiness Checklist

Before running unattended loops, verify:

- [ ] environment can be recreated from script
- [ ] key context lives in files, not only conversation
- [ ] validation gates exist
- [ ] failure states are explicit
- [ ] outputs are machine-readable where needed
- [ ] progress is visible
- [ ] dangerous actions are structurally constrained
- [ ] the loop can start fresh without relying on prior chat state

---

## 19. What This Guide Optimizes For

This guide optimizes for environments that are:

- reproducible
- parseable
- observable
- discoverable
- constrained
- testable
- project-specific
- agent-friendly

It does **not** optimize for:

- vague flexibility
- hidden state
- silent automation
- one-off handcrafted sessions
- prompt-only control where structural control is possible

---

## 20. Final Principle

The environment should do as much of the teaching as possible.

A good AI environment makes the right thing:

- easier to discover
- easier to execute
- easier to validate
- harder to misunderstand
- harder to break by accident

When that is true, prompts can become shorter, agents become more reliable, and human attention can move up to direction-setting instead of supervision.

---

## Appendix A: Example Readiness Response

```json
{
  "ready": false,
  "reason": "GitHub CLI is installed but not authenticated",
  "details": {
    "gh_installed": true,
    "gh_authenticated": false,
    "docker_installed": true,
    "docker_running": true,
    "skills_present": true
  },
  "next_action": "run gh auth login and rerun init-claude.sh"
}
```

## Appendix B: Example Telemetry Event

```json
{
  "tool": "init-claude",
  "command": "bootstrap",
  "params": {
    "project": "sample-repo"
  },
  "options": ["--json"],
  "timestamp": "2026-04-03T09:30:00Z",
  "environment": "codespaces",
  "status": "success",
  "duration_ms": 981
}
```
