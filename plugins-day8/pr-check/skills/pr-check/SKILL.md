---
name: pr-check
description: Use when reviewing GitHub pull requests with gh CLI - filters out resolved/existing threads, scopes to incremental commits, creates pending reviews with code suggestions, and gets explicit user approval before posting.
allowed-tools: AskUserQuestion, Bash(gh:*), Bash(git:*)
---

# GitHub PR Review

## Overview

Workflow for reviewing GitHub pull requests using `gh api` to create pending reviews with code suggestions, while ignoring already-reviewed commits and existing/resolved review threads. **Always use pending reviews to batch comments, even under time pressure.**

**CRITICAL: Always get explicit user approval before posting any review comments.** Show exactly what will be posted and ask for yes/no confirmation using `AskUserQuestion`.

## When to Use

- Reviewing pull requests
- Adding code suggestions to PRs
- Posting incremental or full review comments with the `gh` CLI

## Prerequisites & Step 0: Check `gh` CLI

Before starting any PR review workflow, verify the `gh` CLI is available:

```bash
gh --version
```

**If `gh` is not installed:**

1. **Stop immediately** — do not attempt to run `gh api` commands.
2. **Inform the user** with this message:

```text
The GitHub CLI (gh) is required for this skill but is not installed.

Please install it from: https://cli.github.com

Installation options:
- macOS: brew install gh
- Windows: winget install GitHub.cli
- Linux: See https://cli.github.com for your distro

After installing, authenticate with:
  gh auth login

Then try your PR review request again.
```

---

## Core Workflow (Execute Strictly in Order)

### Step 1: Resolve PR Metadata & Existing Review Threads (GraphQL)

Resolve the PR number (use `$ARGUMENTS` if provided, otherwise detect from the current branch via `gh pr view --json number --jq .number`) and repository owner/name (`gh repo view --json owner,name`).

Then run this GraphQL query to fetch the latest commit SHA, prior review commits, and all existing review threads with their resolution status:

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        headRefOid
        baseRefOid
        reviews(last: 15) {
          nodes {
            author { login }
            state
            commit { oid }
            submittedAt
          }
        }
        reviewThreads(first: 100) {
          nodes {
            isResolved
            isOutdated
            path
            line
            comments(first: 5) {
              nodes {
                author { login }
                body
              }
            }
          }
        }
      }
    }
  }' -F owner="{owner}" -F repo="{repo}" -F pr={pr_number}
```

### Step 2: Determine Review Scope (Incremental vs. Full Diff)

Check `reviews.nodes` from Step 1 for the most recent submitted review (`state != "PENDING"`):

1. **Incremental Review (Prior review exists):**

   - Take the `commit.oid` of the most recent review (`last_reviewed_sha`).
   - If `last_reviewed_sha` equals `headRefOid`, stop and inform the user: *"No new commits have been pushed since the last review (`short_sha`)."*
   - Fetch the incremental diff between the two commits via GitHub's compare API (works even if the PR branch is not checked out locally):

     ```bash
     gh api repos/:owner/:repo/compare/{last_reviewed_sha}...{head_ref_oid} \
       -H "Accept: application/vnd.github.v3.diff"
     ```

   - *(Fallback)*: If the compare API returns a 404 (meaning the branch was force-pushed/rebased and `last_reviewed_sha` no longer exists), fall back to the full PR diff below.

2. **Full Review (No prior review or force-pushed branch):**

   ```bash
   gh pr diff {pr_number}
   ```

### Step 3: Deduplication & Filtering Rules

Before drafting any review comment, cross-check every potential finding against `reviewThreads.nodes` from Step 1:

- **Ignore Resolved Threads:** Do **NOT** re-flag any issue on a file/line where `isResolved == true`, unless the new commits explicitly reintroduced the exact bug.
- **Ignore Active Threads:** Do **NOT** post a comment if an open (`isResolved == false`) thread already addresses the same logic or line.
- **Ignore Outdated Threads:** Do **NOT** resurrect threads marked `isOutdated == true` unless the underlying bug still exists in `headRefOid`.
- **Ignore Non-Code & Bot Noise:** Skip lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, etc.), generated files, and automated bot comments (coverage, linters, security scanners).
- **Zero-Issue Exit:** If no **new** bugs, security risks, or edge cases are found in the scoped diff, inform the user (*"LGTM — no new issues found since `short_sha`"*) and ask if they want to submit an `APPROVE` review with no line comments.

### Step 4: Draft & Get Explicit Approval (`AskUserQuestion`)

Before posting **ANY** review, use `AskUserQuestion` to display:

- File path and line number for each new comment
- Exact comment text (including any ```` ```suggestion ```` blocks)
- Event type (`APPROVE` / `REQUEST_CHANGES` / `COMMENT`)
- Overall review summary message

**Approval Prompt Format:**

```text
Question: "Ready to post this review to PR #{pr_number}?"
Header: "PR Review"
Options:
  - Yes, post it: Creates pending review and submits as shown
  - No, let me revise: Lets you edit comments or event type before posting
```

### Step 5: Post Pending Review & Submit (Only After Approval)

**ALWAYS use the pending review pattern, even for a single comment:**

````bash
# 1. Create PENDING review (omit event field) using headRefOid from Step 1
gh api repos/:owner/:repo/pulls/{pr_number}/reviews   -X POST   -f commit_id="{head_ref_oid}"   -f 'comments[][path]=path/to/file.ts'   -F 'comments[][line]={line_number}'   -f 'comments[][side]=RIGHT'   -f 'comments[][body]=Comment text

```suggestion
// suggested code here
```

Additional explanation...'   --jq '{id, state}'
# Returns: {"id": 123456, "state": "PENDING"}

# 2. Submit the pending review
gh api repos/:owner/:repo/pulls/{pr_number}/reviews/{review_id}/events   -X POST   -f event="COMMENT"   -f body="Optional overall review message"
````

*(Note: If submitting an `APPROVE` or `COMMENT` with zero line comments, skip step 1 and post directly to `repos/:owner/:repo/pulls/{pr_number}/reviews` with `-f event="APPROVE"` and `-f body="..."`.)*

---

## Event Types

| Event Type | When to Use | Example Situations |
|------------|-------------|-------------------|
| `APPROVE` | Non-blocking suggestions, PR is ready to merge | Minor style improvements, optional refactoring, or clean incremental pass |
| `REQUEST_CHANGES` | Blocking issues that must be fixed | Security vulnerabilities, bugs, breaking changes, failing tests |
| `COMMENT` | Neutral feedback or questions | Asking for clarification, architectural observations |

## Syntax & Formatting Rules

### Parameters
- `commit_id`: Always use `headRefOid` fetched in Step 1.
- `'comments[][path]'`: File path relative to repo root (must be single-quoted).
- `'comments[][line]'`: End line number in the PR diff (use `-F` for integers).
- `'comments[][start_line]'`: Start line number for multi-line suggestions (use `-F`).
- `'comments[][side]'`: `RIGHT` for added/modified lines, `LEFT` for deleted lines.
- `'comments[][body]'`: Comment text with optional `suggestion` block (use `-f`).

### Code Suggestions Format
Code suggestions replace the entire line or line range—ensure the replacement code is complete and syntactically valid:

````markdown
```suggestion
const fixed = "like this";
```
````

If suggesting edits inside a Markdown file containing triple backticks, wrap the suggestion in 4 backticks (` ````suggestion `) or tildes (` ~~~suggestion `).

## Red Flags — Stop Immediately

- Skipping `gh --version` check first
- Using `gh pr view --json reviewThreads` (invalid CLI field; always use Step 1's GraphQL query)
- Re-flagging issues already present in `reviewThreads` (`isResolved: true` or `false`)
- Posting a review without calling `AskUserQuestion` and receiving explicit confirmation
- Forgetting single quotes around `'comments[][path]'` or mixing up `-f` (strings) and `-F` (numbers)
