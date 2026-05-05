---
name: review-pr-team
description: "Review a pull request using a team of parallel specialized agents, each focusing on a different review concern (architecture, concurrency safety, testing, performance, PR quality + Jira alignment). Aggregates findings into a single scannable review. Use when the user says 'team review', 'review PR with team', 'agent team review', 'deep review', or wants a multi-perspective PR review. Also trigger when the user says 'review PR #N with agents' or 'thorough PR review'."
---

# PR Review Team

You orchestrate a **team of specialized review agents** that analyze a PR in parallel, then you aggregate their findings into a single, high-quality review.

## Why a team?

A single-pass review tends to go deep on one dimension and shallow on others. By running focused agents in parallel, each one can read the full context and reason deeply within its domain, producing more thorough coverage in less wall-clock time.

## Step 1: Gather Context (you do this — agents inherit it)

Before spawning agents, collect all the context they'll need. This avoids redundant work across agents.

1. **Identify the PR** — from user input (number, URL, or current branch). If none given, use `gh pr view --json number`.
2. **Get PR metadata**:
   ```bash
   gh pr view <number> --json body,title,commits,files,additions,deletions,baseRefName,headRefName,author
   ```
3. **Get the full diff**:
   ```bash
   gh pr diff <number>
   ```
4. **Read ALL changed files in full** — use the Read tool on every file in the diff. Do NOT skip files. Agents need the complete context, not just the diff hunks.
5. **Read neighboring files** — for each changed file, read its imports, the protocols it conforms to, and the callers. This is critical for catching issues at boundaries.
6. **Get CI status and existing reviews**:
   ```bash
   gh pr checks <number>
   gh pr view <number> --json comments,reviews
   ```
7. **Fetch Jira ticket** (if identifiable from branch name or PR description):
   ```bash
   uv run ~/.claude/skills/jira-communication/scripts/core/jira-issue.py --json get <TICKET-KEY>
   ```

At the end of this step you should have:
- PR title, body, author, file list, additions/deletions count
- The full diff text
- The full content of every changed file and key neighboring files
- CI status and reviewer comments
- Jira ticket details (if available)

## Step 2: Spawn Review Agents in Parallel

Launch **all 5 agents in a single message** using the Agent tool. Each agent receives the full context you gathered and a focused review mandate.

### Agent 1: Architecture & Design

```
You are a senior software architect reviewing PR #<N> for architectural quality.

**Your focus areas:**
- Duplication: code repeating patterns already present in changed files or neighbors
- Overengineering: abstractions with one implementation, premature generics, unnecessary patterns
- Readability: naming clarity, function length, comment quality
- Scope: is the PR doing one thing well, or bundling unrelated changes?

**Context:**
<paste: PR metadata, diff, changed files, neighboring files>

**Output format:**
Return a structured list of findings. Each finding must have:
- Category: [DUP], [OE], [RD], or [SCOPE]
- Severity: critical / suggestion / positive
- Location: file:line
- Description: what and why (2-3 sentences)
- Suggestion: how to fix (if applicable)
- **For critical-severity findings ONLY:** include a "Current code:" block with the exact broken code and a "Fix:" block with the corrected, copy-pasteable replacement. Include enough context (5-15 lines) so the author can locate it.

If a dimension has no issues, say so in one line. Lead with the most impactful findings.
End with a 1-sentence overall architectural assessment.
```

### Agent 2: Concurrency & Safety

```
You are a Swift concurrency expert reviewing PR #<N> for thread safety and correctness.

**Your focus areas:**
- `@unchecked Sendable` on classes with mutable state — is the safety invariant documented and sound?
- `nonisolated(unsafe)` on stored properties — is the type actually safe to share?
- Locks/mutexes held across `await` suspension points (deadlock/serialization risk)
- TaskGroup under serial global actors (false parallelism)
- Silent error swallowing in async pipelines — `catch { return default }` without logging
- Data races: shared mutable state without synchronization
- Actor isolation: correct use of actors and global actors, proper actor boundaries
- Cancellation: is `Task.isCancelled` / `checkCancellation()` checked appropriately?

**Context:**
<paste: PR metadata, diff, changed files, neighboring files>

**Output format:**
Return a structured list of findings. Each finding must have:
- Category: [CONCURRENCY], [SENDABLE], [CANCELLATION], [DATA-RACE], or [ERROR-HANDLING]
- Severity: critical / warning / info
- Location: file:line
- Description: what the issue is and why it's dangerous (2-3 sentences)
- Suggestion: how to fix
- **For critical-severity findings ONLY:** include a "Current code:" block with the exact broken code and a "Fix:" block with the corrected, copy-pasteable replacement. Include enough context (5-15 lines) so the author can locate it.

This is the highest-priority review dimension. Be thorough. Read beyond the diff boundary to verify safety invariants hold.
End with a 1-sentence safety assessment.
```

### Agent 3: Testing Quality

```
You are a testing specialist reviewing PR #<N> for test coverage and quality.

**Your focus areas:**
- Are new/changed code paths tested?
- Do tests verify behavior (not implementation details)?
- Edge cases: nil inputs, empty collections, cancellation, error paths
- Test naming: does each name describe scenario + expected outcome?
- Mocks: are they minimal? Only mock external dependencies, prefer real objects when cheap
- Async test patterns: proper concurrency, no `sleep()` or timing-dependent assertions
- Missing test scenarios: what could break that isn't tested?

**Context:**
<paste: PR metadata, diff, changed files, test files>

**Output format:**
Return a structured list of findings. Each finding must have:
- Category: [COVERAGE], [QUALITY], [EDGE-CASE], [MOCK], or [ASYNC-TEST]
- Severity: critical / suggestion / positive
- Location: file:line (or "missing test for file:function")
- Description: what's missing or wrong and why it matters
- Suggestion: specific test scenario to add (if applicable)
- **For critical-severity findings ONLY:** include a concrete example test in a code block showing what the test should look like (mock setup, act, assert). Use the project's existing test patterns (XCTest, mocks via protocol injection). The test must be copy-pasteable with minimal adaptation.

End with: "Coverage assessment: X/Y key scenarios tested" and list what's missing.
```

### Agent 4: Performance & Scalability

```
You are a performance engineer reviewing PR #<N> for efficiency and scalability.

**Your focus areas:**
- O(n^2) or worse algorithms hiding in loops
- Unnecessary allocations in hot paths (arrays/dicts created in loops)
- Retain cycles in closures
- Main thread work that should be offloaded (and vice versa for UI)
- Redundant computation that could be cached/memoized
- Unnecessary AnyPublisher type erasure, excessive value-type copying
- Missing pagination/batching for growable operations
- Hardcoded limits that will break at scale
- Memory: large objects retained longer than needed

**Context:**
<paste: PR metadata, diff, changed files>

**Output format:**
Return a structured list of findings. Each finding must have:
- Category: [PERF], [MEMORY], [SCALABILITY]
- Severity: critical / warning / info
- Location: file:line
- Description: what the issue is and the expected impact
- Suggestion: how to fix
- **For critical-severity findings ONLY:** include a "Current code:" block with the exact broken code and a "Fix:" block with the corrected, copy-pasteable replacement. Include enough context (5-15 lines) so the author can locate it.

Only flag issues that matter given the context. Don't micro-optimize setup code.
End with a 1-sentence performance assessment.
```

### Agent 5: PR Quality & Jira Alignment

```
You are a project management reviewer checking PR #<N> for process quality and requirement alignment.

**Your focus areas:**

PR Description:
- Does it explain WHY this change exists?
- Is the scope focused (one concern per PR)?
- Is there a test plan?
- Are related issues/tickets linked?
- Do commit messages tell a coherent story?

Jira Alignment (if ticket available):
- Does the PR address the core problem in the ticket?
- Scope match: doing more or less than the ticket asks?
- Issue type consistency: bug fix for bug ticket, feature for story, etc.
- Acceptance criteria: which are met, partially met, or missing?

**Context:**
<paste: PR metadata, commits, Jira ticket details>

**Output format:**
Return findings as:
- Category: [PR-DESC], [SCOPE], [JIRA-ALIGN], [ACCEPTANCE]
- Severity: critical / suggestion / info
- Description: what's missing or misaligned

End with: "Ticket alignment: <aligned / partially aligned / misaligned / no ticket found>"
```

## Step 3: Aggregate into Unified Review

Once all agents return, synthesize their findings into a single review document.

### Aggregation rules:

1. **Deduplicate** — if two agents flag the same issue (e.g., architecture and concurrency both flag a lock pattern), keep the more detailed finding and credit both perspectives.
2. **Prioritize** — order findings by severity: critical > warning/suggestion > info/positive.
3. **Cap nits** — include at most 3 low-severity findings. If there are more, mention "N additional minor suggestions omitted."
4. **Highlight positives** — if agents noted good engineering decisions, include 1-2 in the summary.
5. **Include code fixes for every critical finding** — for each critical-severity finding, provide a concrete code snippet showing both the current (broken) code and the proposed fix. This is the most actionable part of the review and saves the author from having to figure out the fix themselves. See the output format below for the expected structure.

### Output format:

Write the review as a markdown file saved to the working directory as `pr-review-team-<PR_NUMBER>.md`.

```markdown
# Team PR Review: <PR title>

**PR:** #<number>
**Author:** <author>
**Files changed:** <count> | **Additions:** +<n> | **Deletions:** -<n>
**Review agents:** Architecture, Concurrency, Testing, Performance, PR Quality

## Summary

<3-4 sentences: overall assessment, key strengths, and top concerns. Be direct.>

## Findings

### 🔴 Critical

<For each critical finding, use the following structure:>

---

#### N. 🔴 [CATEGORY] Short title describing the issue

`file_path:line_range`

<2-3 sentence description of the problem and why it matters.>

**Current code:**
```swift
<Copy the exact current code from the PR that has the issue.
Include enough surrounding context (5-15 lines) so the author
can locate it immediately.>
```

**Fix:**
```swift
<Show the corrected version of the same code block.
Must be a complete, copy-pasteable replacement — not pseudocode.
Include brief inline comments only where the fix is non-obvious.>
```

> **Note:** <Optional 1-line note explaining trade-offs or edge cases the author should be aware of.>

---

### 🟠 Warnings

<Findings with warning severity — real issues that should be addressed but aren't immediate blockers. Each with [CATEGORY], file:line, description, and suggestion. No code fix required but welcome.>

### 🟡 Suggestions

<Findings that would improve the PR but aren't blockers. Same format as warnings but lower priority.>

### 🟢 Positive

<1-2 things done well, especially restraint or good design choices.>

## Agent Assessments

| Agent | Risk | Summary |
|-------|------|---------|
| Architecture | <🟢 / 🟡 / 🟠 / 🔴> | <1-line summary> |
| Concurrency | <🟢 / 🟡 / 🟠 / 🔴> | <1-line summary> |
| Testing | <🟢 / 🟡 / 🟠 / 🔴> | <1-line summary> |
| Performance | <🟢 / 🟡 / 🟠 / 🔴> | <1-line summary> |
| PR Quality | <🟢 / 🟡 / 🟠 / 🔴> | <1-line summary> |

Risk levels: 🟢 pass — 🟡 minor suggestions — 🟠 warnings to address — 🔴 critical issues blocking merge

## Verdict

**<🟢 Approve | 🟡 Approve with suggestions | 🟠 Approve with warnings | 🔴 Request changes>**

<1-2 sentence justification>

---
*Generated by PR Review Team (5 parallel agents)*
```

## Step 4: Present to User

After saving the file:
1. Show the user the verdict and a brief summary
2. Ask if they'd like to post it as a GitHub PR comment (do NOT post automatically)
3. If they say yes, post with `gh pr comment <number> --body-file pr-review-team-<PR_NUMBER>.md`

## Philosophy

- **Critical findings only block**: the verdict is "Request changes" only if there are critical-severity issues (correctness, safety, data loss). Everything else is "Approve with suggestions."
- **No false positives over no false negatives**: it's worse to cry wolf on 10 non-issues than to miss 1 real bug. Each agent should err on the side of fewer, higher-confidence findings.
- **Respect the author**: frame suggestions as questions when the right answer isn't obvious. Reserve strong language for genuine safety issues.
