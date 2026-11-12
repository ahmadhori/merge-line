---
name: merge-line
description: Bring a list or a range of open pull requests into the main branch with the user, one at a time, code PRs first. AI helpers gather each PR's state, review it ahead of time, bring it up to date, explain it on one page and check what is live; the user only says go, merge and deploy. Shows the merge order, gives each deploy group as one command, keeps owner steps and follow-ups on GitHub, and stops the line when something breaks. Use when the user gives PR numbers and asks to review, merge, land or bring them in, asks to postpone, park or resume a PR, or types /merge-line.
argument-hint: "<PR numbers or ranges, e.g. 12 15 20-25> | postpone <N> [reason] | planning <N> [reason] | resume <N>"
---

# /merge-line

Bring open pull requests into the main branch with the user, **one at a time**.
AI helpers do the reading; the user makes three calls: **go**, **merge**,
**deploy**.

```
/merge-line 12 15            single PRs
/merge-line 20-35            a range: every open PR in it
/merge-line 20-35 41         both
/merge-line                  lists the open PRs and asks which
```

## The one rule

**The user merges every PR and runs every deploy.** This session never merges,
deploys, force-pushes, rebases, applies a migration or touches production,
unless the user says so for that one PR.

## Before the first turn: the project profile

Read `.claude/merge-line.md` in the repository if it exists (the template is in
this plugin's `templates/merge-line.md`). It says:

| Section | What it gives | Without a profile |
|---|---|---|
| Branch | the branch PRs merge into | the repo's default branch (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`) |
| Tests | how to run the tests of given files, and the whole suite | look for the project's own test command (package.json, Makefile, pyproject); ask once if unsure |
| Generated files | which files are regenerated, never hand-merged, and the command for each | lockfiles are regenerated with their package manager |
| Version | a bump command each PR runs before it is pushed, or none | none |
| Deploy groups | which paths belong to which deploy, its one command and its live checks | "none": a merge is the end of the line |
| Migrations | the folder whose files make a PR "migration first" | none |
| Review | how many PRs are reviewed ahead (default 4) and with what | `pr-reviewer`, 4 ahead |

Every command below uses the profile's branch where it says `<main>`.

## Who does what

- **This session stays light.** It talks with the user, keeps the order,
  starts helpers and checks their results with commands whose output is a line
  or two (`gh … --jq`). It never reads a diff, a long description, test output
  or a log itself, so its context lasts a whole run.
- **Helpers** (this plugin's `agents/`) do everything that needs reading.
  Installed as a plugin, their names carry its prefix (`merge-line:pr-state`);
  copied by hand into `.claude/agents/`, they are plain (`pr-state`).

| Helper | Does | When |
|---|---|---|
| `pr-state` | one table row per PR | at the start, and for a PR new to the list |
| `pr-reviewer` | reviews one PR, fixes what it is sure of (one commit per finding), pushes | ahead of the merge point (section 2) |
| `pr-updater` | brings one PR up to date with `<main>`, reruns its tests, pushes; or stops on what only the user can decide | at "next" |
| `pr-explainer` | one PR's explanation, in a fixed shape | ahead, once its review has reported; else at "next" |
| `live-check` | the checks a merged PR names, once it is deployed | after each deploy |

- Helpers that don't depend on each other run at once: `pr-state` for up to 10
  PRs per call, one PR's updater beside the next PR's explainer.
- **A helper's report is a report, not proof.** Before saying a PR is ready,
  check it yourself with the ready check (section 4).

### Owner steps: what only the user can do

Anything besides the merge and the deploy command: a secret in a host's
settings, a setting in a web console, DNS, a one-time command. A PR lists them
under `## Owner's steps` as checkboxes, each with when it is due:

```markdown
## Owner's steps
- [ ] **Before the deploy · hosting console:** add `PAYMENTS_KEY`
- [ ] **After the deploy · feature flags:** turn on `new-checkout`
```

While a box is open the PR carries the label `owner-steps`. The boxes in
merged PRs are the only record of what is still to do, so it outlives this
session:

- **Open steps**, one call:
  `gh pr list --state merged --label owner-steps --limit 100 --json number,body --jq '.[] | .number as $n | (([.body | capture("(?s)## Owner.s steps[^\n]*\n(?<s>.*?)(\n## |$)")][0].s // "") | split("\n") | map(select(test("^\\s*[-*] \\[ \\] ")))) as $open | if ($open | length) == 0 then "\($n) none open" else ($open[] | "\($n) \(sub("^\\s*[-*] \\[ \\] "; ""))") end'`.
  For a "none open" line, remove the label.
- **Tick a step** only when the user says it is done: read the body, change
  that one box, and write the body back only if it was read and changed
  (a failed read must never write an empty description).
- **Add a step** a description forgot, on the user's yes, and add the label.
- Create the label once: `gh label create owner-steps --color FBCA04 --description "The owner still has steps to do"`.

### Follow-ups: work nobody has started yet

Every piece of follow-up work this run turns up (a gap a review found, an idea
for later) becomes a GitHub issue labelled `follow-up`, so it outlives the run.

- First look for an open one for the same work:
  `gh issue list --label follow-up --state open --limit 100 --json number,title --jq 'map("#\(.number) \(.title)") | join(" · ")'`.
  When there is one, comment on it instead of opening a second.
- Else `gh issue create --label follow-up --title "…" --body-file <file>`
  (write the body to a file: backticks in a shell string would run).
- The PR that does the work says `Closes #<N>`.
- Create the label once: `gh label create follow-up --color 0E8A16 --description "Proposed work nobody started yet"`.

### Set a PR aside

Two labels take a PR out of a range until the user takes it back:
`postponed` (waits on a release, another PR, the user's time) and `planning`
(a plan still being worked out).

- `/merge-line postpone <N> [reason]`: `gh pr edit <N> --add-label postponed`,
  and with a reason `gh pr comment <N> --body "Postponed: <reason>"`.
- `/merge-line planning <N> [reason]`: the same with `planning`.
- `/merge-line resume <N>`: `gh pr edit <N> --remove-label postponed --remove-label planning`.

Only the user sets or removes these labels.

## 1. The first turn

1. List the PRs in one call per range:
   `gh pr list --state all --limit 1000 --json number,state,labels --jq '.[] | select(.number >= <from> and .number <= <to>) | "\(.number) \(.state) \([.labels[].name] | join(","))"'`.
   A range takes the **open** PRs, skipping the set-aside ones and the PRs
   stacked on them. A single number that is not open is said so, not skipped.
2. `pr-state` for the open PRs, up to 10 per call, all at once.
3. While they run: the open owner steps, the open follow-ups, and whether
   another merge line is running (below).
4. End the turn with one line: what is running.

When `pr-state` has reported, show:

- **Open steps**: each open box of a merged PR, `<N> · <when> · <where>: <what>`.
  A "Before the deploy" step of a PR not deployed yet comes before its deploy;
  any other open step is overdue: say so.
- **Follow-ups**: `#<N> <title>`, joined by " · ".
- **The table** (section 3).
- **Skipped**: `merged 18 · closed 19 · postponed 22 (waits on the release) · planning 30`.

Then one question: **start the reviews of the first 4 code PRs?**

**One merge line at a time.** Every merge makes every other open PR stale. At
the first turn and at each "next", check for merges this run did not make:
`git fetch -q origin <main>; git log origin/<main> --first-parent --merges --since=30.minutes --format=%s | sed -nE 's/^Merge pull request #([0-9]+) .*/\1/p' | paste -sd ' ' -`.
When one appears, say in one line that another line is merging. Stopping it is
the user's call.

## 2. Reviews, just ahead of the merge point

Reviews run **4 ahead** of the merge point (the profile's window), not all at
once: each review then sees a recent `<main>`, and when the user stops halfway
the rest never run.

- The user's go starts the reviews of the first 4 code PRs in the order.
- **Top up in every turn the user's message began** (go, "merged", "next", a
  question): count the code PRs after the merge point whose review is running
  or has reported. While fewer than 4, start the next one's review. A turn
  starts only the gap.
- Each review is the `pr-reviewer` helper, in the background, in its own
  worktree, given the PR number, its head branch, its base and its latest
  commit. If the profile names another reviewer (such as `/code-review <N> --fix`
  in a separate session), use that.
- A PR counts as reviewed when its branch has a commit whose message starts
  with `review-pr-<N>`, or the reviewer reported "nothing to fix".
- **Docs-only PRs** are not code-reviewed. A PR that changes an instruction
  file (`CLAUDE.md`, `AGENTS.md`, anything under `.claude/`) is code: those
  files steer every session.
- **Stacked PRs** (based on another PR's branch): review only the parent;
  the child's review starts once the parent has pushed, and merges the
  parent's branch before `<main>`.

## 3. The table

One table, in the order the user should review and merge:

| Order | PR | Type | Area | What it does | Reviewed | Deploy | Owner steps | State |
|---|---|---|---|---|---|---|---|---|

- **Type**: code, docs, design or planning.
- **Reviewed**: yes, review running, in line, or no review (docs).
- **Deploy**: the PR's deploy group and what it takes ("api: migration, deploy",
  "site: publishes on merge", "none").
- **Owner steps**: where and when each open step is done, or "-".
- **State**: base, stacked or not, mergeable, checks, age when over 14 days,
  and what it waits on from the user.

**Order**: code PRs first, one at a time: what another PR needs comes first (a
stacked parent before its child; a PR that must be live before another one
merges, in an earlier deploy group); then by deploy group, so one deploy covers
the group; then by number. Docs, design and planning PRs come after, with what
each needs from the user (a read, a pick).

## 4. At "next": one PR

In the user's "next" turn, in this order:

1. **Refresh the list** with the first turn's call: a new open PR in the range
   joins (`pr-state` for it), one set aside since leaves, a resumed one comes
   back. One line per change.
2. **Retarget a stacked PR whose parent has merged** first:
   `gh pr edit <N> --base <main>`. A PR still based on its parent's branch
   merges into that branch and never reaches `<main>`.
3. Start `pr-updater` for the PR, and `pr-explainer` unless its explanation is
   ready or on its way.
4. **Explain the next PR ahead**: when the next code PR's review has reported,
   start its `pr-explainer` now, so its explanation waits ready.
5. Top up the reviews (section 2).
6. End the turn with one line: what is running.

Show the explanation unchanged. When the updater did more than merge `<main>`
(resolved a conflict, changed code or a test), say so in one line under it.

**The ready check**, once the updater has reported, one call each:
- `gh pr view <N> --json state,baseRefName,mergeable,headRefOid,statusCheckRollup --jq '"\(.state) base=\(.baseRefName) mergeable=\(.mergeable) head=\(.headRefOid[0:9]) checks=\([.statusCheckRollup[]? | .conclusion // .status] | group_by(.) | map("\(.[0]):\(length)") | join(","))"'`
- `git fetch -q origin && git merge-base --is-ancestor origin/<main> origin/<head> && echo "holds <main>"`

It is ready when it is open, based on `<main>`, mergeable, holds `<main>` (a
docs-only PR need not), and no required check failed. A check still running:
say so with the question; the user merges once it is green.

Then end with the ready line, the updater's tests, and **one question: merge
it?** When the PR has an open "Before the merge" step, list it right above the
question, and the question becomes whether it is done.

When the updater stopped on something only the user can decide, say what, say
that nothing was pushed, and ask.

## 5. After each merge

- **The stop rule**: a failed build, a failed live check, or a test that a
  merge broke on `<main>` **pauses the line**. Update no further PR until the
  break is fixed or reverted. Two PRs that each passed can fail together. The
  fix is a new PR that goes first in the order; a revert is the user's.
- When the merged PR is the last of its deploy group, give the group's deploy
  (section 6).
- A merged PR with an open owner step keeps the label `owner-steps`.

## 6. Deploy a group: one command

When the last PR of a deploy group has merged, give the whole deploy as **one
command** for the user to paste, built from the profile:

1. **Before the deploy**: the open "Before the deploy" boxes of the group's
   PRs. The user does these first.
2. **The command**: the profile's pull, migrations (when a PR is "migration
   first") and deploy command, joined with `&&` so a failed step stops the
   rest. It starts by checking the branch and pulling:
   `[ "$(git branch --show-current)" = <main> ] && git pull --ff-only origin <main> && …`
3. **After the deploy**: the open "After the deploy" boxes.

A group that publishes itself on merge (a static site, say) has no command:
its "Before the deploy" steps come before its merge.

When the command stops, ask for its last lines and name the step that failed.
After a deploy that went through: `live-check` (section 7).

## 7. Check it live

After the user's deploy, run `live-check` with the group's PRs and each one's
"Once it's live" line (a page that should now show a text, a version endpoint,
a health URL). It is read-only. A failed check stops the line (section 5).

## 8. When the user stops

Show what has merged but is not deployed, its command, every open owner step,
and the open follow-ups. Nothing else keeps them: the next run's first turn
shows them again, from the PRs' boxes and the issues.

## Rules

- Never merge, deploy, force-push, rebase or touch production on your own.
- Never print a secret, and never put one on a command line.
- Helpers work in their own worktrees, never on the user's checkout.
- Run `/merge-line` from the repository's main checkout.
- When work turns up a gap, open a `follow-up` issue for it.
- Answers stay short: tables, links to PRs, one question at the end.
