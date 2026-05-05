# /pr

Create a **Pull Request** for the current or specified branch using a structured preparation workflow.

**MANDATORY** Do not commit or push changes. The user must do that before running this command.
**MANDATORY** When a repository PR template exists, use it exactly as defined.
**MANDATORY** Do not create the PR until the user explicitly confirms.

---

# Input

The command accepts an **optional branch name**. If omitted, uses the current branch.

Examples:

```
/pr
```

```
/pr cpf/fix-PROJ-123
```

```
/pr feature/search-caching
```

---

# Agent

### Yago — Release Engineer

Responsible for preparing and creating the Pull Request.

---

# Workflow

---

# Step 1 — Determine Branch

If a branch name was provided as input, use it.

Otherwise, detect the current branch:

```
git branch --show-current
```

Verify the branch has commits ahead of the base branch:

```
git log main..HEAD --oneline
```

If no commits ahead, **stop and inform the user**.

---

# Step 2 — Gather Context

Collect information for the PR:

1. Full commit history since diverging from base branch:

```
git log main..HEAD
```

2. Full diff against base branch:

```
git diff main...HEAD
```

3. List of modified files:

```
git diff main...HEAD --name-only
```

---

# Step 3 — Locate PR Template

Search for a repository Pull Request template:

```
.github/PULL_REQUEST_TEMPLATE.md
.github/pull_request_template.md
docs/pull_request_template.md
PULL_REQUEST_TEMPLATE.md
```

If a template is found, use it exactly as defined and populate it with the gathered context.

If no template exists, use this structure:

```
## Summary

## Changes

## Test plan
```

---

# Step 4 — Present PR Draft

Present the complete PR proposal to the user:

Branch name

Base branch

PR title

PR body (filled using the repo template if found)

---

# Step 5 — User Confirmation

Ask the user:

```
Do you want me to create the Pull Request using this information? (yes/no)
```

If **yes** → create the PR.

If **no** → allow modifications and present again.
