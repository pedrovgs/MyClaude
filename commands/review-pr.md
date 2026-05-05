# /review-pr

Review a Pull Request using **three specialized agents in parallel**, producing actionable suggestions with file/line references and code change proposals.

**MANDATORY** Every suggestion must point at a specific `file:line`.
**MANDATORY** Keep suggestions short and to the point — what to change and why.
**MANDATORY** Suggest code changes in markdown code blocks when possible.
**MANDATORY** Flag bugs or dangerous changes with a red flag.

---

# Input

The command accepts a **PR number or URL** (required).

Examples:

```
/review-pr 42
```

```
/review-pr https://github.com/org/repo/pull/42
```

---

# Agents

### Pedro — Code Reviewer

Reviews code quality, correctness, and maintainability.

### Natalia — Manual Testing Evaluator

Identifies use cases that need manual testing.

### Raúl — Architecture Reviewer

Evaluates architectural decisions and structural patterns.

---

# Workflow

---

# Step 1 — Fetch PR Data

Retrieve all PR information:

```
gh pr view <pr> --json title,body,baseRefName,headRefName,files
gh pr diff <pr>
```

Collect:

* PR title and description
* Base and head branches
* Full diff
* List of changed files

---

# Step 2 — Parallel Review

Run **all three agents in parallel**. Each agent receives the full PR diff, changed files, and PR description.

---

## Pedro — Code Review

Agent: **Pedro**

Review the PR for:

* bugs or incorrect logic
* error handling gaps
* naming and readability issues
* code duplication
* unnecessary complexity
* missing edge case handling
* security concerns

For each finding, produce a suggestion following the output format below.

**Flag bugs or dangerous changes with a red flag.**

---

## Natalia — Manual Testing Evaluation

Agent: **Natalia**

Evaluate the PR for:

* use cases that should be tested manually
* risky areas not covered by automated tests
* edge cases that are hard to automate
* UI/UX flows that need human verification
* regression risks in related features

For each finding, produce a suggestion following the output format below.

---

## Raúl — Architecture Review

Agent: **Raúl**

Evaluate the PR for:

* alignment with existing architecture and patterns
* coupling and cohesion issues
* abstraction quality — too much or too little
* violation of SOLID principles
* impact on maintainability and extensibility
* missing or misplaced boundaries

For each finding, produce a suggestion following the output format below.

**Flag structural violations that could cause long-term damage with a red flag.**

---

# Suggestion Format

Every suggestion from every agent **must** follow this format:

```
### [flag] file:line — Short title

Explanation of what to change and why (1-2 sentences max).

Suggested change:

\`\`\`language
// proposed code
\`\`\`
```

Where `[flag]` is:

* Leave blank for regular suggestions
* Use 🚩 for bugs or dangerous changes

If no code change applies (e.g. manual test scenarios), replace the code block with a description of the action.

---

# Step 3 — Aggregate Results

Combine all suggestions into a single report:

```
# PR Review — #<number>

## Summary

- Pedro: X suggestions (Y red flags)
- Natalia: X suggestions
- Raúl: X suggestions (Y red flags)

---

## Code Review (Pedro)

<suggestions>

---

## Manual Testing (Natalia)

<suggestions>

---

## Architecture Review (Raúl)

<suggestions>
```
