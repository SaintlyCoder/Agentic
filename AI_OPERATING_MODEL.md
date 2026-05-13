# AI Operating Model

> This document defines how we turn ideas into autonomous, validated engineering work. It describes the workflow, roles, checkpoints, and feedback loops that govern execution. It is the bridge between our philosophy and our implementation standards.

**Audience:** Tech leads, architects, senior ICs, operators  
**Purpose:** Explain how work moves from idea to autonomous execution  
**Authority level:** Operational  
**Read time:** 10-15 minutes  
**Related documents:** `AI_PHILOSOPHY.md`, `ENVIRONMENT_ENGINEERING_GUIDE.md`, `REPO_STANDARDS.md`, project-level `AGENTS.md`

---

## 1. What This Document Covers

This is the document for **how we work**.

It explains:
- how ideas become executable work
- how agents and humans divide responsibilities
- how we decide when work is ready to scale
- how validation happens before human attention is consumed
- how lessons from execution feed back into the system

It does **not** define low-level shell commands, tool installation steps, branch naming syntax, or repository-specific rules. Those belong in implementation and standards documents.

A useful test is simple:
- if the question is **"Why do we believe this?"**, that belongs in philosophy
- if the question is **"How does work move through the system?"**, it belongs here
- if the question is **"How do I implement this technically?"**, that belongs in the engineering guide
- if the question is **"What is mandatory?"**, that belongs in standards

---

## 2. Core Operating Assumptions

Our operating model rests on a small number of assumptions.

### 2.1 Work is refined before it is scaled

We do not treat the first plan as the plan. Before meaningful execution begins, we deliberately generate variation, compare results, and improve the plan. The aim is to discover blind spots while change is still cheap.

A weak plan scaled aggressively produces expensive noise. A refined plan reduces surprises, raises the quality floor, and improves every later loop.

### 2.2 The environment is part of the workflow

Agents do not work in a vacuum. The environment, documentation, available tools, constraints, and validation layers are all part of the operating model.

We do not think of setup as a prelude to the work. In many cases, **setup is the work**. A well-prepared environment reduces prompt burden, increases consistency, and makes autonomous execution safer.

### 2.3 Human attention is the scarce resource

Code generation is no longer the bottleneck. Review capacity, judgement, and decision-making are.

The system should therefore be designed so that humans are asked for:
- direction
- prioritisation
- trade-off decisions
- exception handling
- final judgement where judgement is genuinely required

Humans should not be used as routine verifiers when tests, review agents, and environment controls can do that first.

### 2.4 Validation must be separated from production

A single agent should not be the only judge of its own work. Production and validation are separate concerns.

The default pattern is:
1. a builder produces work
2. the work is checked against objective tests or criteria
3. a separate reviewing agent audits the result
4. only then does the work earn human attention or promotion

This is how we reduce slop, increase reliability, and create tighter autonomous loops.

### 2.5 Fresh loops are healthier than stale loops

During repeated autonomous execution, each iteration should begin from a clean enough context that it can stand on its own. The more an agent depends on accumulated conversational state, the more fragile the loop becomes.

During early bootstrapping, shared continuity can be useful. During steady-state execution, freshness is preferred.

### 2.6 Progress must be observable

Autonomous work should never feel like a black box. At every stage, the system should make it clear what it is doing, what it found, what gate it is waiting at, and why work progressed or stopped.

Silent systems drain trust. Observable systems earn it.

---

## 3. Roles in the System

The operating model works by separating roles clearly.

| Role | Primary Responsibility | Typical Questions It Answers |
|---|---|---|
| **Human architect / sponsor** | Sets direction, defines value, approves meaningful trade-offs | What matters? What is in scope? What level of risk is acceptable? |
| **Operator** | Prepares and governs the environment, coordinates loops, enforces gates | Is the system ready to run? Are roles separated correctly? Are we operating safely? |
| **Builder agent** | Produces implementation or documentation within the declared scope | How do we complete this task inside the given constraints? |
| **Reviewer / auditor agent** | Critiques output, checks quality, rejects weak work | Does this hold up? What is missing? What should be sent back? |
| **Git steward** (especially early in a project) | Maintains clean history, branch discipline, and promotion flow | How should this work be packaged, named, and promoted? |

In stricter environments, builder roles may be split further. For example:
- a **code agent** may change implementation but not tests
- a **test agent** may change tests but not implementation
- a **review agent** may reject or score work but not author the final change

This separation matters because it prevents the most common autonomous failure mode: a system quietly changing the criteria in order to make weak work appear acceptable.

### Default Working Shape

A common default setup is:
- one persistent **operator session** governing the environment and workflow
- one or more **builder sessions** working inside the codebase or task surface
- one **reviewing session** used before promotion

The exact tooling may change by project. The role separation should not.

---

## 4. The Standard Lifecycle

This model moves work through six phases.

### 4.1 Phase 1: Capture Intent

Every cycle starts by turning a vague idea into a usable working brief.

Inputs may include:
- conversation notes
- voice transcripts
- tickets or issue descriptions
- prior plans
- existing repository context
- external critique or structured feedback

The output of this phase is an **intent packet**: a compact set of materials that define what success looks like and what must not happen.

A good intent packet usually includes:
- a concise statement of the job to be done
- user stories or task statements
- acceptance criteria
- constraints
- anti-requirements or out-of-scope notes
- any known architectural decisions or assumptions

The goal is not perfect specification. The goal is enough clarity to begin critique and variation.

### 4.2 Phase 2: Expand Through Variation

Once intent is captured, we avoid overcommitting to the first framing. We generate multiple variants of the plan or approach, then compare them.

This phase exists to surface:
- blind spots
- missing constraints
- assumptions that were never stated
- areas of convergence across variants
- useful disagreement between variants

### Default Variation Policy

Our default operating policy is tiered:
- **Minimum:** for any non-trivial task, generate at least **3 meaningful variations**
- **Expanded mode:** for high-leverage planning work, architecture, or reusable systems, go much wider — often **10, 20, or more** variations
- **Refinement cadence:** before executing at scale, run **3 rounds of critique and improvement** where the work justifies it

The purpose is not variety for its own sake. The purpose is to improve the plan before cost compounds.

The output of this phase is a **refined execution plan**, not a pile of drafts.

### 4.3 Phase 3: Prepare the Environment

Once the plan is strong enough, the next question is not "Can an agent start coding?" It is: **"Is the environment ready to govern the work?"**

At this stage we shape the execution surface so that the agent can discover the right path from the environment itself.

This includes:
- putting the right documentation in place
- making constraints visible
- preparing the right tools and interfaces
- establishing permissions and boundaries
- deciding which roles are separated
- defining the gates that work must pass before promotion

A key principle here is that the environment should carry as much instruction as possible. The better the environment, the lighter the prompt can be.

This phase ends with an **environment readiness decision**:
- if the environment is ready, execution can begin
- if it is not ready, we continue preparing the system rather than pretending implementation has started

### 4.4 Phase 4: Execute in Controlled Loops

When plan and environment are both ready, execution begins in bounded cycles.

The operator remains responsible for the shape of the system. Builder agents work within that shape.

Execution may happen:
- sequentially for tightly coupled work
- in parallel for bounded variations or independent task slices
- across multiple isolated workspaces when comparison or throughput matters

The operating preference is for **short, validated loops** rather than long, opaque runs.

A healthy execution loop:
1. takes a bounded task or task slice
2. runs inside a controlled environment
3. produces a result that can be checked quickly
4. either passes forward, fails fast, or returns for another iteration

### Context Strategy During Execution

We use two different context patterns depending on the stage of work:

- **Bootstrap pattern:** while building the environment or resolving foundational uncertainty, continuity is useful; the operator may carry forward knowledge across iterations
- **Loop pattern:** once execution becomes repetitive, each run should be able to start fresh and re-orient from the environment and documentation

The target state is an environment where a fresh agent can enter, discover the rules, and do useful work without a long conversational preamble.

### 4.5 Phase 5: Validate Before Human Review

Validation exists to protect human attention.

Before work is surfaced to a person, it should first pass through automated checks and independent review.

The default order is:
1. **self-validation** by the producing agent where possible
2. **objective checks** such as tests, acceptance criteria, and measurable validations
3. **independent agent review** focused on defects, omissions, slop, risk, or non-adherence
4. **human review** only for work that has earned it

This creates a double gate:
- an **objective gate** through tests and measurable criteria
- a **critical gate** through review by another agent

If the work fails, it does not get waved through because effort has already been spent. It gets rejected, refined, or re-queued.

### 4.6 Phase 6: Integrate and Learn

Passing work is promoted. Failing work is mined for lessons.

Integration is not the end of the process. It is one step in a longer feedback loop.

Every meaningful run should leave behind one or more of the following:
- better tests
- better constraints
- sharper anti-requirements
- clearer documentation
- stronger environment defaults
- more robust review criteria

The system improves when execution leaves the environment better than it found it.

---

## 5. Decision Gates

A good operating model is shaped by gates, not optimism.

| Gate | Core Question | Pass Result | Fail Result |
|---|---|---|---|
| **Intent gate** | Is the task defined clearly enough to critique and vary? | Move into variation | Clarify the brief |
| **Variation gate** | Have we learned enough from alternatives to avoid scaling a weak first draft? | Freeze a refined plan | Generate more critique or alternatives |
| **Environment gate** | Can the environment guide and constrain execution reliably? | Start controlled execution | Keep preparing the environment |
| **Execution gate** | Is work progressing in bounded, observable loops? | Continue or scale | Reduce scope, split tasks, or intervene |
| **Validation gate** | Has the output passed objective checks and independent review? | Promote for integration or human judgement | Reject, revise, or re-queue |
| **Integration gate** | Does the result deserve promotion into the mainline flow? | Integrate and capture lessons | Hold back and feed findings into the next iteration |

These gates should be visible. A person or agent should always be able to answer: **what gate are we at, and what evidence gets us through it?**

---

## 6. Core Working Artifacts

The operating model depends on structured artifacts that reduce ambiguity and support fresh-context work.

| Artifact | Role in the Operating Model |
|---|---|
| **Intent brief** | States the job to be done in a compact form |
| **User stories / task statements** | Translate intent into actionable units |
| **Acceptance criteria** | Define what success looks like objectively |
| **ADRs** | Record decisions so agents do not reopen settled questions |
| **Q&A docs** | Pre-answer recurring questions that slow down execution |
| **Anti-requirements** | Make clear what must not be built or changed |
| **Constraints docs** | Declare boundaries around technology, scope, security, performance, or process |
| **Task lists / queues** | Provide the next actionable surface for agents |
| **Review scorecards** | Turn critique into a repeatable evaluation pattern |

These artifacts matter because they let a new agent orient itself from the environment instead of depending on a human to restate context each time.

---

## 7. Autonomy Levels

Not all work should be autonomous at the same level. We move through stages.

| Mode | Human Involvement | Best Used For | Exit Condition |
|---|---|---|---|
| **Interactive** | High | Early setup, ambiguity reduction, environment shaping, initial architecture | The task is bounded and the environment can carry more of the instruction |
| **Semi-autonomous** | Medium | Early delivery loops, monitored execution, proving that gates work | The system can progress for multiple iterations without active supervision |
| **Fully autonomous** | Low | Stable repeated loops, overnight execution, parallel bounded workloads | Work is safe to run unattended because constraints and gates are trusted |

The progression should normally be:
1. interact long enough to shape the environment and define the work
2. prove that the system can make progress with limited intervention
3. move the work out of the human's immediate inbox as soon as the system is trustworthy enough

A common failure is staying interactive for too long. Once the environment is good enough, continued manual supervision becomes a tax on attention.

---

## 8. Review and Escalation Model

Autonomy does not mean agents decide everything. It means routine execution is automated while judgement is escalated only when necessary.

### Agents should usually decide
- how to complete a bounded task within declared constraints
- how to iterate when objective checks fail
- how to gather context from the environment and docs
- how to compare bounded alternatives and return a recommendation

### Humans should usually decide
- meaningful scope changes
- architectural trade-offs with long-term consequences
- trust-boundary changes
- exceptions to declared constraints
- prioritisation across projects or workstreams
- whether the system itself needs to be restructured

### Escalate when
- the documentation conflicts with itself
- passing the tests still leaves a real judgement call
- the requested change crosses a trust or risk boundary
- the work requires relaxing a previously declared constraint
- the environment no longer gives reliable guidance

The goal is not to remove human judgement. The goal is to reserve it for work that actually requires judgement.

---

## 9. Operating Cadence

Our default cadence for meaningful work is:

1. capture the intent
2. generate multiple variants
3. critique and refine
4. prepare the environment
5. execute in bounded loops
6. validate through objective checks and independent review
7. integrate what passes
8. feed lessons back into the system

A few default habits follow from this:
- do not scale the first draft of a consequential plan
- do not begin autonomous execution in an unprepared environment
- do not let a single agent both build and validate without an independent check
- do not spend human attention where a reliable gate can decide first
- do not treat failures as waste when they can improve the next loop

---

## 10. What a Healthy System Looks Like

A healthy implementation of this operating model has recognizable properties.

- A fresh agent can enter the environment and orient quickly.
- Work is shaped by documentation, constraints, and visible task surfaces rather than long custom prompts.
- Human attention is spent on direction and judgement, not routine verification.
- The system can run repeated loops without accumulating confusion.
- Poor work is rejected early rather than polished late.
- Review is adversarial enough to be useful, but cheap enough to run frequently.
- Lessons from execution show up in better environments, better docs, and better gates.
- The path from idea to autonomous execution gets shorter over time.

That last point matters. The goal is not just to deliver today's task. The goal is to build a system that becomes easier to trust with tomorrow's work.

---

## 11. Relationship to the Other Documents

This document should sit in the middle of the documentation stack.

- **`AI_PHILOSOPHY.md`** explains the worldview and the reasons behind this model.
- **`ENVIRONMENT_ENGINEERING_GUIDE.md`** explains how to technically build the environments, scripts, tool surfaces, and controls that support this model.
- **`REPO_STANDARDS.md`** defines the non-negotiable rules for branches, commits, quality gates, documentation norms, and repository hygiene.
- **Project-level docs** such as `AGENTS.md`, task files, and constraints docs apply this model to a specific codebase.

The philosophy says **why**.  
This document says **how work moves**.  
The engineering guide says **how the system is built**.  
The standards document says **what must be followed**.

---

## 12. Summary

We operate by refining work before scaling it, preparing environments before trusting them, separating production from validation, and protecting human attention with layered gates.

The operating model is simple to state:

> define the work, vary the plan, prepare the environment, execute in controlled loops, validate aggressively, integrate selectively, and feed every lesson back into the system.

That is how ideas become autonomous execution without surrendering quality or control.
