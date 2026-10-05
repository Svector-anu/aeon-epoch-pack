---
name: epoch-build
description: execute exactly one work order in an isolated checkout, verify it locally, and open one pull request. never merges.
metadata:
  title: epoch build
  mode: write
  category: dev
  var: ""
  tags:
    - dev-loop
    - pull-request
  requires:
    - GH_GLOBAL
---

# epoch-build

execute exactly one epoch work order: change code in an isolated checkout, prove it locally against the order's VERIFY line, and open one pull request. one order, one branch, one pr. you never merge.

`${var}` selects the work:

- `order:<path>` - a work order written by `epoch-spec`, normally `memory/topics/<project>/orders/<id>.md`. the usual form.
- `repair:<owner/repo#N>@<40-char-sha>` - the one bounded repair pass, authorised only by a review verdict bound to that exact sha. see **repair**.
- anything else is a free-text instruction against this instance's own repo. treat it as an order whose GOAL is that text, with the rest inferred and stated in your result.

empty `${var}` is a no-op: say so in one line and stop.

today is `${today}`.

## isolation

do all repo work in a throwaway directory outside the workspace, never inside it:

```
work=$(mktemp -d)
echo "$work"
```

shell variables don't survive between tool calls, so print the path once and paste the absolute path into later commands.

a full clone of a big repo can exhaust the shell backend and kill the run, so clone blobless and shallow, with explicit timeouts:

```
timeout 600 git clone --filter=blob:none --depth 1 https://github.com/<owner>/<repo>.git <work>/repo
```

if the clone times out anyway, stop and report it. don't open a pr you couldn't build and verify.

## do

1. **read the order.** for `order:<path>`, read the file and restate its GOAL, SCOPE, ACCEPTANCE, VERIFY and FORBIDDEN in your own words before touching anything. a field you can't fill is a blocker, not a guess: stop and report it. read the project's `map.md` and `handoff.md` next to the order if they exist. if the order's SCOPE and the repo disagree, the repo wins and you report the drift.

2. **clone into the throwaway directory** and create one branch named `epoch/<order-id>`. never work on the default branch. never force-push. never rebase.

3. **make the smallest change the order asks for.** stay inside SCOPE and respect FORBIDDEN literally. work the repo already has is not yours to refactor: a change outside the order is a follow-up, recorded in your result, not a commit. if the order turns out to need work it doesn't name, do the part it names and report the rest.

4. **verify locally before pushing.** run the order's VERIFY commands in the foreground, wrapped in `timeout N`, and paste the verbatim output into your result. a failing VERIFY means no pr: fix it, or stop and report the failure with its output. never open a pr whose own stated verification you haven't run.

5. **commit and push.** one commit unless the order asks for more. message: `<type>(<scope>): <what changed and why>`, lowercase, under 72 characters, no trailing period. push the branch.

6. **open one pull request** with `gh pr create`. the body states: what changed and why, the order path, each ACCEPTANCE line and whether it's met, the verbatim VERIFY output, and anything you deliberately left out. keep it factual. don't claim a check you didn't run.

7. **leave a pointer for the next stage.** write `memory/skills/epoch-build/pull-request.json` with exactly:

   ```json
   { "url": "https://github.com/<owner>/<repo>/pull/<N>", "head_sha": "<40-char-lowercase-sha>" }
   ```

   take the sha from the pr itself after pushing, never from local state that may have moved:

   ```
   gh pr view <N> --repo <owner>/<repo> --json headRefOid -q .headRefOid
   ```

   it's a pointer, not evidence. `epoch-review` and `epoch-watch` read it when their `${var}` is empty, and only to learn which pr and which sha. write it once, last, and only after the pr exists.

## repair

`repair:<owner/repo#N>@<sha>` is the single bounded repair pass. fail closed unless all of these hold: the pr is still open, its current head sha equals `<sha>` exactly, and a review verdict bound to that same target and sha exists and is actionable (`memory/skills/epoch-review/verdict.json` says `"actionable": true`, or the pr carries an `aeon-review` receipt with critical or issues above zero). then check out the pr's existing head branch, address only the findings that verdict names, run VERIFY, and push to the same branch. no new branch, no second pr. rewrite the pointer file with the new head sha. if anything is ambiguous, change nothing and report the blocker. one pass only: if review is still actionable afterwards, that is a terminal result for this order, not a reason to try again.

## do not

- merge, approve or close any pull request. merging is the operator's call.
- open more than one pr per run.
- touch a repo the order doesn't name.
- write outside `memory/skills/epoch-build/` and `memory/logs/`, and the throwaway directory.
- run anything in the background. every command runs in the foreground; wrap long ones in `timeout N`.
- report success for work whose VERIFY you didn't run and paste.

## result

your final message is the result. state: the order id and its GOAL, the branch, the pr url and head sha, each ACCEPTANCE line and whether it's met, the verbatim VERIFY output, what you deliberately left out, and any follow-up worth its own order. if you opened no pr, say exactly why in the first line.

append a `### epoch-build` entry to `memory/logs/${today}.md` naming the order id, the pr and the head sha.
