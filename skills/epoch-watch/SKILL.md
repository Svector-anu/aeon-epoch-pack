---
name: epoch-watch
description: decide whether one pull request is merge-ready, from github's own state plus the review and proof receipts at the exact current head. reports only, never merges.
metadata:
  title: epoch watch
  mode: write
  category: dev
  var: ""
  tags:
    - dev-loop
    - merge-readiness
  requires:
    - GH_GLOBAL?
---

# epoch-watch

decide whether one pull request is merge-ready, from github's own state plus the review and proof receipts at the exact current head. you report readiness. you never merge, never push, never approve.

`${var}` is `<owner/repo#N>`. empty means read `memory/skills/epoch-build/pull-request.json` and take its `url`. today is `${today}`.

merge-ready means four things are true at the same head sha. if any one is false it's not ready, and you name which.

1. the pr is open and not a draft
2. github is willing to merge it
3. a review receipt exists at this head, verdict not `blocked`
4. a proof receipt exists at this head

## do

1. **read the state once, and pin the head sha from it.** everything after uses this sha. if it changes mid-run, start over rather than mixing two heads.

   ```
   gh pr view <N> --repo <owner>/<repo> --json number,state,isDraft,headRefOid,mergeable,mergeStateStatus,statusCheckRollup
   ```

2. **assess whether github will merge it**, from `mergeStateStatus` and the head's check rollup. `BLOCKED` with a rollup of `FAILURE` or `ERROR` is a refusal: checks are genuinely red. `BLOCKED` with any other rollup is allowed on the rollup's basis, because `BLOCKED` alone also covers a required review the repo may not use. anything other than `BLOCKED` is allowed on the merge-state basis. `DIRTY` means conflicts and `BEHIND` means the base moved: both are not-ready, with that reason named.

   name the failing checks when the rollup is red. a red check is the most common reason a pr isn't ready, and naming it saves the next reader a click.

3. **check the review receipt** with the dev-loop gate:

   ```
   bash scripts/dev-loop-review.sh verify <owner>/<repo>#<N> <sha>
   ```

   success prints a normalized receipt carrying `verdict` and `actionable`. a `blocked` verdict is not ready. `discussion-needed` is not ready while `actionable` is true. absence is not ready: say "no review receipt at this head", not "review failed".

4. **check the proof receipt** the same way:

   ```
   bash scripts/dev-loop-proof.sh verify <owner>/<repo>#<N> <sha>
   ```

   absence means the behaviour was never demonstrated at this head. that's not ready, and it isn't a defect in the pr.

5. **write the verdict** to `memory/skills/epoch-watch/readiness.json`:

   ```json
   {
     "target": "<owner>/<repo>#<N>",
     "sha": "<sha>",
     "ready": false,
     "github": "allowed|refused|dirty|behind|closed|draft",
     "review": "approve-ready|discussion-needed|blocked|absent",
     "proof": "proven|absent",
     "blocking": ["the reasons, shortest first"],
     "checked_at": "<rfc3339>"
   }
   ```

   `ready` is true only when `github` is `allowed`, `review` is `approve-ready` and `proof` is `proven`. write `blocking` as an empty list when ready.

6. **regenerate the handoff** for the project this pr belongs to, if there is one. find it by the order the pr body names, or by matching the branch to `memory/topics/*/orders/`. if there's no project, skip this and say so. rewrite `memory/topics/<project>/handoff.md` whole from what is true now on disk and on github: the readiness verdict above, the pr and its head sha under active branches, the verification status per pr and sha, and a next action naming the skill and var that would unblock it. never append, never narrate events into it. a section you can't derive stays empty.

   the test is unchanged: a fresh run reading only that file knows what to do next.

## do not

- merge, approve, close, reopen or push anything.
- re-run a gate to get a different answer.
- treat green ci as proof, or a passing review as behaviour having been demonstrated. they're separate conditions, and the whole point of this skill is that all of them hold at one sha.
- report ready when a receipt is merely absent. absent is not ready.
- run anything in the background. foreground, `timeout N` on anything slow.

## result

one line first: `READY` or `NOT READY: <shortest blocking reason>`. then the target and sha, the four conditions with their values, the verbatim gate output for review and proof, and any failing check by name. if the operator has to act, say exactly what.

append a `### epoch-watch` entry to `memory/logs/${today}.md` with the target, sha and readiness.
