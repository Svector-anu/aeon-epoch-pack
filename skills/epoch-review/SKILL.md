---
name: epoch-review
description: review one pull request against its work order at one pinned head sha, and post a single receipt-bearing github review that the dev-loop gate accepts. never changes or merges the pr.
metadata:
  title: epoch review
  mode: write
  category: dev
  var: ""
  tags:
    - dev-loop
    - code-review
  requires:
    - GH_GLOBAL
---

# epoch-review

review one pull request against the work order it claims to satisfy, at one exact head sha, and post a receipt-bearing github review that the dev-loop review gate will accept. you judge the diff. you never change it, never merge it, and never read how it was built.

`${var}` selects the target:

- `<owner/repo#N>` - review that pr at its current head.
- `<owner/repo#N>@<40-char-sha>` - review it only if its head still equals that sha.
- empty - fall back to `memory/skills/epoch-build/pull-request.json` and take **only** its `url` and `head_sha`. that file is a pointer, not evidence.

today is `${today}`.

## independence

your verdict is worth nothing if it inherits the builder's reasoning. don't read `output/epoch-build/`, any builder result card, the builder's run record, or its log entries. read the order, the diff and the repo. if you've already seen the builder's rationale, say so in your result and mark the review `discussion-needed` rather than pretend to an independence you don't have.

if build and review ran on the same model family, say that in your result. a fresh context with no access to the builder's reasoning is the independence you do have, so don't overclaim it.

## do

1. **pin the sha first, from github, not from local state.**

   before anything else, refuse to review a pull request whose state isn't OPEN. a verdict on a landed pr gates nothing, and the head-sha check alone won't catch it: a pr can merge seconds after opening, before any review runs.

   ```
   gh pr view <N> --repo <owner>/<repo> --json state -q .state
   ```

   if it isn't OPEN (merged or closed), stop without posting. report the state and exit.

   ```
   gh api "repos/<owner>/<repo>/pulls/<N>" --jq .head.sha
   ```

   if `${var}` supplied a sha and this differs, stop: the pr moved, and a review of the old head would be a lie. report the mismatch and exit without posting. every later step uses this one pinned value.

2. **get the diff without cloning.** read the change through the api:

   ```
   gh pr diff <N> --repo <owner>/<repo>
   gh api "repos/<owner>/<repo>/pulls/<N>/files" --jq '.[] | "\(.status) \(.filename) +\(.additions)/-\(.deletions)"'
   gh api -H "Accept: application/vnd.github.raw+json" "repos/<owner>/<repo>/contents/<path>?ref=<sha>"
   ```

   that's enough to review a diff and read any file at the pinned sha. clone only if step 5 needs to execute something, and then blobless and shallow, into a throwaway directory (print the path once and paste it; shell variables don't survive between tool calls):

   ```
   work=$(mktemp -d); echo "$work"
   timeout 600 git clone --filter=blob:none --no-checkout --depth 1 https://github.com/<owner>/<repo>.git <work>/repo
   timeout 300 git -C <work>/repo fetch --depth 1 origin <sha>
   git -C <work>/repo checkout <sha>
   ```

   if a clone times out anyway, review from the diff alone and record that you couldn't execute the pr's verification. that's a `discussion-needed` at worst, never a made-up pass.

3. **read the order the pr claims to satisfy.** the pr body names it; otherwise look under `memory/topics/*/orders/`. restate its ACCEPTANCE lines. a pr whose order you can't find is reviewable only against the repo's own standards, and you say so.

4. **review the diff** from step 2's `gh pr diff` output. judge only what changed. every finding names a file and a line and states the concrete failure, not a preference. classify each:

   - **critical** - merging this causes a real defect: wrong behaviour, data loss, a broken build, a security hole, or a claim in the pr body that the diff doesn't support.
   - **issue** - actionable and worth fixing before merge, but not a defect that breaks something.

   style, naming and taste aren't findings. neither is work the order forbade. if the diff does something outside SCOPE, that's critical.

5. **check the pr's own claims.** if the body pastes verification output, re-run those commands at this sha and compare. a claim you can't reproduce is a critical finding. this is the highest-value thing you do: a diff that looks right and a claim that is false are different problems.

6. **decide the verdict**, consistent with the counts. the gate enforces this and rejects a mismatch:

   | verdict | requires |
   |---|---|
   | `approve-ready` | `critical == 0` and `issues == 0` |
   | `discussion-needed` | `critical == 0` and `issues > 0` |
   | `blocked` | `critical > 0` |

7. **check no receipt already exists at this sha, then post exactly one github review** carrying it. the gate requires exactly one, so a second voids both, including one that another skill or an earlier run already posted, which leaves the pr ungateable with two agreeing verdicts. count it the way the gate does (this account's reviews at this sha, a missing body treated as empty, the literal receipt prefix), and if the count isn't zero, stop without posting and report the existing receipt:

   ```
   actor=$(gh api user --jq .login)
   gh api repos/<owner>/<repo>/pulls/<N>/reviews \
     | jq --arg actor "$actor" --arg sha "<sha>" \
       '[.[] | select(.user.login == $actor and .commit_id == $sha) | .body // empty | select(contains("<!-- aeon-review:"))] | length'
   ```

   use `--comment`, never `--approve`: when this account authored the pr, github refuses a self-approval, and the verdict rides in the receipt rather than in github's approval state.

   ```
   gh pr review <N> --repo <owner>/<repo> --comment --body-file <file>
   ```

   the body states each finding with its file and line, what you re-ran and its verbatim output, and ends with the receipt on its own line, exactly one per review:

   ```
   <!-- aeon-review:{"schema":1,"target":"<owner>/<repo>#<N>","sha":"<sha>","verdict":"<verdict>","critical":<n>,"issues":<n>} -->
   ```

   keys must be exactly `critical`, `issues`, `schema`, `sha`, `target`, `verdict`. no extra keys, no missing keys, no second marker anywhere in the body.

8. **write the verdict** to `memory/skills/epoch-review/verdict.json`. this write is unconditional: it happens whether or not the next step's self-check passes. the durable local record doesn't depend on that check.

   ```json
   { "target": "<owner>/<repo>#<N>", "sha": "<sha>", "verdict": "<verdict>", "critical": 0, "issues": 0, "actionable": false }
   ```

   `actionable` is `critical > 0 || issues > 0`. it's what a repair pass reads to know it's authorised.

9. **validate your own receipt with the gate** and paste the verbatim output:

   ```
   bash scripts/dev-loop-review.sh verify <owner>/<repo>#<N> <sha>
   ```

   a failure here means the posted review isn't admissible as a gate-passing receipt. it doesn't mean the verdict from step 8 is wrong or should be retracted. fix the receipt and post a corrected review only if the first one was malformed; otherwise report the failure and stop. never post two receipt-bearing reviews for the same sha, because the gate requires exactly one and rejects both.

## do not

- modify the pr, push to its branch, merge it, close it or approve it.
- post more than one receipt-bearing review per sha.
- review a sha other than the pinned one.
- invent findings to look rigorous, or suppress one to look agreeable. a clean diff gets `approve-ready`.
- report a verdict whose counts contradict it.
- run anything in the background. foreground only, `timeout N` for long commands.

## result

state: the target and pinned sha, the verdict and both counts, each finding with file and line, what you re-ran and whether it reproduced, the verbatim gate output from step 9, and whether a different model family was involved. if you posted no review, the first line says why.

append a `### epoch-review` entry to `memory/logs/${today}.md` with the target, sha and verdict.
