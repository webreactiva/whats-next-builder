---
name: whats-next
description: Use when the user asks what they have pending today, what to work on next, what is assigned to them or unassigned, what they are waiting on, whether anyone answered their comments, what they have to review or what came back from review, what is blocked or stuck, or how the <<sprint / milestone / project>> is going in <<SYSTEM>> (<<SCOPE>>). Trigger in any language and however casually, e.g. <<5-8 real phrasings in the user's language>>, and at the start of a work session when the user asks for orientation without naming an item.
---

<!--
Template for the generated skill. Fill every <<...>>, delete what the system
cannot support (see "When a signal is missing" in the state model), and remove
this comment. Keep the section order: the answer depends on it. The filled
description must stay under 1024 characters.
-->

# What's next

<<SYSTEM>> stores facts: <<an issue is open, a PR has a reviewer, a pipeline
failed, a thread is unresolved>>. None of those facts says *what you should do
now*. This skill does that translation, and only that.

Scope: <<projects, with links>>. <<How to add or remove one: the single place
where scope lives.>>

**Read-only, without exceptions.** Never <<approve, merge, comment, transition,
assign, label, push or edit>> anything while running this skill. The user asked
a question; answer it. If the answer makes an action obvious, name it in plain
words and stop. They decide whether to take it.

## Setup

<<Verified in step 2 of the builder, dated. What access the collector needs (CLI
login or env var NAMES) and the minimum read-only scope; the steps to create or
renew it; when it expires; how to check it works (the identity call); and what
each auth error from the collector means:>>

| Error | Means | Fix |
|---|---|---|
| <<invalid key / 401>> | <<credential wrong or revoked>> | <<step N>> |
| <<missing scope / 403>> | <<valid but too narrow>> | <<step N>> |
| <<env var not set>> | <<profile not loaded>> | <<step N>> |

## Collect first, reason second

Run the collector once and work from its output:

```bash
<<COLLECT COMMAND>>
```

<<What it calls, how it authenticates (CLI session or the env var NAME), how
long it takes.>> It prints the JSON described in `references/state-model.md`:
`now`, `waiting`, `stuck`, `unassigned`, `other`, `containers`, `projects`,
`drift`, `partial`.

<<Flags table, only the flags the collector really implements:>>

| Flag | When |
|---|---|
| `--stale-days N` | The user thinks 3 days of silence is too tight or too loose |
| `--project P` | They ask about one project of the scope |
| `--container "T"` | They ask about a <<milestone/sprint>> that is not the current one |
| `--user someone` | "What is Ana waiting on?": that person's board |
| `--render` | The user wants the report printed straight in the terminal. Print it as is; do not rewrite it |
| `--plain` | With `--render`, when the terminal shows the hyperlink escapes as text |

<<MCP-only variant: replace the block above with: "Make exactly these tool
calls, in this order: ... Then classify with references/state-model.md before
answering." and say in the edge cases that the result is less repeatable.>>

Do not replace the collector with ad-hoc calls. The point of the skill is that
the same reasoning runs the same way every time. Reach for an extra read-only
call only when the user asks something the JSON does not carry (the content of
a thread, a diff, a CI log).

Treat the classification as the finding. If a line looks wrong, say so and show
the evidence (`why`, `idle_days`) instead of silently re-classifying it. A
disagreement between the rules and reality is information the user wants.

## The questions

One run answers all of them; never re-run per question.

| They ask | Use | Lead with |
|---|---|---|
| "what do I have today", "what now" | `now` | The single next thing, then the rest ranked; call out `today`/`overdue` |
| "what am I waiting on", "did they answer" | `waiting`, plus `REPLY_NEEDED`/`FOLLOW_UP` in `now` | Who owes the answer, since when |
| "what do I review" | `now` → `REVIEW_NEEDED`, `RE_REVIEW_NEEDED`, `PUSHED_SINCE_REVIEW`, `FOLLOW_UP` | Oldest wait first |
| "what came back from review" | own <<CRs>> in `CHANGES_REQUESTED`, `THREADS_TO_ANSWER`, `READY_TO_MERGE` | What each reviewer asked |
| "what is stuck" | `stuck` | What stopped, and why |
| "what is unassigned" | `unassigned` | By priority |
| "how is the <<container>> going" | `containers`, `projects`, `drift` | Days left, split by owner, anomalies |

With no qualifier ("whats-next"), answer `now` and add one line each for
`waiting` and `stuck` so nothing hides.

## Reading the NOW list

Items arrive ranked: the collector orders them so every answer agrees. The order
means *how much gets unblocked per minute spent*:

1. `READY_TO_MERGE`: one action releases finished work.
2. `CONFLICTS` / `CI_RED` on your own <<CR>>: it cannot move at all.
3. `CHANGES_REQUESTED` / `THREADS_TO_ANSWER` / `REPLY_NEEDED`: someone is parked until you answer.
4. `RE_REVIEW_NEEDED`, `FOLLOW_UP`, `REVIEW_NEEDED`: someone is blocked on you.
5. `PUSHED_SINCE_REVIEW`: probably yours; may be a rebase.
6. `IN_PROGRESS`, `NO_REVIEWER`: your work, already started.
7. `TODO`: not started, by priority then due date.

<<Delete the tiers this system does not produce.>>

Present about eight lines, then `+N more` with the states they hold. Reorder only
when the user gives a reason (a deadline, twenty free minutes), and say so.

## Answer format

Default: **<<emojis | table>>**. Switch when the user asks for the other ("as a
table", "with emojis", in whatever language they ask) and keep it for the rest
of the conversation. <<If the default is emojis:>> when the answer runs past
about eight items or spans several projects, close with one short line offering
the table view.

Write in the user's language: the examples below are in English, translate the
headings and the reasons, and keep identifiers (`!482`, `#512`, `ABC-123`,
status names, branch names) verbatim.

**Every reference is a link.** Each item carries the `url` of its page in
<<SYSTEM>>: write `[!482](url)`, never bare `!482`. The terminal turns markdown
links into clickable references, which is the difference between a report the
user reads and one they can act on. `--render` does the same with OSC 8
hyperlinks: iTerm2, WezTerm, Kitty, VS Code and GNOME Terminal make them
clickable, other terminals drop the escape and show plain text, and `--plain`
turns them off where the escapes show up literally.

### Emojis

```
📋 5 on you · ⏳ 2 waiting · 🧊 1 stuck · 🎯 <<Sprint 42>> closes in 3 days
⏰ Due today: [#512](url)

🔥 NOW
  🔀 [!482](url) Upgrade the HTTP client library
     Approved · CI green · no open threads
     → merge it
  🔁 [!477](url) Add CSV export to reports
     Ana asked you to review again 5 h ago · 1 open thread
     → read it again
  💬 [#530](url) Retry failed webhook deliveries
     Luis mentioned you yesterday and nobody answered
     → reply
  +3 more: 2 👀 reviews, 1 📌 not started

⏳ WAITING
  [!471](url) Ana has owed you the review for 4 days · 🔔 worth a nudge

🧊 STUCK: nothing idle for more than 3 days.

🙋 UNASSIGNED (3 in <<Sprint 42>>)
  [#541](url) Search filters on the dashboard · high priority

📊 <<SPRINT 42>> · 12 open / 30 closed · 3 days left
  ana 5 · you 4 · unassigned 3
  ⚠️ 2 not started with the deadline close

🩹 BOARD VS REALITY
  [#503](url) says "To do" · [!466](url) is open → "In progress"
```

One emoji per state, so the eye finds the kind of work before reading:

| Emoji | States |
|---|---|
| 🔀 | `READY_TO_MERGE` |
| 💥 | `CONFLICTS` |
| 🔴 | `CI_RED` |
| ✍️ | `CHANGES_REQUESTED` |
| 💬 | `THREADS_TO_ANSWER`, `REPLY_NEEDED`, `FOLLOW_UP` |
| 🔁 | `RE_REVIEW_NEEDED`, `PUSHED_SINCE_REVIEW` |
| 👀 | `REVIEW_NEEDED` |
| 🚧 | `IN_PROGRESS` |
| 🙈 | `NO_REVIEWER` |
| 📌 | `TODO` |
| 👉 | `DELEGATED` |
| ⛔ | `BLOCKED` |
| 📤 | `HANDED_OFF` |

With several projects, add a `▸ project` heading inside each block rather than
repeating the project on every line.

### Table

Same verdict line on top, then one table per non-empty block:

```
📋 5 on you · ⏳ 2 waiting · 🧊 1 stuck · 🎯 <<Sprint 42>> closes in 3 days

**🔥 Now**

|   | Ref | Title | Why | Next step | Idle |
|---|---|---|---|---|---|
| 🔀 | [!482](url) | Upgrade the HTTP client library | approved · CI green | merge it | 0.5 d |
| 🔁 | [!477](url) | Add CSV export to reports | Ana asked again | read it again | 0.2 d |

**⏳ Waiting**

| Ref | Title | Who | Since |
|---|---|---|---|
| [!471](url) | Onboarding checklist | Ana | 4 d 🔔 |
```

With several projects, add a `Project` column. Empty blocks stay one line of
text (`🧊 Stuck: nothing`), never an empty table.

### Habits in both formats

- **Open with the verdict.** One line with the counts and the deadline. If one
  thing dominates ("12 of the 18 are re-reviews Ana asked for today"), say it.
- **Every line says why.** `why` and `idle_days` let the user disagree without
  opening <<SYSTEM>>.
- **Say the empty buckets out loud.** "Nobody owes you an answer" is a real
  answer; a missing section reads like a bug.

## Board drift

<<Only if the system has statuses and a CR link.>> `drift` compares what the
statuses claim against what the work did. Report it as a short appendix, never
mixed into the work lists, and never fix it: changing statuses is the user's
call, and this skill does not write.

## <<Container>> panorama

`containers` carries the split by owner, open vs closed, unassigned, blocked and
not started. Flag two things even when the user only asked "how are we doing":
items with no assignee, and items nobody started while the due date is close.
Show `why_this_one` when more than one <<container>> is open at once.

## Edge cases

- **`error` in the output**: the collector failed (auth, network, API change).
  Report the message; for auth errors point to the matching row in *Setup*. Do
  not fall back to scattered calls.
- **`partial` non-empty**: a collection hit its cap; say the list is partial.
- **`now` is empty**: a good answer, not a broken run. Show what is waiting and
  what the <<container>> still needs.
- **Asked about someone else**: rerun with `--user`; their `waiting` is
  usually your `now`.
<<System-specific cases found while generating.>>

## Where the rules live

`references/state-model.md` holds the states this skill uses, where each signal
comes from in <<SYSTEM>>, and the known limits. Read it when a classification
looks wrong, when the user challenges a line, or when you need to explain why
something landed where it did.
