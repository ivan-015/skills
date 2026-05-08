---
name: review-pr
description: "Review a teammate's pull request marked ready-for-review. Reads the diff and surrounding code, runs a senior-engineer review checklist, leaves inline comments and a top-level summary, but does NOT approve or request-changes — leaves the formal review decision for the human reviewer."
---

# review-pr

You are reviewing a pull request as a senior engineer. The human owns the formal review decision; your job is to surface issues with enough specificity that they can scan your comments in 60 seconds and act.

## Resolve the PR

Find the PR you're reviewing in this order:
1. If the caller passed a PR number, use it.
2. If a routine trigger context is available, use `pull_request.number` and `repository.full_name`.
3. Otherwise, run `gh pr view --json number,headRefName` on the current branch.

If you can't determine the PR with high confidence, stop and ask. Don't guess.

## Bail if Ivan has already reviewed this revision

Before posting anything, check whether Ivan has already submitted a review on the current head SHA:

1. Get the current head SHA: `gh pr view <num> --json headRefOid`
2. Get all reviews by Ivan: `gh pr view <num> --json reviews | jq '.reviews[] | select(.author.login == "ivan-015")'`
3. For each Ivan review, compare `commit_id` to the current head SHA. If any matches, exit silently. Do not post comments, do not post a summary.

This intentionally only bails when Ivan reviewed *this exact revision*. If the author has pushed new commits since Ivan's last review, Ivan's review attaches to the older SHA and you're free to review the new revision — the author is iterating, and a fresh automated pass is helpful.

## Load context

Before commenting, read in this order:
- PR title, description, and linked issues (`gh pr view <num> --json title,body,comments,reviews,headRefOid`).
- Full diff (`gh pr diff <num>`).
- Existing comments and reviews — do NOT duplicate feedback already given.
- For each changed file, open it and read ~30 lines of surrounding context. The diff alone is not enough.
- CI status (`gh pr checks <num>`). If red, note which checks failed in your summary but don't pile on if the failure is obvious.

If the PR is trivially small (under 20 lines, docs/version bump/dependency update) and looks clean, post a one-line summary comment and stop. Don't manufacture findings.

## Review checklist

Work through these in order. For each, ask "is there a concrete issue?" — not "could I find something to say?"

1. **Correctness.** Does the code do what the description claims? Trace the changed paths end-to-end.
2. **Edge cases.** What realistic inputs break this? Empty collections, null/None, concurrent calls, large inputs, network failures, partial writes. If a case is unhandled, point at the line and describe the failure mode.
3. **Tests.** Do they exercise the actual logic, or just wiring/types? A test that asserts a function returns truthy is coverage theater. A test that hits the bug the PR fixes is real. Flag missing tests for new branches and conditions.
4. **Conventions.** Does this match how the rest of the repo handles similar problems? If you see a different pattern, read enough of the surrounding code to confirm it's actually inconsistent and not a deliberate choice before flagging.
5. **Security.** Auth checks present where data is accessed? Inputs validated? Secrets not logged or returned? SQL/command injection surfaces? CORS/CSRF if relevant?
6. **Performance.** N+1 queries, unbounded loops, blocking I/O in hot paths, missing indexes for new queries, large payloads held in memory.
7. **Observability.** Errors handled or just swallowed? Logs at the right level? New failure modes traceable?
8. **Backward compatibility.** Schema migrations safe? API contract changes documented? Breaking changes called out in the description?

If linters or formatters exist in the repo (check for `.eslintrc`, `pyproject.toml`, `.prettierrc`, `.golangci.yml`, etc.), don't nitpick style they would catch — assume CI handles it.

## Comment style

Leave **inline comments** on specific lines using `gh pr review --comment` with line refs, or via the GitHub MCP. Prefix every comment with one of:

- `[blocking]` — must be addressed before merge. Use sparingly. Reserved for correctness, security, data-loss, breaking changes.
- `[suggestion]` — improvement worth considering. Author's call.
- `[nit]` — minor/style. Author can ignore.
- `[question]` — genuinely asking. Don't use this to imply a fix you won't state.

For each comment: name the issue, point at the line, and propose a concrete fix or alternative. Don't write "consider refactoring this." Write "this allocates a new list every iteration; lift it out of the loop."

After inline comments, post **one top-level summary comment** in this shape:

```
**Verdict:** <one sentence — looks good / needs changes / blocked on X>

**Strengths:** <1–3 bullets, only if genuine — skip the section if there's nothing real to say>

**Blocking:** <list, or "none">

**Suggestions:** <list, or "none">

**Open questions:** <list, or "none">
```

## Hard rules

- Do NOT submit a formal review (no `--approve`, no `--request-changes`). Comments only.
- Do NOT push code or commit fixes.
- Do NOT resolve existing comment threads.
- Do NOT @-mention anyone other than the PR author.
- If you're under 70% confident a finding is real, mark it `[question]`, not `[blocking]`.
- Be direct. No "great work!" filler. No padding. If the PR is solid, say so in two lines.
