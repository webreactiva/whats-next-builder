# Guided access setup

How to walk the user through configuring access when the CLI or the API needs
setup before the collector can read anything. Read it whenever step 2 of
`SKILL.md` does not end in a working identity call on the first try.

The user is building a skill, not studying their tracker's auth model. Every
wrong screen, wrong credential or surprise field costs them a round trip and
their trust in the rest of the process. Auth flows change without notice
(consoles get redesigned, OAuth replaces API keys, scopes get split), so what
you remember is the least reliable source you have.

## Contents

- [1. Research before instructing](#1-research-before-instructing)
- [2. Choose the credential](#2-choose-the-credential)
- [3. Show the whole path, then go one step at a time](#3-show-the-whole-path-then-go-one-step-at-a-time)
- [4. When the screen does not match](#4-when-the-screen-does-not-match)
- [5. Check every value the user brings back](#5-check-every-value-the-user-brings-back)
- [6. Secrets](#6-secrets)
- [7. Make it persist](#7-make-it-persist)
- [8. CLI-specific setup](#8-cli-specific-setup)
- [9. Leave the setup inside the generated skill](#9-leave-the-setup-inside-the-generated-skill)

## 1. Research before instructing

Before the first instruction, read:

- the system's **current** authentication docs (WebFetch the official page),
- its developer **changelog** for recent auth changes (a new OAuth flow, a
  deprecation, a console redesign),
- a quick search for **open issues** with new credentials (401s on fresh keys,
  scope bugs).

Note the date of what you read. If the docs and the user's screen disagree
later, the screen wins and the docs were stale.

## 2. Choose the credential

For a local, read-only collector, in this order:

1. **Official CLI login**: the CLI stores and refreshes credentials.
2. **Personal token** (API key + token, personal access token, API token) with
   the narrowest read-only scope.
3. **OAuth app flow** only when nothing else exists. It is built for
   user-facing apps: callback URLs, a browser consent, expiring tokens and
   refresh tokens stored somewhere. Say so before choosing it.

Community CLIs rarely help: they usually need the same token, and add a third
party with access to the account. Do not install one just to avoid a token.

## 3. Show the whole path, then go one step at a time

Start with an honest overview: how many steps, which prerequisites (register an
app or integration first, workspace or admin rights, a paid plan, 2FA, SSO or a
VPN for self-hosted instances), and roughly how long it takes. If it is more
involved than "copy a token", say it up front. Discovering it halfway feels
like being misled.

Then one step per message:

- **One action**: the exact URL, what to click, which fields to fill and which
  to leave empty.
- **What success looks like**: "you will see a page with a 32-character key".
- **Wait** for the user before the next step. Do not stack five steps and hope.

## 4. When the screen does not match

It will happen. The user reports a field you did not mention, a missing tab, a
required URL.

- Ask what they see (a screenshot is best) instead of guessing.
- Look it up again in the docs, with the exact label they quote.
- Explain what the field is for and whether this use case needs it. Required
  fields often belong to a path you do not need (an OAuth callback, an embedded
  app URL); look for the option that avoids them before inventing a value.
- If there is no way around it and the docs do not say what is accepted, say
  that plainly and propose the least risky value. Never present a guess as a
  fact.

## 5. Check every value the user brings back

Users paste the wrong credential in good faith: a client id instead of an API
key, a secret instead of a token.

- Recognise it by where it came from and by its shape (length, alphabet,
  prefix), and say which kind it looks like.
- Validate it immediately with one read-only call **and a known-invalid
  control**, so the error tells "invalid credential" apart from "valid but
  missing scope" and "valid, no access to this project".
- Only move on once the identity call (whoami) returns the user.

## 6. Secrets

- Say which values are **public** (client ids, and API keys in systems that
  treat them as public) and which are **secret** (tokens, client secrets,
  passwords).
- Ask for secrets to go into the shell profile or a secret manager, not the
  chat. Ask only for what the collector needs, never a client secret it will
  not use.
- If a secret lands in the chat anyway: do not write it to any file, use it only
  inline to validate, and tell the user once that the conversation is stored
  locally, so they should revoke or rotate it after testing, and where.
- Prefer short expirations for a trial run; say when the credential expires and
  how to renew it.

## 7. Make it persist

- Commands you run load the user's shell profile, but an `export` typed with
  `!` does not survive between commands. Credentials go into the profile
  (`~/.zshrc`, `~/.bashrc`) or a secret manager the collector can read.
- Tell the user the exact variable names the collector reads.
- Verify by name, never by value: `env | grep -c '^TRELLO_TOKEN='` style checks.

## 8. CLI-specific setup

- Install from the official docs for their OS and package manager.
- Interactive login runs in this session with `! <cli> auth login` (or the
  CLI's equivalent).
- A probe failing with "insufficient scope" usually has a CLI command to add
  scopes to the existing login; find it rather than asking for a new token.
- Self-hosted instances need the host configured in the CLI (and sometimes
  certificates); confirm with the CLI's status command against that host.
- Very old CLI versions may lack subcommands the collector needs; check
  `<cli> --version` and prefer its raw `api` subcommand, which ages best.

## 9. Leave the setup inside the generated skill

The credential will expire, a teammate will install the skill, the user will
change laptops. The generated `SKILL.md` gets a **Setup** section with:

- what access it needs (CLI login or which env vars) and the minimum scope,
- the verified steps to create or renew it, with the date they were checked,
- how to verify it works (the identity call),
- what the collector's auth errors mean and which step fixes each.
