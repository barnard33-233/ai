---
name: code-reviewer
description:
  Use this skill to review code. It focuses on correctness, maintainability,
  and adherence to project standards. Also use this skill when a user provides
  a GitHub PR link (URL containing /pull/{number}) or a GitLab MR link (URL
  containing /-/merge_requests/{number}, works with any GitLab host including
  self-hosted instances) and wants a code review — the skill will fetch the
  remote branch locally and review all changes introduced by the PR/MR.
---

# Code Reviewer

This skill guides the agent in conducting professional and thorough code reviews, both for local changes and for remote GitHub PRs / GitLab MRs.

## Determine Review Mode

First, figure out which mode applies:

- **Remote PR/MR mode**: The user provided a GitHub PR or GitLab MR link. Go to **Workflow A**.
- **Local changes mode**: The user wants to review local uncommitted or staged changes. Go to **Workflow B**.

---

## Workflow A: Review a GitHub PR or GitLab MR

Use this workflow when the user provides a PR/MR URL.

### A1. Parse the Link

Detect the platform by **URL path pattern**, not by hostname (both GitHub and GitLab can be self-hosted):

| Pattern in URL path | Platform |
|---|---|
| `/-/merge_requests/{number}` | GitLab (any host, including self-hosted) |
| `/pull/{number}` | GitHub (any host, including GitHub Enterprise) |

Extract from the URL:
- **host**: e.g. `https://gitlab.example.com`, `https://github.com`
- **project path**: the path segments between host and the MR/PR pattern (e.g. `group/subgroup/repo`)
- **number**: the MR/PR number

### A2. Get Metadata via API

Use `curl` to call the platform's REST API and get the **target/base branch** name. This avoids depending on `gh` or `glab` CLI tools.

**GitLab** (works for any self-hosted instance):
```bash
# URL-encode the project path (replace / with %2F)
PROJECT_PATH_ENCODED=$(echo "{project_path}" | sed 's/\//%2F/g')
curl --silent "https://{host}/api/v4/projects/${PROJECT_PATH_ENCODED}/merge_requests/{number}"
```
Extract `target_branch` and `title` from the JSON response.

If the GitLab instance requires authentication, use a `PRIVATE-TOKEN` header with token provided in environment variable:
```bash
curl --silent --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "https://{host}/api/v4/projects/${PROJECT_PATH_ENCODED}/merge_requests/{number}"
```

**GitHub**:
```bash
curl --silent "https://api.{host}/repos/{owner}/{repo}/pulls/{number}"
# For github.com, the API host is api.github.com
# For GitHub Enterprise, it's typically {host}/api/v3/repos/...
```
Extract `base.ref` and `title` from the JSON response.

**Fallback**: If the API is unreachable (auth issues, network), fall back to the repo's default branch:
```bash
git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
```

### A3. Fetch the Branch Locally

1. **Record the current branch** so you can return to it later:
   ```bash
   ORIGINAL_BRANCH=$(git branch --show-current)
   ```

2. **Stash uncommitted changes if the working tree is dirty**:
   ```bash
   if [ -n "$(git status --porcelain)" ]; then
     git stash push -m "code-review-auto-stash"
   fi
   ```
   Remember whether a stash was created (for restoring later in A6).

3. **Fetch and create a local review branch**:
   - GitLab MR:
     ```bash
     git fetch origin merge-requests/{number}/head:review-mr-{number}
     git checkout review-mr-{number}
     ```
   - GitHub PR:
     ```bash
     git fetch origin pull/{number}/head:review-pr-{number}
     git checkout review-pr-{number}
     ```
   - If the repo is not the same as the PR/MR's source, fetch from the fork remote instead.

3. **Fetch the base branch** to ensure it's up to date:
   ```bash
   git fetch origin {base_branch}
   ```

### A4. Generate the Diff for Review

Get the full diff of everything the PR/MR introduces relative to the base branch:

```bash
git diff origin/{base_branch}...HEAD
```

This shows all changes introduced by the PR/MR — exactly the scope to review.

For large PRs/MRs, also check which files changed to understand the scope:

```bash
git diff origin/{base_branch}...HEAD --stat
```

If the diff is very large, read the changed files directly to get full context rather than relying solely on the diff output.

### A5. Analyze (same pillars as local review)

Proceed to the **In-Depth Analysis** section below. Apply all the same review pillars to the PR/MR diff.

### A6. Clean Up

After the review is complete:

1. **Switch back and restore stash**:
   ```bash
   git checkout {ORIGINAL_BRANCH}
   # If a stash was created in A3, restore it
   git stash pop
   ```

2. **Delete the review branch** (ask the user first):
   ```bash
   git branch -D review-pr-{number}   # or review-mr-{number}
   ```

---

## Workflow B: Review Local Changes

Use this workflow when reviewing uncommitted or staged local changes.

### B1. Preparation

1.  **Identify Changes**:
    *   Check status: `git status`
    *   Read diffs: `git diff` (working tree) and/or `git diff --staged` (staged).

---

## In-Depth Analysis

This section applies to both Workflow A and Workflow B. Analyze the code changes based on the following pillars:

*   **Correctness**: Does the code achieve its stated purpose without bugs or logical errors?
*   **Maintainability**: Is the code clean, well-structured, and easy to understand and modify in the future? Consider factors like code clarity, modularity, and adherence to established design patterns.
*   **Readability**: Is the code well-commented (where necessary) and consistently formatted according to our project's coding style guidelines?
*   **Efficiency**: Are there any obvious performance bottlenecks or resource inefficiencies introduced by the changes?
*   **Security**: Are there any potential security vulnerabilities or insecure coding practices?
*   **Edge Cases and Error Handling**: Does the code appropriately handle edge cases and potential errors?
*   **Testability**: Is the new or modified code adequately covered by tests (even if preflight checks pass)? Suggest additional test cases that would improve coverage or robustness.

## Provide Feedback

#### Structure
*   **Summary**: A high-level overview of the review.
*   **Findings**:
    *   **Critical**: Bugs, security issues, or breaking changes.
    *   **Improvements**: Suggestions for better code quality or performance.
    *   **Nitpicks**: Formatting or minor style issues (optional).
*   **Conclusion**: Clear recommendation (Approved / Request Changes).

#### Tone
*   Be constructive, professional, and friendly.
*   Explain *why* a change is requested.
*   For approvals, acknowledge the specific value of the contribution.
