# /apply-pr-feedback

Apply review feedback from a **Pull Request** by fetching comments, letting the user choose which to apply, and implementing the accepted ones.

**MANDATORY** Do not commit or push changes. The user must do that after the command finishes.
**MANDATORY** Present all comments to the user before applying any changes.
**MANDATORY** Only apply comments explicitly accepted by the user.

---

# Input

The command accepts a **PR number or URL** (required).

Examples:

```
/pr-feedback 42
```

```
/pr-feedback https://github.com/org/repo/pull/42
```

---

# Agent

### Pedro — Principal Software Engineer

Responsible for applying the accepted review feedback.

---

# Workflow

---

# Step 1 — Fetch PR Comments

Retrieve all review comments from the PR:

```
gh pr view <pr> --json reviews,comments
gh api repos/{owner}/{repo}/pulls/{number}/comments
```

Collect both:

* top-level review comments
* inline code review comments

For each comment extract:

* author
* file path and line (if inline)
* comment body
* whether it's part of a resolved conversation

---

# Step 2 — Present Comments

Present a **numbered list** of all unresolved comments to the user.

Format:

```
# PR #<number> — Review Comments

1. [author] file.ts:42
   "Rename this variable for clarity"

2. [author] general
   "Missing error handling for the timeout case"

3. [author] utils.ts:10
   "This duplicates logic from helpers.ts"
```

If there are **resolved conversations**, list them separately at the end under a `## Resolved (skipped)` section for reference only.

---

# Step 3 — User Selection

Ask the user which comments to apply:

```
Which comments should I apply?

Enter the numbers to apply (e.g. 1,2,5) or "all" to apply everything:
```

All comments **not selected** are considered discarded.

---

# Step 4 — Apply Feedback

Agent: **Pedro**

For each accepted comment, in order:

1. Read the relevant file and understand the context
2. Implement the change requested by the reviewer
3. Keep changes minimal and scoped to what the reviewer asked

Pedro must:

* apply one comment at a time
* not introduce unrelated changes
* not refactor beyond what the comment requests

---

# Step 5 — Verification

After all accepted comments are applied:

1. Run the project's test suite
2. Run lint if available
3. Confirm all tests pass

If any test fails, Pedro must fix the issue before continuing.

---

# Final Output

```
# PR Feedback Summary

## Applied
- #1: <short description of change>
- #3: <short description of change>

## Discarded
- #2: <comment summary>

## Test Results
<pass/fail>
```
