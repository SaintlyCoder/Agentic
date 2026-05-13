# REPO_STANDARDS.md

**Audience:** Contributors, maintainers, AI agents  
**Purpose:** Define mandatory repository conventions, traceability rules, and integration gates  
**Authority:** Normative  
**Read time:** 8-10 minutes  
**Related docs:** `AI_PHILOSOPHY.md`, `AI_OPERATING_MODEL.md`, `ENVIRONMENT_ENGINEERING_GUIDE.md`, `AGENTS.md`, `smart-commit/SKILL.md`

This document defines the baseline standards for repositories worked on by humans and AI agents.

Repository-local documents may extend these rules, but they **must not weaken** branch protection, traceability, validation, or review requirements. When a repository needs stricter rules, the stricter rule wins.

---

## 1. Normative Language

The keywords **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are used intentionally:

- **MUST / MUST NOT** — mandatory
- **SHOULD / SHOULD NOT** — strong default; deviation requires a clear reason
- **MAY** — optional

---

## 2. Scope and Precedence

These standards apply to all repositories in this operating model.

### 2.1 Precedence Order
1. Security and legal requirements
2. Repository-specific standards and protected branch rules
3. This document
4. Higher-level philosophy and operating-model documents

### 2.2 Extension Rules
- Repository-specific documents **MAY** add stricter requirements.
- Repository-specific documents **MUST NOT** remove structured history, validation, or review expectations without an explicit documented exception.
- Any exception **MUST** be written down in the repository root and include an owner and rationale.

---

## 3. Required Branching Model

### 3.1 Standard Branches
- Every repository **MUST** have a `main` branch.
- Every repository **MUST** have a `development` branch.
- `main` **MUST** be protected.
- Direct pushes to `main` **MUST NOT** be allowed.

### 3.2 Where Work Happens
- Active work **MUST** happen on `development` or on a topic branch created from `development`.
- Topic branches **MUST** be scoped to a single coherent unit of work.
- Work **MUST NOT** begin from stale or ad hoc branches.

### 3.3 Promotion Flow
- Changes **SHOULD** flow from topic branch to `development`.
- Promotion from `development` to `main` **MUST** pass the full quality gate defined in this document.

---

## 4. Branch Naming Standard

### 4.1 Format
Branch names **MUST** follow this format:

```text
<type>/<locator>
```

Examples:

```bash
git checkout development
git checkout -b feat/user-auth-flow
git checkout -b bug/null-ref-on-login
git checkout -b refactor/split-api-handlers
```

### 4.2 Allowed Types
Valid branch prefixes **SHOULD** come from the same type system used for commits. Default types are:

- `feat`
- `fix`
- `bug`
- `refactor`
- `test`
- `docs`
- `chore`
- `ci`
- `perf`
- `ops`
- `design`

Repositories **MAY** narrow or extend this list, but the valid set **MUST** be documented.

### 4.3 Locator Rules
- The `locator` **MUST** identify the subsystem, component, script, or concern being changed.
- The `locator` **SHOULD** be concise, stable, and machine-parsable.
- Kebab-case **SHOULD** be used unless a repository defines a different convention.

### 4.4 Variation Branches
When multiple implementation or planning variants are explored, variation branches **MUST** append a version suffix to the base task branch:

```text
feat/user-auth-flow-v1
feat/user-auth-flow-v2
feat/user-auth-flow-v3
```

If a platform prevents explicit branch naming, the mapping between the platform session and the intended branch name **MUST** be recorded in the task, PR, or issue.

---

## 5. Structured Commit Message Standard

This rule is **non-negotiable**.

### 5.1 Required Format
Every commit message **MUST** follow this format:

```text
type(locator): <description>
```

Examples:

```text
feat(gh-cli): add authentication check before install
fix(system-recon): handle missing git repos in downloads
refactor(orchestrator): split prerequisite checks into stages
docs(philosophy): add repository standards summary
ops(debug-logs): add timestamped log output to scripts
```

### 5.2 Commit Message Rules
- `type` **MUST** come from the repository's approved type system.
- `locator` **MUST** use the repository's defined nomenclature.
- `description` **MUST** be concise and describe the change directly.
- Commit messages **MUST** remain machine-parseable.
- Freeform or vague commit messages **MUST NOT** be used.

### 5.3 Commit Scope
- Each commit **SHOULD** represent one coherent change.
- Mixed-purpose commits **SHOULD NOT** be created when the work can be separated cleanly.
- Commit history **MUST** tell a coherent story that another human or agent can reconstruct later.

### 5.4 Repository-Specific Nomenclature
Each repository **MUST** define its valid commit types and locators in a `smart-commit/SKILL.md` file or equivalent canonical source.

Structured history is part of the environment. Agents are expected to use it as context.

---

## 6. Repository Bootstrap Requirements

Before significant autonomous or multi-agent work begins, the repository **MUST** have the following in place:

- Protected `main`
- A `development` branch
- A defined commit type and locator scheme
- A documented validation path
- A canonical agent instruction file such as `AGENTS.md` or equivalent
- A `smart-commit/SKILL.md` file or equivalent commit guidance

For agent-heavy repositories, the following documents **SHOULD** be present early:

- User stories or task documents with acceptance criteria
- ADRs for settled architectural decisions
- Constraints documents
- Anti-requirements or explicit out-of-scope notes
- Q&A or knowledge documents for recurring project questions

---

## 7. Pull Requests and Integration Gates

### 7.1 Protected Integration
- Nothing **MUST** merge to `main` without passing the repository's quality gate.
- Changes targeting `main` **MUST** go through the repository's pull request process.
- `main` **MUST NOT** be used as a working branch.

### 7.2 Minimum Gate
At minimum, promotion to `main` **MUST** include:

- Passing CI / pipeline execution
- Passing the test suite defined by the repository
- Review by an independent reviewer, agent, or automated review system as defined by the repository

### 7.3 Important Repositories
Repositories designated as important or high-value **SHOULD** enable automated review tooling such as CodeRabbit in addition to CI.

---

## 8. Validation Standards

### 8.1 Test-Driven Default
Test-driven development is the default operating mode.

- For non-trivial implementation work, tests **SHOULD** be defined before autonomous scaling.
- The test suite **MUST** be treated as an objective scoring function for implementation quality.
- Work that fails the declared validation path **MUST NOT** be promoted.

### 8.2 Self-Validation
- Agents **SHOULD** validate their own work before asking for human attention.
- Work surfaced for review **SHOULD** already have passed objective checks where available.
- Human review time **MUST** be reserved for judgement, not routine verification.

---

## 9. Separation of Concerns

### 9.1 Code and Test Role Separation
For autonomous loops and agent-based workflows:

- Code agents **MUST NOT** modify tests when operating in a code-authoring role.
- Test agents **MUST NOT** modify implementation when operating in a test-authoring role.
- A single agent **MUST NOT** both author and validate the same change when an independent review path is available.

### 9.2 Objections and Escalation
- If a code agent believes a test is wrong, it **MUST** raise an explicit objection for separate review.
- It **MUST NOT** silently weaken, rewrite, or remove the test to make its own implementation pass.

This separation exists to prevent the most common autonomous failure mode: changing the test to hide bad code.

---

## 10. Agent Review Requirements

### 10.1 Independent Review
- Agent-generated work **SHOULD** be reviewed by another agent, an automated review system, or a human reviewer before promotion.
- The reviewer **SHOULD** have a focused mandate where possible: security, test coverage, style, commit hygiene, performance, or slop detection.

### 10.2 Rejection Flow
- Reviewers **MAY** reject work and send it back for another iteration.
- Rejected work **MUST NOT** be promoted without passing the next review cycle.

The default model is: build, validate, review, then promote.

---

## 11. Documentation Standards for Agent-Worked Repositories

Repositories intended for autonomous or semi-autonomous agent work **MUST** expose intent clearly in repository-native documents.

### 11.1 Required Intent Signals
Where applicable, repositories **SHOULD** document:

- What to build
- What not to build
- Acceptance criteria
- Constraints and hard boundaries
- Settled architectural decisions
- Repeated questions and their answers

### 11.2 Why This Matters
Agents expand scope unless constraints are explicit. Documentation is not supplementary; it is part of the control surface.

---

## 12. Early-History Discipline

The early commit history of a repository sets the pattern future agents will copy.

- The first phase of repository history **SHOULD** be kept especially clean and consistent.
- A dedicated git steward agent or maintainer **SHOULD** own commit ceremony, branch creation, and merge discipline during the early lifecycle.
- Repositories **SHOULD** front-load commit quality rather than trying to repair history later.

As a working default, the first ~25 commits are treated as pattern-setting commits.

---

## 13. Non-Compliance

When these standards are violated:

- The violation **MUST** be corrected before promotion to `main`, unless an explicit exception has been documented.
- Repeated violations **SHOULD** trigger a review of repository instructions, skills, hooks, or automation.
- If agents repeatedly miss a rule, the environment **SHOULD** be improved so that the rule is enforced structurally rather than conversationally.

---

## 14. Minimal Checklist

Before promoting work, confirm the following:

- Branch created from `development`
- Branch name follows `<type>/<locator>`
- Commit messages follow `type(locator): description`
- Validation path has passed
- Review has occurred through an independent path
- Documentation and constraints remain consistent with the change
- No protected-branch rules have been bypassed

---

## 15. Canonical Principle

Repository standards exist to make work:

- traceable
- parseable
- reviewable
- testable
- promotable

If a change breaks those properties, it is not ready.
