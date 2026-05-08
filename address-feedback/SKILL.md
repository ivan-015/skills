---
name: address-feedback
description: "Address review feedback on Ivan's own pull request. Reads each unresolved review comment, evaluates it against the code, triages into valid/partial/discuss/stale, implements the right fixes and pushes them, or pushes back with a reasoned reply. Does NOT force-push, push to protected branches, or re-request review — those decisions stay with Ivan."
---

# address-feedback

You are addressing review feedback on Ivan's PR as a senior engineer who can disagree. Make valid fixes, push back on incorrect feedback with reasoning, and keep the PR moving without taking actions that should stay with Ivan.

## Resolve the PR

Find the PR you're addressing in this order:
1. If the caller passed a PR number, use it.
2. If a routine trigger context is available, use `pull_request.number` and `repository.full_name`.
3. Otherwise, run `gh pr view --json number,headRefName` on the current branch.
The PR author must be `ivan-015`. If not, exit silently.

## Collision check

Inspect the latest commit on the PR's head ref. If it was authored by `ivan-015` within the last 30 minutes, do NOT touch code. Post one top-level comment: "Skipping this round — recent activity detected on the branch. Will pick up next trigger." Then exit.

This protects against pushing fixes on top of Ivan's in-progress work in routine mode. In `/loop` mode the cost is one skipped iteration, which is fine.

## Load context

- PR description and full diff (`gh pr diff <num>`).
- All comments and reviews (`gh pr view <num> --json comments,reviews,reviewThreads`).
- For each thread, check whether it already has a reply from Ivan or from this skill. Skip threads that are already addressed.
- For each unresolved comment, open the referenced file and read ~30 lines around the line. The code may have changed since the comment.
- CI status (`gh pr checks <num>`).

## Triage each unresolved comment

Assign exactly one bucket:

- **[valid]** — reviewer is correct, code change needed, you know the right fix.
- **[partial]** — reviewer has a real concern, but the right fix differs from what they implied.
- **[discuss]** — you genuinely disagree, the comment is missing context, or it's a judgment call. No code change.
- **[stale]** — the referenced code has already changed. Acknowledge and resolve.

**Confidence threshold for any code change ([valid] or [partial]):** you must be ≥80% confident that both (a) the issue is real and (b) the fix you have in mind is right. Under 80% → downgrade to [discuss]. False fixes are worse than no fixes — they erode trust in this skill.

## Action by bucket

**[valid]:**
1. Make the smallest change that resolves it. No drive-by refactors.
2. If a test exists for the area, run it. If it fails, the fix is wrong — stop, downgrade to [discuss], and reply asking the reviewer.
3. Commit: `address review: <short description> (re: <thread anchor>)`.
4. Reply on the thread: "Fixed in <sha>: <what changed and why>." Don't resolve — let the reviewer resolve.

**[partial]:**
1. Implement what you believe is right.
2. Commit (same format).
3. Reply: "Read the underlying concern as <X>. Fixed as <Y> rather than <what reviewer suggested> because <reason>. Open to other framings — <sha>." Don't resolve.

**[discuss]:**
1. No code change.
2. Reply with your reasoning and one specific question that would unblock you. Don't hedge.
3. Don't resolve.

**[stale]:**
1. Reply: "Addressed in <sha> — referenced code no longer exists in this form."
2. Resolve the thread.

## Push

After all comments are processed:
- If any commits were made, `git push origin HEAD`.
- Never `--force` or `--force-with-lease`.
- If push fails (non-fast-forward, etc.), stop and post a top-level comment: "Push failed — branch has diverged. Ivan, please rebase." Don't try to recover.
- Never push to `main`, `master`, `develop`, or any branch matching the repo's protected-branch rules. If you somehow find yourself on one, stop.

## Top-level summary

After all threads are processed, post one top-level comment in this shape:

```
**Pass complete.** <N threads processed>

**Addressed:** <thread refs + commit shas, or "none">

**Open for discussion:** <thread refs + one-sentence summaries, or "none">

**Stale (resolved):** <count, or skip the line if zero>

_Pushed: <sha list, or "no code changes this pass">._
```

If nothing was changed and nothing needed a reply, post nothing.

## Hard rules

- Only act on PRs where `pull_request.user.login == "ivan-015"`. Otherwise exit silently.
- Never `git push --force` or `--force-with-lease`.
- Never push to a protected branch.
- Never run `gh pr ready`, `gh pr review`, `gh pr merge`, or any state-changing PR command. Code changes and comment replies are the entire surface area.
- Never re-request review. Ivan does that manually after reviewing the changes.
- Never @-mention anyone other than the comment author you're replying to.
- Never resolve threads except for [stale] cases.
- Don't make changes outside the scope of the comments you're addressing. If you spot an unrelated bug, mention it in the summary; don't fix it.
- Under 80% confidence on a code change → downgrade to [discuss]. Always.
