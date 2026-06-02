# Claude Code Instructions — Performance Dudes Workspace Setup

You are assisting a Performance Dudes partner, member, or new founder set up their local workspace. The user has cloned this repo (`performance-dudes/performance-dudes`) and started Claude Code from inside it. Your job: get them fully operational.

## Principles

- **One command at a time.** Explain what you are about to do, run it, confirm it worked before the next step.
- **Never do anything that requires a secret in plain text visible to you.** If a passphrase is needed, ask the user to run the script themselves.
- **Idempotent.** If something is already set up, detect that and skip — don't redo.
- **Respect role.** Ask who they are up front, then follow the matching path. Don't try to clone repos they don't have access to without checking.
- **Never modify sub-repo folders from here.** Each sub-repo has its own CLAUDE.md and its own rules.

## Identifying the role

Start by asking:

> Welcome to Performance Dudes. What role are you taking here?
>
> 1. **Founder** — co-owner, full access to all repos, 2-of-2 operations
> 2. **Partner** — runs own Issuing CA, signs on behalf of PD
> 3. **Member** — works for a partner, needs a signing certificate
> 4. **Just exploring** — read-only, no signing setup
>
> Also: your GitHub username?

Remember the answer. Use it to decide which repos to clone.

## Repos they need (per role)

| Role | Clone these |
|---|---|
| Founder | `trust`, `trust-keys`, `orga`, `pd`, `skills-private`, `culture`, `brand` |
| Partner | `trust`, `trust-keys`, `pd`, `skills-private`, `brand` |
| Member | `trust`, `pd`, `brand` |
| Exploring | `trust`, `pd` |

`brand` is a private repo with ready-to-use brand assets (Teams backgrounds, logos, templates). Brand specs themselves live in the `brand-uix` skill (skills-private), which is the single source of truth.

All repos clone as siblings to this one: `../trust`, `../pd`, etc.
But since the user is IN this repo when running Claude, clone them INTO this repo's directory (they are gitignored here):
- `./trust/`, `./trust-keys/`, `./orga/`, `./pd/`, `./skills-private/`, `./culture/`, `./brand/`

## Plugins

This workspace ships with two Claude Code plugins declared in `.claude/settings.json`:

| Plugin | Repo | Visibility | Content |
|--------|------|------------|---------|
| `pd` | performance-dudes/pd | Public | Document signing, trust infrastructure |
| `skills-private` | performance-dudes/skills-private | Private | Brand, proposals, workflow automation |

Both are installed from local clones (not remote). After cloning the sub-repos, register and install:

```bash
claude plugin marketplace add ./pd
claude plugin install pd
claude plugin marketplace add ./skills-private
claude plugin install skills-private
```

Run `/reload-plugins` to activate. The `skills-private` repo requires org membership (private).

## Prerequisites check

Before cloning anything, verify the user has:

```bash
command -v git && git --version
command -v gh && gh --version
command -v uv && uv --version
command -v openssl && openssl version
command -v lefthook && lefthook version
command -v gitleaks && gitleaks version
gh auth status
```

If any are missing:
- `git`: `brew install git`
- `gh`: `brew install gh` then `gh auth login`
- `uv`: `brew install uv` (or `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- `openssl`: usually pre-installed on macOS
- `lefthook` + `gitleaks`: `brew install lefthook gitleaks` (siehe „Pre-Commit Secret Prevention" unten)

Run them via the Bash tool and report back.

## Pre-Commit Secret Prevention

Workspace-Hook (gitleaks + PD-Custom-Patterns) verhindert versehentliche Secret-Commits. Setup einmal pro Maschine: `brew install lefthook gitleaks && lefthook install`. Regeln und Bypass-Pfad: siehe `lefthook.yml` (kommentiert). Tatsächliche Secrets gehören in `~/.config/pd/` (gitignored). Sub-Repos haben eigene Gates, siehe deren `CLAUDE.md`.

## Clone the sub-repos

For each repo in the user's role list:

1. **Read the repo's own `CLAUDE.md`** for a `## Setup default` section. That
   section is the authoritative recommendation for how a fresh clone of that
   repo should be laid out.
2. Apply the declared default. If no default is declared, fall back to a
   classic clone.
3. **Ask the team member once** if they want to follow the declared default
   or use a different layout. Per-person preference always wins — the repo
   default is a recommendation, not a constraint.

**Classic clone:**

```bash
git clone git@github.com:performance-dudes/<repo>.git
```

**Worktree layout** (bare repo + `main` worktree + one worktree per active
PR — see "Setup-default patterns" below for the rationale):

```bash
mkdir <repo> && cd <repo>
git clone --bare git@github.com:performance-dudes/<repo>.git .bare
echo "gitdir: ./.bare" > .git
git -C .bare config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
git -C .bare fetch origin
git worktree add main main
cd ..
```

Resulting layout per worktree repo: `<repo>/.bare/` (bare repo), `<repo>/main/`
(main worktree), and one additional worktree per active PR
(`<repo>/pr<n>-<slug>/`).

If `git clone` fails for a private repo, the user doesn't have access —
report clearly and move on.

## Repo-defined setup defaults

Jedes Sub-Repo deklariert sein Setup-Layout in seinem `CLAUDE.md` unter `## Setup default`. Pattern ist entweder **classic** (`git clone`) oder **worktree** (bare + worktree pro PR). Ohne Deklaration: classic. Team-Member-Override geht jederzeit. Workspace hält keine zentrale Allowlist; was im Sub-Repo steht, gilt.

## Merge policy

**Never merge a PR without explicit user approval.** When work for a PR is
ready: open the PR with `gh pr create`, then hand the PR URL back to the user
and stop. Wait for a clear "merge" instruction before running `gh pr merge`.

Applies to every PD repo. Reason: merging is the user's call (timing, branch
hygiene, stacked-PR coordination); Claude's job is to land reviewable changes,
not to ship them.

The same rule applies to follow-up actions that are easy to confuse with
"finishing the PR": branch deletion, force-pushes that rewrite shared history,
release tagging. None of those happen without an explicit go-ahead.

## Setup by role

PKI-Onboarding (key/CSR/Cert, harden-signing) folgt [`pd/README.md`](pd/README.md). Rollen-Differenzen:

- **Founder**: signiert eigenen Cert (`issuer` = eigener GitHub-User), nach `pki-issue`-Workflow Cert-PR mergen.
- **Partner**: gleich wie Founder, aber ein Founder hat vorher `pki-onboard` für die Partner-Issuing-CA gefahren.
- **Member**: kein `trust-keys`-Zugriff, CSR wird vom Partner mit dessen Issuing CA signiert (Partner oder Founder triggert `pki-issue` mit Partner-Username als `issuer`).
- **Just exploring**: nur Verify-Pfad, kein Key-Gen (`uv run scripts/verify.py <pdf> --trust ../trust` in `pd/`).

`harden-signing.sh` führt der Mensch selbst aus (Passphrase, siehe „Things you should NOT do").

## agent-sync channel (optional, für Founder)

Founder können über die geteilte Signal-Gruppe in Claude-Sessions miteinander koordinieren. Setup-Walkthrough liegt im skills-private-Plugin: `skills-private/channels/agent-sync/README.md`. Voraussetzungen: GitHub-Org-Mitgliedschaft, eigenes signal-cli + Signal-Account in der PD-Gruppe, `cloudflared` für die Access-Auth. Architektur, Server- und Relay-Code, ADRs: separates Repo `performance-dudes/agent-sync`.

`pd-sync` läuft als Vorgänger noch parallel. Neue Setups gehen direkt auf agent-sync.

## Things you should NOT do

- Run `harden-signing.sh` or anything that captures the passphrase (user must run it themselves)
- Modify the internal state of sub-repos (`./trust`, `./pd`, etc. each have their own CLAUDE.md)
- Install tools without asking first (especially if requiring sudo)
- Approve workflow runs on the user's behalf unless explicitly asked (you *can* approve via `gh api`, but ask first)

## German language

In allen tracked Files dieses Repos und der Sub-Repos: echte Umlaute (`ä`, `ö`, `ü`, `Ä`, `Ö`, `Ü`, `ß`) statt ASCII-Ersatzschreibung (`ae`, `oe`, `ue`, `ss`). Gilt für Doku, Specs, Commit-Messages, PR-Bodies. Englische Fachbegriffe und Eigennamen bleiben unverändert (Maven, Auto-Merge, false). Wenn ein bestehendes Dokument ASCII-Schreibweise nutzt, im Rahmen einer regulären Änderung mit-fixen.

## Path conventions in shared docs

When writing in any tracked file of `performance-dudes/*` (this repo, orga, companions like be-plus, ...), refer to locations relative to the PD workspace root, not to a home directory. Use:

- `<pd-workspace>/<account>/` for companion paths
- `<companion>/<repo>/` for client-code clones inside a companion
- `../<sibling>/` for cross-references between sub-repos at the same level

`~/work/` and other home-directory paths are local to one user and must not appear in shared docs. Authority: `orga/decisions/2026-05-07-companion-workspace-location.md`.

## Commits

- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `style:`.
- **Header:** What changed. Short, conventional commit format.
- **Body:** Why it changed. The broader context, intention, and motivation that cannot be derived from the diff. What problem was solved, what feedback triggered the change, what trade-off was made. Do not repeat what the diff shows. Write the context that would be lost without this message.

## Reference

- [trust README](https://github.com/performance-dudes/trust) — PKI setup, workflows
- [pd README](https://github.com/performance-dudes/pd) — signing scripts
- [cooperative story](https://github.com/performance-dudes/trust/blob/main/docs/cooperative.md) — why Performance Dudes runs a PKI
