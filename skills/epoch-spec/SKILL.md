---
name: epoch-spec
description: turn one intent into a repo map, an optional spec and one runnable work order per unit. plans and records, never writes product code.
metadata:
  title: epoch spec
  mode: write
  category: dev
  var: ""
  tags:
    - dev-loop
    - planning
---

# epoch-spec

the front of the epoch loop. turn one intent into durable project knowledge plus exactly one runnable next unit. this skill plans and records. it never writes product code and never opens a pr.

`${var}` is the intent: a sentence on what to build, fix or understand, optionally prefixed `project:<slug>` to bind it to an existing project. empty `${var}` is a no-op: say so in one line and stop.

today is `${today}`.

## where things live

plain files, no new store:

```
memory/topics/<project>/map.md          durable repo knowledge
memory/topics/<project>/spec.md         what success means (feature and project only)
memory/topics/<project>/orders/<id>.md  one work order per unit
memory/topics/<project>/handoff.md      the resumable state of the project
```

`<project>` is a kebab-case slug. reuse an existing one whenever the intent touches the same area. a second slug for the same area splits the knowledge, and that is the main way this skill goes wrong.

## do

1. **read before writing.** `STRATEGY.md`, `memory/MEMORY.md`, and `memory/me.md` if they exist. list `memory/topics` with the read tool, then read any existing `map.md`, `spec.md` and `handoff.md` for a project whose slug plausibly covers the intent. a directory listing is discovery, not reading. don't guess filenames.

2. **classify the intent** into exactly one class, and name the class and the reason in your result.

   | class | when | spec? | units |
   |---|---|---|---|
   | `task` | one mechanical change, no behaviour question | no | 1 |
   | `fix` | something is wrong; a repro is the deliverable before the fix | no | 1 |
   | `feature` | new or changed behaviour a user would notice | yes | 1-5 |
   | `project` | several features, or work that outlives one session | yes | 1-5 now, more later |

   when two classes fit, take the smaller. a `task` that turns out to need a spec is found out during the work, not predicted here.

3. **write or refresh the map** at `memory/topics/<project>/map.md`. it stops the next agent from rediscovering the repo. derive every line from files you read this run, and mark anything inferred as inferred. sections in this order, each present even if short:

   ```
   # map: <project>

   **what it is** - one paragraph a stranger could act on.
   **architecture** - the real shape, named by file and directory.
   **relevant files** - path, one line each on what it owns.
   **dependencies** - internal and external, and what breaks without each.
   **existing behaviour** - what works today, as observable behaviour.
   **constraints** - invariants and anything a change must not break.
   **implementation status** - shipped / partial / planned, per piece.
   **known gaps** - what is missing or wrong, with evidence.
   **verification requirements** - how a change here is proven. real commands.
   **related decisions** - links to prior records, issues, prose.
   **related prs and commits** - with shas where known.
   **remaining work** - what is not done.

   repository: <owner>/<repo> @ <sha>
   ```

   the footer is the freshness key (`git rev-parse HEAD`). a map whose footer sha matches HEAD may be updated in place. a stale one gets its changed sections rewritten, not appended to.

4. **write the spec** at `memory/topics/<project>/spec.md`, for `feature` and `project` only. skip it for `task` and `fix`, and say so in your result. a spec that exists to satisfy ceremony costs more than it earns. contents: the problem, who it is for, the observable outcome that means success, explicit non-goals, acceptance criteria as checkable lines, and the verification commands.

5. **write one work order per unit** at `memory/topics/<project>/orders/<id>.md`, `<id>` a kebab-case slug. size the order to the unit: a one-command unit gets a short paragraph that still names goal, scope, verify and report. an order must be small enough for one `epoch-build` run to finish. this is not style: an order that wires several scripts into a new skill plus tests burns a whole run budget and ends with no commit, while a single-file order finishes in a handful of calls and opens a pr. an intent that needs a whole subsystem is a project to split into several orders, not one unit to hand to one build run. fields:

   ```
   GOAL        one sentence, executable by someone with no access to this run
   SCOPE       paths this unit may write; paths it may not
   CONTEXT     pointers into the map, plus anything an executing agent cannot see from the repo
   ACCEPTANCE  checkable criteria, one per line
   VERIFY      exact commands, plus known gotchas
   FORBIDDEN   out-of-scope changes, and anything this unit must not touch
   REPORT      what the executing run must state back
   ```

6. **regenerate the handoff** at `memory/topics/<project>/handoff.md`, last, from what is on disk now. rewrite the whole file every run. never append and never narrate events into it. every line must be derivable from the map, the orders and the repo; if you can't derive it, leave the section empty rather than guess.

   ```
   # handoff: <project>

   **objective** - the outcome this project exists to reach.
   **phase** - which stage is live: spec | build | review | prove | watch | ship.
   **orders** - every order id with its status: queued | in progress | done.
   **done** - completed units, with pr numbers.
   **in progress** - units started and where they stopped.
   **blocked** - what is stuck, on what, and who can unstick it.
   **decisions** - decided, and why, one line each.
   **discoveries** - what was learned that isn't obvious from the repo.
   **active branches and prs** - with head shas where known.
   **verification status** - per pr and sha, what has been proven.
   **next action** - the single next thing, naming the skill and its var, e.g. epoch-build with order:memory/topics/<project>/orders/<id>.md
   **open questions** - unresolved, each with the default if nobody answers.
   **repro commands** - worth keeping, copy-pasteable.

   generated <timestamp>.
   ```

   the test this file must pass: a fresh run that reads only this file knows what to do next without asking the operator to explain anything.

## do not

- write product code, create a branch or open a pr. that is `epoch-build`.
- write outside `memory/topics/<project>/` and `memory/logs/`.
- invent a second state store, or a second `<project>` slug for an area that already has one.
- run anything in the background. foreground only.

## result

your final message is the result. state: the class and why, the project slug, whether a spec was written or skipped, the map's path and whether it was created or refreshed, each work order id, the next-action line from the handoff, and anything you couldn't determine. if `${var}` was empty, say so in one line and stop.

append a `### epoch-spec` entry to `memory/logs/${today}.md` naming the project, the class and the order ids.
