# Known systems: where to start looking

A head start, not a support list. For each system: whether it tracks work items
or change requests, its **official** CLI if one exists, and the official API
reference. Nothing here replaces steps 1-4 of `SKILL.md`: detect, research the
current auth docs (`access-setup.md`), probe real data, map the signals. A
system missing from this table is handled exactly the same way.

Checked on 2026-09-17 (every URL resolved; official status confirmed against the
vendor's own site or organization). Consoles, CLIs and auth flows change,
so treat anything older than a few months as a lead to verify, not a fact.

## Contents

- [Code hosting with review](#code-hosting-with-review)
- [Issue and project trackers](#issue-and-project-trackers)
- [Task managers](#task-managers)
- [How to read this table](#how-to-read-this-table)

Columns: **WI** = work items, **CR** = change requests (PRs/MRs).

## Code hosting with review

| System | WI | CR | Official CLI | API reference |
|---|---|---|---|---|
| GitHub | ✓ | ✓ | `gh`: [manual](https://cli.github.com/manual/) (`gh api` reaches REST and GraphQL) | [REST](https://docs.github.com/en/rest) · [GraphQL](https://docs.github.com/en/graphql) |
| GitLab | ✓ | ✓ | `glab`: [docs](https://docs.gitlab.com/cli/) (`glab api` reaches REST and GraphQL) | [REST](https://docs.gitlab.com/api/rest/) · [GraphQL](https://docs.gitlab.com/api/graphql/) |
| Azure DevOps (Boards, Repos) | ✓ | ✓ | `az` + `azure-devops` extension: [docs](https://learn.microsoft.com/en-us/azure/devops/cli/) | [REST](https://learn.microsoft.com/en-us/rest/api/azure/devops/) |
| Bitbucket Cloud | ✗ | ✓ | Atlassian CLI `acli`: [commands](https://developer.atlassian.com/cloud/acli/reference/commands/) (check which products it covers) | [REST](https://developer.atlassian.com/cloud/bitbucket/rest/) |
| Gitea | ✓ | ✓ | `tea`: [repo](https://gitea.com/gitea/tea) | [API usage](https://docs.gitea.com/development/api-usage) (Swagger at `/api/swagger` on the instance) |
| Forgejo / Codeberg | ✓ | ✓ | none official | [API usage](https://forgejo.org/docs/latest/user/api-usage/) (Swagger at `/api/swagger` on the instance) |
| Gerrit | ✗ | ✓ | SSH commands: [index](https://gerrit-review.googlesource.com/Documentation/cmd-index.html) | [REST](https://gerrit-review.googlesource.com/Documentation/rest-api.html) |

## Issue and project trackers

| System | WI | CR | Official CLI | API reference |
|---|---|---|---|---|
| Jira Cloud (incl. Jira Service Management) | ✓ | ✗ | Atlassian CLI `acli`: [intro](https://developer.atlassian.com/cloud/acli/guides/introduction/) | [Platform REST v3](https://developer.atlassian.com/cloud/jira/platform/rest/v3/) · [Software (boards, sprints)](https://developer.atlassian.com/cloud/jira/software/rest/) · [Service Management](https://developer.atlassian.com/cloud/jira/service-desk/rest/) |
| Jira Data Center | ✓ | ✗ | none official | [REST](https://developer.atlassian.com/server/jira/platform/rest/) |
| Linear | ✓ | ✗ | none maintained (the official npm package is abandoned) | [Developers](https://linear.app/developers) · [GraphQL](https://linear.app/developers/graphql) |
| YouTrack | ✓ | ✗ | `youtrack-app`: [repo](https://github.com/JetBrains/youtrack-apps) (app tooling with a raw REST command) | [REST](https://www.jetbrains.com/help/youtrack/devportal/youtrack-rest-api.html) |
| Shortcut | ✓ | ✗ | none official | [REST v3](https://developer.shortcut.com/api/rest/v3) |
| Plane | ✓ | ✗ | none for work items (Prime CLI manages instances) | [Developers](https://developers.plane.so/) |
| Redmine | ✓ | ✗ | none official | [REST](https://www.redmine.org/projects/redmine/wiki/Rest_api) |
| OpenProject | ✓ | ✗ | none official | [API](https://www.openproject.org/docs/api/) |
| Taiga | ✓ | ✗ | none official | [API](https://docs.taiga.io/api.html) |
| Zoho Projects | ✓ | ✗ | none official | [REST](https://www.zoho.com/projects/help/rest-api/zohoprojectsapi.html) |

## Task managers

| System | WI | CR | Official CLI | API reference |
|---|---|---|---|---|
| Trello | ✓ | ✗ | none official | [REST](https://developer.atlassian.com/cloud/trello/rest/) · [auth](https://developer.atlassian.com/cloud/trello/guides/rest-api/authorization/) |
| Asana | ✓ | ✗ | none official | [Developers](https://developers.asana.com/docs) |
| ClickUp | ✓ | ✗ | none official | [API](https://developer.clickup.com/) |
| monday.com | ✓ | ✗ | `mcli`: [guide](https://developer.monday.com/api-reference/docs/mcli-cli-guide) (raw GraphQL `mcli query`) | [API reference](https://developer.monday.com/api-reference/) |
| Notion | ✓ | ✗ | `ntn`: [docs](https://developers.notion.com/cli/get-started/overview) | [API](https://developers.notion.com/) |
| Todoist | ✓ | ✗ | `td`: [repo](https://github.com/Doist/todoist-cli) | [API v1](https://developer.todoist.com/api/v1/) |
| Basecamp | ✓ | ✗ | `basecamp`: [repo](https://github.com/basecamp/basecamp-cli) | [API](https://github.com/basecamp/bc3-api) |
| Wrike | ✓ | ✗ | none official | [Developers](https://developers.wrike.com/) |
| Smartsheet | ✓ | ✗ | none official | [Developers](https://developers.smartsheet.com/) |
| Microsoft Planner / To Do | ✓ | ✗ | none dedicated (`az rest` can call Microsoft Graph) | [Planner in Graph](https://learn.microsoft.com/en-us/graph/api/resources/planner-overview) |
| Google Tasks | ✓ | ✗ | none official | [Tasks API](https://developers.google.com/workspace/tasks) |
| Taskwarrior | ✓ | ✗ | `task` is the tool itself (local data, `task export`) | [Docs](https://taskwarrior.org/docs/) |

## How to read this table

- **Official CLI first, but check what it is for.** Some official CLIs are app
  or instance tooling rather than a data client; the useful ones expose a raw
  API command (`gh api`, `glab api`, `mcli query`), which is what a collector
  should call.
- **Agent-oriented CLIs write.** Several new official CLIs are built for agents
  to create and update work. The collector uses only their read commands.
- **"None official" means go to the API** with a personal token, not to a
  community CLI, which needs the same token and adds a third party.
- **CR column empty** means the system has no code review: every change request
  state is dropped, and it is worth asking whether code lives in a second
  system.
- **Detection**: the binary names above (plus common community ones such as
  `jira` or `linear`) are evidence of which system the user works with, even
  when the collector will not use them.
