# Onboarding — Performance Dudes Workspace

> **Read this file only when the gate in [`../CLAUDE.md`](../CLAUDE.md) says so.**
> In a configured workspace it is dead weight — it costs ~3,500 tokens of context
> and describes steps that are long done.

You are assisting someone setting up the Performance Dudes workspace. They have
cloned `performance-dudes/performance-dudes` and started Claude Code inside it. Your
job: get them fully productive. **There are no roles here** — nobody is constrained
by a "Founder/Partner/Member" label. What someone can do follows solely from which
repos they can actually see on GitHub.

## Principles for this flow

The standing rules from [`../CLAUDE.md`](../CLAUDE.md#how-to-work) apply — one
command at a time, never a secret in plain text, idempotent, access rather than
roles, install nothing without asking. Not duplicated here.

Additionally for this flow: **report after every step**, and stop at anything that
needs a secret, an installation, or a decision the human owns.

## The flow

These steps **in order**, one command per step, idempotent, reporting after each:

1. **Identity & visible repos** — `gh auth status`, then list the visible PD repos.
   → [Discover what the user can access](#discover-what-the-user-can-access)
2. **Clone what is visible** — exactly the repos from `gh repo list`.
   → [The PD repos](#the-pd-repos)
3. **Prerequisites** — run the tool checks, install what is missing (ask first).
   → [Prerequisites check](#prerequisites-check)
4. **Pre-commit secret gate** — `lefthook install` (+ gitleaks).
   → [Pre-commit secret prevention](#pre-commit-secret-prevention)
5. **Plugins** — register reachable marketplaces at user scope, enable per project.
   → [Plugins setup](#plugins-setup)
6. **Clone the sub-repos** — each following its own `## Setup default`.
   → [Clone the sub-repos](#clone-the-sub-repos)
7. **cockpit — set up AND test** → [cockpit channel](#cockpit-channel)
8. **Handover** — show what is there, explain the structure, invite questions.
   → [Final handover](#final-handover)

Stop at any step that needs a secret, an installation, or a decision the user owns.
Never merge or sign on their behalf.

## Discover what the user can access

Setup is driven entirely by **which repos the person can see in the org**:

1. `gh auth status`; `gh api user -q .login` → their username.
2. **List the visible PD repos — this is the whole input:**
   ```bash
   gh repo list performance-dudes --limit 200 --json name,visibility -q '.[].name'
   ```
   The org base permission is `none`; whatever shows up here is exactly what may be
   cloned. To check a single repo: `gh repo view performance-dudes/<repo>`.
3. What someone can do follows from what is visible — no roles, no classification.
   Clone what is there and read each repo's own `CLAUDE.md`.

Greet them by name and with what you found ("you can see these repos — let's set
them up").

## The PD repos

Which repos exist, which are public, and the fact that every sub-repo declares its
own layout is documented in [`../CLAUDE.md`](../CLAUDE.md#the-pd-repos) — that is
where it is maintained, not duplicated here.

For onboarding, only this matters: **clone exactly what `gh repo list` returned**.
Every person sees a different set; there is no fixed clone list. Do not guess which
private repos exist. (Plugins such as `cockpit@ai-plugins-internal` arrive through
marketplaces, not as clones.)

## Prerequisites check

```bash
command -v git && git --version
command -v gh && gh --version
command -v uv && uv --version
command -v openssl && openssl version
command -v lefthook && lefthook version
command -v gitleaks && gitleaks version
gh auth status
```

If something is missing:

- `git`: `brew install git`
- `gh`: `brew install gh`, then `gh auth login`
- `uv`: `brew install uv` (or `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- `openssl`: normally preinstalled on macOS
- `lefthook` + `gitleaks`: `brew install lefthook gitleaks`

For **cockpit**, add: the `cockpit` binary (set up in the cockpit step) and
**Node.js** — the plugin runs its hooks as `.mjs`.

## Pre-commit secret prevention

The workspace hook (gitleaks + PD custom patterns) prevents accidental secret
commits. Once per machine:

```bash
brew install lefthook gitleaks && lefthook install
```

Rules and the bypass path are documented in the comments of `lefthook.yml`. Real
secrets belong in `~/.config/pd/` (gitignored). Sub-repos have their own gates, see
their `CLAUDE.md`.

## Plugins setup

The model (marketplace at user scope, plugin per project) is described in
[`../CLAUDE.md`](../CLAUDE.md#plugins). Here are the one-time setup commands.

**Once per machine — register the marketplaces at user scope:**

```bash
claude plugin marketplace add performance-dudes/ai-plugins
claude plugin marketplace add performance-dudes/ai-plugins-internal
claude plugin marketplace add performance-dudes/ai-plugins-enterprise   # only with access
```

This lands in `~/.claude/settings.json` (`extraKnownMarketplaces`, github source) —
not per project, but once for all. Private marketplaces require GitHub org
membership (those without access simply do not see them).

Then enable per project, in that project's **own**
`<project-root>/.claude/settings.json`:

```json
{ "enabledPlugins": { "cockpit@ai-plugins-internal": true } }
```

`/reload-plugins` activates. Plugins with their own MCP server (e.g. `cockpit`)
bring their `.mcp.json` along — no manual wiring needed.

**External dependency: `context-mode`.** The plugin `context-aware@ai-plugins`
depends on the **context-mode MCP** (retrieval via `ctx_*`, keeps raw bytes out of
the window). It comes from a **third-party marketplace** (`mksglu/context-mode`),
not from the PD repos — register and install it separately, otherwise enabling
`context-mode` goes nowhere and `ctx_*` is missing. Following the PD plugin rule
(plugin-hygiene): register + install at user scope, disable it there, enable per
project.

```bash
claude plugin marketplace add mksglu/context-mode
claude plugin install context-mode@context-mode   # auto-enabled at user scope -> disable afterwards
```

Then activate per project, e.g. in the workspace root:
`{ "enabledPlugins": { "context-mode@context-mode": true, "context-aware@ai-plugins": true } }`,
`/reload-plugins`. Check: `/context-aware-doctor`.

## Clone the sub-repos

For every repo the person has access to: read the repo's own `CLAUDE.md`, apply the
`## Setup default` declared there, and **ask once** whether that is wanted. The
convention behind it is in [`../CLAUDE.md`](../CLAUDE.md#the-pd-repos).

> **Right after cloning, run `cat <repo>/CLAUDE.md` once** — and tell the person
> what it says. That file does **not** load automatically while the session runs in
> the workspace root (it sits one level below); skipping it means working against
> repo rules you have never seen. Why that is and what follows from it:
> [Never change sub-repos from here](../CLAUDE.md#never-change-sub-repos-from-here).

**Classic clone:**

```bash
git clone git@github.com:performance-dudes/<repo>.git
```

**Worktree layout** (bare repo + `main` worktree + one worktree per active PR):

```bash
mkdir <repo> && cd <repo>
git clone --bare git@github.com:performance-dudes/<repo>.git .bare
echo "gitdir: ./.bare" > .git
git -C .bare config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
git -C .bare fetch origin
git worktree add main main
cd ..
```

Result per worktree repo: `<repo>/.bare/` (bare repo), `<repo>/main/` (main
worktree) and one additional worktree per active PR (`<repo>/pr<n>-<slug>/`).

If `git clone` fails on a private repo, the person has no access — report that
clearly and move on.

The `## Setup default` convention itself (classic vs. worktree, who declares it,
whose preference wins) is documented in [`../CLAUDE.md`](../CLAUDE.md#the-pd-repos) —
it applies permanently, not just during setup.

## cockpit channel

Whoever has channel access coordinates through **cockpit** directly from Claude Code
sessions: agent↔agent, direct messages, presence, bridges into external channels.
The channel is the plugin `cockpit@ai-plugins-internal`; the authoritative
walkthrough is the plugin's `cockpit` skill. Here are the steps as an onboarding
checklist:

1. **Marketplace + plugin**: `ai-plugins-internal` registered at user scope,
   `cockpit@ai-plugins-internal` enabled in the project
   (`<project>/.claude/settings.json`).
2. **Prerequisite**: GitHub org membership in `performance-dudes` — the hub derives
   group membership from it.
3. **Install the `cockpit` binary:**
   ```bash
   curl -fsSL https://cockpit.performance-dudes.com/install | sh
   # Windows: curl -fsSL https://cockpit.performance-dudes.com/install.ps1 | powershell -c -
   ```
   If it is missing, the hook reports that quietly and starts nothing. The local
   **`cockpit node`** — the daemon, not Node.js — does **not** need to be started by
   hand: a hook checks it at SessionStart, UserPromptSubmit and before cockpit tool
   calls, and brings it up when needed.
4. **Log in once per hub:**
   ```bash
   cockpit login --remote https://cockpit.performance-dudes.com \
     [--provider github|signal] [--label <name>]
   ```
   The default is the **browser flow**; `--device` forces code-copying instead (for
   headless/SSH) — the login also falls back to it on its own when no browser can be
   opened. The JWT lands in `~/.config/cockpit/tokens.json` (0600, per remote).
   Until login the hub stays fail-closed (401 on `/send`/`/poll`) — that is correct.
   The `--label` is the channel handle (e.g. `benny`) — ask the user for it, do not
   guess.

   **Konto verknüpfen.** Group membership for org-bound projects (e.g.
   `performance-dudes`) is derived from the GitHub identity linked to the cockpit
   account — not from the login provider alone. Right after the first login,
   complete **„Konto verknüpfen"** with GitHub in the cockpit web UI
   (https://cockpit.performance-dudes.com). Skip it and topics stay invisible even
   though the group itself shows up (sticky-guest gap, tracked in
   `performance-dudes/cockpit#471`).
5. **Start Claude with the channel flag** — without the flag the channel stays
   reachable via `poll`, but nothing pops up on its own. The **flag is mandatory**,
   the alias **name is free** — `clc` below is just the convention this workspace
   settled on:
   ```bash
   alias clc='claude --dangerously-load-development-channels plugin:cockpit@ai-plugins-internal'
   ```
   Add it to `~/.zshrc`; from then on, `clc` instead of `claude`.
6. **Verify — prove that it works:**
   - `cockpit doctor` → end-to-end health check (node/auth/hub/bridge) with
     prioritised actions. The first thing to reach for when something is missing.
   - `cockpit status` → label, name, group of this session.
   - `cockpit ping` → checks **hub delivery** (session→node→hub→node→push).
   - At startup the bridge pushes a **probe** block with a nonce. If you see it,
     call `confirm_channel` **once** with the nonce — that proves local push
     rendering. To repeat manually: `probe_channel`.

   The two checks verify **different things**: `confirm_channel` only local
   rendering, `ping` the actual hub delivery. Both belong to the setup.

**When something jams** (dead node, 401 from the hub, nothing pops up): that is
operational knowledge and lives in
[`../CLAUDE.md`](../CLAUDE.md#cockpit--the-shared-channel) — it applies in every
session, not just during setup.

Architecture, server/node code, specs, ADRs: separate repo
`performance-dudes/cockpit`.

## Final handover

When setup is done, do not just stop — orient them. A short, friendly closing
(adapted to what was actually set up):

- **What you now have** — the cloned repos with one line each, the active plugins,
  and the cockpit status (node up? did `ping` get through?).
- **The structure** — everything sits as a sibling in this workspace folder
  (`./trust/`, `./pd/`, `./cockpit/`, …), each repo with its own `CLAUDE.md` and its
  own rules. This repo is the workspace meta repo.
- **Next steps** — start Claude with the `clc` alias, try `cockpit status`;
  PKI/signing is there **when it is needed** (on demand).
- **Offer to explain cockpit** — proactively offer a short walkthrough: how the
  channel works (agent↔agent + humans), `@group` vs. topics vs. `label::name`, how
  to see who is working on what, and the collaboration skills (brainstorming,
  pairing, feedback-and-culture, mentoring). Point at the `cockpit` and
  `channel-addressing` skills.
- **Invite questions, anytime** — make it explicit that anything can be asked at any
  time: setup, repos, cockpit.
