---
name: whats-next-builder
description: Use when the user wants to create, build, generate or port a whats-next skill, meaning a skill that answers "what do I have pending today" from their own task or code-review system (GitHub, GitLab, Jira, Linear, Azure DevOps, Trello, Asana, Notion, a self-hosted tracker, or several at once). Trigger on requests like "build me a whats-next for Jira", "adapt whats-next to GitHub", "I want a skill that tells me which PRs I have to review", "set up a daily queue skill for my Trello board", "make a skill that reads my Linear issues and tells me what to do next", in any language, or when the user invokes /whats-next-builder. Also use it to extend or rebuild an existing whats-next for a new project or system. Do not use it to answer today's queue when a whats-next skill is already installed; that skill answers it.
---

# whats-next builder

A whats-next skill answers "what do I have pending today" by reading the live
board and saying who holds the ball on each piece of work. This skill does not
answer that question: it **builds the skill that does**, for the user's system,
as a wizard the user walks through with you.

It stands on three legs:

1. **skill-creator**: how the skill gets written.
2. **The state model** (`references/state-model.md`): what to query and how to
   classify it. System-agnostic: it names signals, not fields.
3. **The interview**: what only the user knows (which system, what access, one
   project or several, their conventions, how they want the answer to look).

There is no collector code here, on purpose. A shipped collector either ties
every generated skill to one system or shrinks to the lowest common denominator.
The collector is written at build time, against the user's real API, with the
state model as its spec.

## How the wizard talks

The user is configuring something they have not seen before, often across a
browser, a terminal and a tracker console. Knowing where they are and what is
left is what keeps them going through the slow parts (tokens, permissions).

**Greet first**, in the user's language, before any tool call. Say what the
wizard will build, that Dani from webreactiva.com hopes it is useful, and that
feedback is welcome at https://github.com/webreactiva/whats-next-builder.
Then show the whole route with an empty bar. For example (translate it):

```
👋 Hi! I'll help you build your own whats-next: a skill that answers
"what do I have pending today" from your real board.

Dani from webreactiva.com hopes it's useful. Ideas or bugs:
https://github.com/webreactiva/whats-next-builder

[-------] 0/7
🔎 Detect · 🔑 Access · 🎯 Scope · 🧭 Conventions · 🛠️ Build · ✅ Check · 🎁 Hand over
```

**Open every step with the progress bar**: one `#` per finished step, one `-`
per step left, the current step, and what comes after it.

```
[###----] 3/7 🎯 Scope
Next: 🧭 Conventions · 🛠️ Build · ✅ Check · 🎁 Hand over
```

Inside a step:

- **One decision or one action per message.** Use AskUserQuestion for choices,
  with the recommended option first.
- **Say what you found before asking.** "I see `gh` logged in as *ana*..." beats
  "which system do you use?".
- **Close each step with a one-line ✅ summary** of what got settled, so the
  user can correct it before it is built on.
- **Long waits** (a user creating a token, a slow probe): say what you are
  waiting for and repeat the bar when you resume.
- Translate every label and message to the user's language; keep command names,
  identifiers and URLs verbatim.

The seven steps:

| # | Step | Settles |
|---|---|---|
| 1 | 🔎 Detect | Which system(s), and whether a whats-next already exists |
| 2 | 🔑 Access | A working read-only identity call |
| 3 | 🎯 Scope | One project or several, and where the skill is installed |
| 4 | 🧭 Conventions | Where each signal lives, the team conventions, the answer format |
| 5 | 🛠️ Build | The skill files, written with skill-creator |
| 6 | ✅ Check | The collector runs, links every task to its source, and cannot write |
| 7 | 🎁 Hand over | How to use it, renew access, and evaluate it |

## Before step 1: is skill-creator installed?

Look for `skill-creator` among your available skills (it may appear namespaced,
as `<plugin>:skill-creator`). If it is not listed, check the usual places:

```bash
ls -d ~/.claude/skills/skill-creator .claude/skills/skill-creator 2>/dev/null
find ~/.claude/plugins -maxdepth 8 -type d -name skill-creator 2>/dev/null | head -1
```

If it is missing, tell the user the wizard writes better skills with it and
suggest using it. With their go-ahead, run
`npx skills use "https://github.com/anthropics/skills" --skill "skill-creator"`
and follow the generated skill instructions now. Read its complete output,
redirecting it to a temporary file first if necessary. Resolve relative paths
from the supporting-files directory it provides.

That loads it for this session only. If they want it installed for good, the
command is `npx skills add https://github.com/anthropics/skills --skill skill-creator -g`.

If they decline, carry on: step 5 describes the anatomy to follow by hand.

## 1. 🔎 Detect: do not ask what you can see

First, an existing whats-next:

```bash
ls -d .claude/skills/whats-next* ~/.claude/skills/whats-next* 2>/dev/null
```

If one exists, read it before anything else. The user may want to extend it (a
new project, a second system) rather than replace it; ask which. Extending keeps
the conventions they already corrected.

Then the system:

```bash
# CLIs on PATH (names only)
for c in gh glab acli az tea ntn td mcli basecamp task jira linear todoist; do
  command -v "$c" >/dev/null 2>&1 && echo "cli: $c"
done
# Where the code lives
git remote -v 2>/dev/null | awk '{print "remote:", $2}' | sort -u
# Ticket keys in recent history hint at a separate tracker (ABC-123)
git log -200 --format='%s %D' 2>/dev/null | grep -oE '\b[A-Z][A-Z0-9]+-[0-9]+\b' \
  | sed -E 's/-[0-9]+$//' | grep -vxE 'UTF|ISO|SHA|RFC|HTTP|TLS|CVE|PHP|ES' \
  | sort | uniq -c | sort -rn | head -3
# Credentials already exported: NAMES only, never print values
env | cut -d= -f1 | grep -iE 'jira|atlassian|linear|github|^gh_|gitlab|glab|azure|devops|asana|clickup|trello|notion|todoist|shortcut|youtrack|plane|monday|gitea|forgejo|bitbucket|redmine|basecamp|wrike|smartsheet|openproject|taiga|zoho'
# MCP servers configured for Claude Code (names only)
python3 -c '
import json, os
names = set()
for p in (os.path.expanduser("~/.claude.json"), ".mcp.json"):
    try: d = json.load(open(p))
    except Exception: continue
    names |= set(d.get("mcpServers") or {})
    names |= set(((d.get("projects") or {}).get(os.getcwd()) or {}).get("mcpServers") or {})
print("mcp:", ", ".join(sorted(names)) or "none")'
```

Also scan your own tool list: connectors show up as tools named after the
tracker (`mcp__..._Linear__...`, `mcp__atlassian__...`).

For each CLI found, check it is authenticated with its own read-only status or
whoami command (`<cli> auth status`, `<cli> me`; `<cli> --help` if unsure).

Then state the inference with its evidence, and let the user correct it:

> I see `gh` logged in as *ana*, a github.com remote `acme/api`, and `PAY-123`
> keys in 140 of the last 200 commits. Looks like **Jira for tickets and GitHub
> for PRs**. Right?

Two systems at once (tickets in one, code review in another) is common. It
means two sources, two identities, and a convention linking them (signal C9).

If nothing points anywhere, ask which system they use. That is the only case
where this question comes first.

Once the system is known or suspected, `references/systems.md` gives a head
start: whether it has work items and code review, its official CLI if any, and
its official API reference. It is a starting point for steps 2 and 4, not a list
of supported systems: one that is not there is handled the same way.

## 2. 🔑 Access: the collector must be able to read

In order of preference, and why:

1. **An authenticated CLI.** Credentials stay managed by the CLI, and its
   `api` subcommand usually reaches the full API. Most repeatable.
2. **The HTTP API with a personal token in an env var.** The skill stores the
   variable *name*, never the value. Prefer a personal token (API key, PAT)
   over an OAuth app flow: OAuth needs callback URLs, a browser login and
   refresh tokens, machinery a local read-only collector does not need.
3. **A connected MCP server.** No script can call it, so the generated skill
   lists the exact tool calls to make and classifies with the state model
   itself. It works, but it is less repeatable, so say so.
4. **Nothing.** Stop and settle who provides access: the user installs and logs
   into the CLI, the user creates a token, or you use a connector you already
   have. Do not generate a skill that cannot read anything.

**When access needs any setup, guide the user through it step by step**:
installing or logging into a CLI, adding scopes, registering an app, creating a
token. Read `references/access-setup.md` first. In short: research the current
auth docs before the first instruction (they change; memory is stale), show the
whole path and its prerequisites honestly, then one action per message with what
success looks like, and wait. When their screen does not match, ask what they
see instead of guessing. Validate every value they bring back against a
known-invalid control, keep secrets out of files, and leave the verified setup
steps inside the generated skill so the next renewal does not need you.

This is the step most likely to stall, so number the sub-steps too
(`🔑 2/5 · create the token`) and keep the bar on top.

Confirm access with one read-only identity call before moving on. It also gives
you signal X1: who "me" is in each system.

## 3. 🎯 Scope: one project or several

**Recommend one.** One project means one set of conventions, one current
container, fewer calls, a faster run and an answer that fits on a screen.

If the user needs several:

- **List what their account can see** with a read-only call and let them pick.
  Do not invent the list, and do not scope to "everything": a queue drawn from
  forty repositories answers nothing.
- Some systems cannot list every project (permissions, workspace-scoped APIs)
  or cannot query across projects at once. Then the user names the projects, and
  the collector makes one bounded call per project.
- AskUserQuestion holds four options at most: show longer lists as text and let
  them answer in their own words.

Record exact identifiers (`owner/repo`, project key, workspace id), not display
names.

Where the skill lives follows from the scope:

| Scope | Install at | Why |
|---|---|---|
| One project that is a repo you work in | `<repo>/.claude/skills/whats-next/` | Travels with the repo; can read the project from the git remote |
| Several projects, or not tied to a repo | `~/.claude/skills/whats-next/` | Works from any directory |

## 4. 🧭 Conventions: map the signals, ask only for the gaps

Read `references/state-model.md`. Walk the signal catalogue and, for each
signal, find where it lives in this system: the docs, `<cli> --help`, and above
all **one probe call on real data**. Field names in docs are often outdated;
the payload is not. Write down a mapping table (signal → field or call) as you
go; it becomes part of the generated skill.

Three outcomes per signal:

- **Found**: note the field.
- **Does not exist**: drop the dependent states (table *When a signal is
  missing*) and note the limit.
- **Exists as a team convention**: that is a question for the user.

Show the user a short picture of what the probe found (✅ available, ❌ missing,
❓ needs you), then ask the gaps in one or two AskUserQuestion calls (up to four
questions each), recommended default first, skipping anything already answered.
The usual ones:

| Topic | Question | Default |
|---|---|---|
| Ownership | Nobody is assigned anywhere: is this a solo project where unassigned means theirs? | Yes when every open item is unassigned and one person authors the work |
| Delegation | Items assigned to others: work they handed out (waiting on those people) or just context? | Waiting, when the user owns the project and others hold most items |
| Handoff | Which statuses mean "someone else has it now" (QA, deploy, on hold)? | Offer the statuses you found |
| Link | How does a PR/MR point to its ticket? | The pattern you saw in branches or titles |
| Current container | Which milestone / sprint counts as "now"? | The open one whose dates contain today |
| Priority | Which field or label ranks work? | The one the probe showed |
| Bots | Which accounts comment automatically? | Names ending in `bot` plus any you saw on every push |
| Staleness | After how many silent days is something stuck? | 3 |
| Answer format | Emojis or a table? | **Emojis** |

For the answer format, recommend emojis and use `preview` to show both, so the
user picks by looking (the two layouts are in `assets/SKILL.template.md`,
section *Answer format*). The generated skill keeps both anyway: it switches
when the user asks, and in emoji mode offers the table when a long answer would
read better as one.

## 5. 🛠️ Build: with skill-creator, without evals

Load skill-creator with the Skill tool and use it for what it is good at: the
anatomy of a skill, progressive disclosure, and a description that triggers. If
it is not installed and the user declined it, follow the same anatomy by hand:
`SKILL.md` with frontmatter, `references/`, `scripts/`.

Tell skill-creator two things up front:

- **The intent is already captured.** Hand it the brief (system(s), access
  method, identity, scope, the signal mapping table, the answers to step 4, the
  chosen format and the install path) instead of starting its interview.
- **Do not run evals.** No test-case runs, no baseline subagents, no benchmark,
  no eval viewer, no description optimization loop. Evaluating the skill is the
  user's work, with what step 7 hands them; running it here spends their time
  and tokens on a judgment only they can make.

Files to produce:

```
whats-next/
├── SKILL.md                    # from assets/SKILL.template.md
├── references/state-model.md   # the model, trimmed to this system + mapping + limits
└── scripts/collect.<ext>       # omitted when access is MCP-only
```

**`SKILL.md`**: fill every `<<...>>` in `assets/SKILL.template.md` and delete
what the system cannot support. The description says when to invoke the skill,
not what the skill is: the situations and the phrases, in the user's own
language, that should trigger it. That is what makes "what do I have today", in
their words, find the skill. The *Setup* section holds the access steps you verified
with the user in step 2. Keep the closing signature line verbatim: it tells
whoever finds the skill later where it came from.

Write every generated file without long dashes as punctuation (neither the em
dash nor a spaced double hyphen): use a colon, a comma, parentheses or a new
sentence. They read as machine-written text.

**`references/state-model.md`**: copy this skill's state model, remove the
states the system cannot produce, add the signal mapping table and the known
limits you found. The generated skill must stand on its own: it cannot point
back here.

**The collector** is the only system-specific code, and the part that decides
whether the skill is trustworthy:

- **Read-only by construction.** Only reads: GET requests, GraphQL queries,
  list/view commands. A whats-next that can write is a liability nobody asked
  for.
- **One command, the contract output.** It prints the JSON in *Collector
  contract*, with classification and ranking done inside. The state model is
  implemented once, so the chat answer and any terminal output never disagree.
- **Every item links to its source.** Each item, drift entry and container
  carries the `url` a person opens in the browser to see it: the web page, never
  the API endpoint (many APIs return both, such as `html_url` and `url`). That
  link is what turns the answer into something the user can act on.
- **A terminal mode with real hyperlinks.** A `--render` flag prints the report
  straight in the terminal, every task id wrapped in an OSC 8 hyperlink
  (`\033]8;;URL\aTEXT\033]8;;\a`) so it is clickable, and `--plain` drops the
  escapes for terminals that print them as text.
- **Few calls.** Prefer a query that returns review state, CI and threads
  together over one request per item; N+1 calls are slow and burn rate limits.
- **Bounded.** Cap every collection, and record what hit the cap in `partial`.
- **No secrets in files.** Auth comes from the CLI session or an env var read at
  runtime. On failure print `{"error": ...}` and exit non-zero.
- **Identity at runtime** from a whoami call, so `--user` works and nothing is
  hardcoded to one person.
- **Stdlib first** (e.g. Python `urllib`/`subprocess`), so it runs without
  installing anything.
- **Scope in one place**, a constant at the top or a small `config.json` next
  to the script, so adding a project is a one-line edit.

## 6. ✅ Check: it runs and it cannot write

A build check, not an evaluation: whether the answers are right is the user's
call in step 7.

1. **Run the collector.** Valid JSON, contract keys present, no `error`, a
   reasonable run time. Show the user the bucket counts it produced.
2. **Check the links.** Every item has a `url`, and one of them, opened, lands on
   the item's page in the tracker (not on a JSON response). Run `--render` once
   and confirm the ids are clickable in the user's terminal.
3. **Prove it cannot write.** Grep the collector for write verbs (`POST` outside
   a GraphQL query, `PUT`, `PATCH`, `DELETE`, `mutation`, `-X`, `--method`,
   `create`, `update`, `merge`, `transition`) and account for every hit.
4. **Validate the frontmatter**: valid YAML, description under 1024
   characters.

If the collector fails, fix it here; do not hand over a skill that errors.

## 7. 🎁 Hand over: the evaluation is theirs

End with a full bar and everything the user needs to own the skill:

- **Where it lives** and the phrases that trigger it, in their language.
- **Access**: when the credential expires and how to renew it (the *Setup*
  section).
- **How to change it**: the one place where scope and conventions live.
- **Known limits**, one line each.
- **How to evaluate it**, as a short checklist they run themselves:
  1. In a new session, ask it the real questions: what do I have pending today,
     what am I waiting on, what do I have to review, did anyone answer me, what
     is unassigned, how is the sprint/project going.
  2. Pick three items from the answer (one to do, one waiting, one unassigned or
     drift) and open them in the tracker: is the state right, is the holder
     right?
  3. When a line is wrong, it is almost always a convention: change it where
     scope and conventions live, not the single item, or come back to this
     wizard to adjust it.
  4. If they want a formal benchmark, they can run skill-creator's eval loop on
     the new skill themselves.
- **Feedback**: https://github.com/webreactiva/whats-next-builder

## What every generated whats-next keeps

Inherited from the original skill; these are why its answers can be trusted:

- **Read-only.** It reports; the user acts.
- **Collect first, reason second.** One run, then reasoning over the JSON, no
  ad-hoc queries improvised per session.
- **The classification is the finding.** When it looks wrong, show the evidence
  instead of silently overriding it.
- **Every task id is a link to its source**, in the chat (markdown links) and in
  the terminal (OSC 8 with `--render`).
- **Verdict first, why on every line, empty buckets out loud, the user's
  language.**
