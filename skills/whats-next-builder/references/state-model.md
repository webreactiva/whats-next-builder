# State model

The template every generated whats-next inherits. A tracker stores facts (an
issue is open, a PR has a reviewer, a pipeline failed, a thread is unresolved);
this model turns them into **who holds the ball** on each piece of work.

It names *signals*, never fields or endpoints. Generating a skill means finding
where each signal lives in the user's system, or confirming it does not exist
and dropping the states that depend on it. The signals the API cannot settle on
its own are exactly the questions to ask the user.

## Contents

- [Vocabulary](#vocabulary)
- [The questions the skill answers](#the-questions-the-skill-answers)
- [Signal catalogue](#signal-catalogue)
- [States on a work item](#states-on-a-work-item)
- [States on your own change request](#states-on-your-own-change-request)
- [States on someone else's change request](#states-on-someone-elses-change-request)
- [Buckets and ranking](#buckets-and-ranking)
- [Urgency, staleness and risk](#urgency-staleness-and-risk)
- [Board drift](#board-drift)
- [Collector contract](#collector-contract)
- [When a signal is missing](#when-a-signal-is-missing)
- [Known limits every skill inherits](#known-limits-every-skill-inherits)

## Vocabulary

Generic words, so the model reads the same whatever the tracker calls things.
The generated skill uses the system's own words in its answers.

| Here | What it covers |
|---|---|
| **work item** | issue, ticket, task, card, story, bug |
| **change request** (CR) | pull request, merge request, changeset, patch under review |
| **container** | milestone, sprint, cycle, iteration, fix version, release: a timebox with a due date |
| **project** | repository, project, board, workspace, list: the unit the user scopes to |
| **holder** | whoever has to move next: you, a named person, CI, QA, nobody |

## The questions the skill answers

One collection answers all of them. The phrasing picks a slice.

| The user asks | Slice | Lead with |
|---|---|---|
| "what do I have pending today", "what now", "my queue" | `now` | The single next thing, then the rest ranked; call out what is due today or overdue |
| "what am I waiting on", "did anyone answer me" | `waiting` + reply states in `now` | Who owes the answer, since when |
| "what do I have to review" | `now` filtered to review states | Oldest wait first: their wait is your cost |
| "what came back from review" | own CRs in `CHANGES_REQUESTED`, `THREADS_TO_ANSWER`, `READY_TO_MERGE` | What each reviewer asked |
| "what is stuck", "anything blocked" | `stuck` | What stopped moving, and why |
| "what is unassigned", "what can I pick up" | `unassigned` | By priority, inside the current container |
| "how is the sprint / milestone / project going" | `containers` + `projects` + `drift` | Days left, split by owner, anomalies |

## Signal catalogue

The checklist for generation. For each signal, find the field or call that
carries it in the user's system, probing real data rather than trusting docs.
Column **Ask if not inferable** is the interview.

### Context

| Id | Signal | Settles | Ask if not inferable |
|---|---|---|---|
| X1 | Current user's identity (login, account id) in each system | Every "mine" decision | Never ask; get it from a whoami call |
| X2 | Projects in scope | What is collected | One project or several; which ones |
| X3 | Which container is "current" | TODO vs BACKLOG, the panorama | Rule: dates contain today, an explicit "active" flag, or a named one |
| X4 | Bot accounts | Idle time and "who moved last" | Which accounts comment automatically (coverage, linters, dependency bots) |

### Work items

| Id | Signal | Enables | Ask if not inferable |
|---|---|---|---|
| W1 | Assignees | mine / theirs / unassigned | In a solo project nobody assigns: ask whether unassigned counts as theirs |
| W2 | Workflow status (column, state, status label) | `IN_PROGRESS`, `HANDED_OFF`, drift | Which statuses mean "someone else has it" (QA, deploy, on hold) and which mean "not started" |
| W3 | Due date on the item | "today" and "overdue" | Nothing to ask |
| W4 | Container membership, with start/due or active flag | `TODO` vs `BACKLOG`, panorama | See X3 |
| W5 | Priority | Order inside a tier | Which field or label carries it, and its order |
| W6 | Blockers / dependencies, with the blocker's open state | `BLOCKED` | Nothing to ask |
| W7 | Comment stream: author, timestamp, mentions | `REPLY_NEEDED` | Nothing to ask |
| W8 | Last update | Idle time where there is no comment stream | Nothing to ask |

### Change requests

| Id | Signal | Enables | Ask if not inferable |
|---|---|---|---|
| C1 | Author, draft flag | Own vs theirs, `IN_PROGRESS` | Nothing to ask |
| C2 | Reviewers and each one's review state (none, commented, changes requested, approved) | Review states | Nothing to ask |
| C3 | Re-request of review, and to whom | `RE_REVIEW_NEEDED` vs `FOLLOW_UP` | Nothing to ask |
| C4 | Approval / merge eligibility, and whether review is required at all | `READY_TO_MERGE`, `NO_REVIEWER` | No (branch rules usually say it) |
| C5 | CI status on the head commit | `CI_RED`, `WAITING_FOR_CI` | Nothing to ask |
| C6 | Conflicts with the target | `CONFLICTS` | Nothing to ask |
| C7 | Unresolved threads count | `THREADS_TO_ANSWER`, `WAITING_FOR_REVIEWER` | Nothing to ask |
| C8 | Timeline: comments (who, when), pushes (who, when) | Who moved last | Nothing to ask |
| C9 | Link CR ↔ work item | "covered" items, drift | The convention: native link, branch name (`feature/123`), key in title (`ABC-123`) |
| C10 | Merged and closed-unmerged CRs in the current container | Drift, abandoned work | Nothing to ask |

If work items and change requests live in different systems (tickets in one,
code in another), C9 is the bridge and it is almost always a convention the user
has to confirm. Identities differ per system too: X1 becomes one per source.

## States on a work item

Evaluated in order; the first match wins.

| State | Condition | Holder | Bucket |
|---|---|---|---|
| `REPLY_NEEDED` | Someone else commented after your last comment, and either mentioned you or you were already in that conversation | you: answer | now |
| `BLOCKED` | An open work item blocks it | nobody: the blocker moves first | waiting if yours, else other |
| `HANDED_OFF` | Status means QA, deploy, staging or on hold | QA / deploy / nobody | waiting if yours, else other |
| `UNASSIGNED` | No assignee, inside the scope (current container, or the project when there are none) | nobody | unassigned |
| `DELEGATED` | Assigned to someone else, and the user hands out work on this project (a convention to confirm) | them | waiting |
| `THEIRS` | Assigned to someone else | them | other |
| `IN_PROGRESS` | Yours, status says started | you: finish it | now |
| `TODO` | Yours, in the current container, or due today/overdue/soon, or the system has no containers | you: start it | now |
| `BACKLOG` | Yours, later container or none | you, later | other |

`REPLY_NEEDED` requires a mention or earlier participation on purpose: a comment
on your ticket from the person who filed it is often context, not a question,
and flagging every one of them buries the real replies.

An item whose change request is already open is **covered**: it is shown once,
through the CR line. One piece of work, one line.

## States on your own change request

Evaluated in order; earlier conditions block the later ones.

| State | Condition | Holder |
|---|---|---|
| `IN_PROGRESS` | Draft | you: finish it and mark it ready |
| `CONFLICTS` | Conflicts with the target | you: rebase |
| `CI_RED` | CI failed on the head | you: fix it |
| `CHANGES_REQUESTED` | Any reviewer requested changes | you: address and re-request |
| `THREADS_TO_ANSWER` | Open threads and a reviewer spoke after you | you: reply and resolve |
| `READY_TO_MERGE` | Approved (or no approval required), CI green or absent, no open threads | you: merge |
| `WAITING_FOR_CI` | CI running or pending | CI |
| `NO_REVIEWER` | Review is required and nobody was asked | you: ask someone |
| `WAITING_FOR_REVIEWER` | Open threads, you spoke last | the reviewer |
| `WAITING_FOR_REVIEW` | Reviewer assigned, nothing else pending | the reviewer |

`NO_REVIEWER` sits in `now`, not `waiting`: a CR nobody was asked to review is
not waiting, it is invisible.

## States on someone else's change request

| State | Condition | Holder |
|---|---|---|
| `THEIR_DRAFT` | Draft, you are a reviewer | them; nothing to review yet |
| `WAITING_FOR_MERGE` | You approved it | them: they merge |
| `RE_REVIEW_NEEDED` | You requested changes and they moved since; or they re-requested you after your comment | you |
| `REVIEW_NEEDED` | You are a reviewer and never commented | you |
| `PUSHED_SINCE_REVIEW` | Commits after your comment, no re-request | you, with less certainty |
| `FOLLOW_UP` | They replied to your comments, no new code | you |
| `WAITING_FOR_AUTHOR` | You spoke last, nothing since | them |
| `REPLY_NEEDED` | Not a reviewer, but mentioned after your last comment | you |
| `NOT_MINE` | Anything else | them |

**A reply is not a fix.** A comment after your review is `FOLLOW_UP` (read it; it
may not need another pass). Commits plus an explicit re-request is
`RE_REVIEW_NEEDED` (a full re-read). Collapsing the two either wastes the user's
time or drops work.

Some systems keep a reviewer's old verdict after a re-request, others reset it.
Normalize in the collector: a reviewer whose review was requested again counts
as not reviewed.

## Buckets and ranking

Every item lands in exactly one bucket: `now`, `waiting`, `unassigned`, `other`,
or `covered`. `stuck` is a **view** over `now` and `waiting`, not a bucket: an
item that is yours and stale stays in `now`, where the ball is.

`now` is ranked by how much gets unblocked per minute spent:

| Tier | States | Why here |
|---|---|---|
| 1 | `READY_TO_MERGE` | One action releases finished work |
| 2 | `CONFLICTS`, `CI_RED` | The CR cannot move at all |
| 3 | `CHANGES_REQUESTED`, `THREADS_TO_ANSWER`, `REPLY_NEEDED` | Someone already spent their time and is parked until you answer |
| 4 | `RE_REVIEW_NEEDED`, `FOLLOW_UP`, `REVIEW_NEEDED` | Someone is blocked on you; oldest wait first |
| 5 | `PUSHED_SINCE_REVIEW` | Probably yours, but may be a rebase |
| 6 | `IN_PROGRESS`, `NO_REVIEWER` | Your work already started |
| 7 | `TODO` | Not started; by priority, then due date |

Inside a tier: urgency first (`overdue` > `today` > `due_soon` > `normal` >
`none`), then idle days, longest first.

`waiting` is ranked by how long the other side has been silent; that alone
decides when a wait becomes a nudge. `unassigned` is ranked by priority, then
urgency.

Rank in the collector, not in the answer: the order must not change between runs
or between the chat answer and anything printed in the terminal.

## Urgency, staleness and risk

Three independent dimensions. Do not invent combined states
(`URGENT_REVIEW_NEEDED`); they multiply the table for nothing.

- **`urgency`**: from the earliest of the item's own due date and its
  container's due date: `overdue` (past), `today`, `due_soon` (≤ 3 days by
  default), `normal`, `none`. "What do I have pending **today**" is answered by
  `now` plus a call-out of everything `today` or `overdue`.
- **`stale`**: no human activity for N days (default 3) on work in flight in
  `now` or `waiting`. Not applied to `TODO`/`BACKLOG`: work never started has no
  ball to lose.
- **`at_risk`**: a `TODO` nobody started whose urgency is `overdue`, `today`
  or `due_soon`. The one case where "not started" is alarming.

`stuck` = items in `now`/`waiting` that are `stale`, `at_risk`, `BLOCKED`,
`NO_REVIEWER`, `CONFLICTS` or `CI_RED`.

## Board drift

Statuses are a human-maintained mirror of the work, so they lag. Reality (CRs,
CI, merges) wins; drift is reported, never corrected. Only applies when the
system has statuses (W2) and a CR link (C9).

| Drift | Suggests |
|---|---|
| Open CR while the work item still says not started (or has no status) | move it to in progress |
| CR merged, work item still open, not handed off, no other open CR | move it on (QA/deploy) or close it |
| CR closed without merging, work item open, no other CR | find out whether the work was redone elsewhere |
| Work item in the current container with no assignee | someone has to own it |

## Collector contract

The JSON every generated collector prints, whatever the system. The generated
SKILL.md reasons over these keys, so keep them stable; add fields freely, do not
rename these.

```json
{
  "generated_at": "2026-09-17T08:00:00Z",
  "me": {"<system>": "<login>"},
  "projects": [{"id": "acme/api", "counts": {"now": 5, "waiting": 2, "stuck": 1}}],
  "containers": [{
    "project": "acme/api", "title": "Sprint 42", "start": "2026-09-08", "due": "2026-09-19",
    "why_this_one": "dates contain today; also open: Hotfix 3",
    "open": 12, "closed": 30, "by_owner": {"ana": 5, "me": 4, "unassigned": 3},
    "unassigned": ["#541"], "blocked": ["#538"], "not_started": ["#542"]
  }],
  "now": [{
    "kind": "change_request", "project": "acme/api", "ref": "!482", "title": "Upgrade the HTTP client library",
    "url": "https://...", "state": "RE_REVIEW_NEEDED", "holder": "me",
    "next_action": "they pushed and asked you again, read it again",
    "why": ["CI green", "1 open thread", "re-requested 5h ago"],
    "idle_days": 0.2, "urgency": "due_soon", "due": "2026-09-19", "container": "Sprint 42",
    "stale": false, "at_risk": false, "linked": ["#512"]
  }],
  "waiting": [], "stuck": [], "unassigned": [], "other": [],
  "drift": [{"ref": "#503", "url": "https://...", "says": "To do", "reality": "!466 is open", "suggest": "In progress"}],
  "partial": ["acme/web: more than 100 open PRs, showing the 100 most recent"],
  "errors": []
}
```

`url` is required on every item, drift entry and container: the page a person
opens in the tracker, never an API endpoint. The answer links every task id to
it.

On a fatal failure print `{"error": "<message>"}` and exit non-zero. The skill
reports it instead of guessing with scattered calls.

## When a signal is missing

Drop the states that depend on it and say so in the generated skill's known
limits, so a missing line reads as a limit, not a bug.

| Missing | Dropped | The skill says |
|---|---|---|
| Change requests (pure task manager) | Every CR state, C9 drift | "this system has no code review"; ask whether code lives elsewhere (second source) |
| Comment stream (W7) | `REPLY_NEEDED` | "comments are not read" |
| Per-reviewer review state (C2) | `REVIEW_NEEDED`/`RE_REVIEW_NEEDED` split | fall back to the timeline: who spoke last |
| Re-request events (C3) | `RE_REVIEW_NEEDED` becomes `PUSHED_SINCE_REVIEW` | Nothing extra |
| CI (C5) | `CI_RED`, `WAITING_FOR_CI` | `READY_TO_MERGE` rests on approval alone |
| Containers (W4) | `BACKLOG`, the container panorama | a per-project panorama instead; `TODO` ranked by due date |
| Statuses (W2) | `IN_PROGRESS` on items, `HANDED_OFF`, drift | Nothing extra |
| Blockers (W6) | `BLOCKED` | Nothing extra |
| Cross-project queries | nothing, but one call per project | list the projects that hit a cap in `partial` |

## Known limits every skill inherits

- **A rebase looks like an answer.** A push cannot be told from a real fix
  without reading the diff; hence `PUSHED_SINCE_REVIEW` separate from
  `RE_REVIEW_NEEDED`.
- **Threads are counted, not read.** The collector knows how many are open, not
  what they say. Read the specific thread when the user asks.
- **Bots fake activity.** Coverage, linters and dependency bots comment on every
  push; without filtering them a dead CR looks alive.
- **Caps.** Every collection is bounded; hitting a bound goes into `partial`, and
  the answer says the list is partial.
- **Two containers at once** (a hotfix beside the regular release):
  `why_this_one` names the others so the numbers are not read against the wrong
  one.
