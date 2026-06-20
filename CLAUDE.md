# Claude Code Instructions — Performance Dudes Workspace Setup

You are assisting a Performance Dudes partner, member, or new founder set up their local workspace. The user has cloned this repo (`performance-dudes/performance-dudes`) and started Claude Code from inside it. Your job: get them fully operational.

## Principles

- **One command at a time.** Explain what you are about to do, run it, confirm it worked before the next step.
- **Never do anything that requires a secret in plain text visible to you.** If a passphrase is needed, ask the user to run the script themselves.
- **Idempotent.** If something is already set up, detect that and skip — don't redo.
- **Derive role from GitHub — never ask.** Detect the user via `gh auth status` and their org membership/teams. The user must **not** be asked whether they are founder/partner/member. Clone only repos they actually have access to (verify with `gh`).
- **Never modify sub-repo folders from here.** Each sub-repo has its own CLAUDE.md and its own rules.

## Determine the role — automatically, do NOT ask

The user must never self-declare a role. Derive it from GitHub:

1. `gh auth status` (confirm login); `gh api user -q .login` → their username.
2. **Founder?** `gh api orgs/performance-dudes/memberships/<user> -q .role` → `admin` = Founder (org owner, full access).
3. Otherwise team membership: `gh api user/teams -q '.[] | select(.organization.login=="performance-dudes") | .slug'`
   - in `dudes` → **Dude** (internal team, full working access).
   - in `partners` → **Partner / customer**.
   - in neither → **external** (verify only, no setup needed).

Then welcome them by the derived role (e.g. „Erkenne dich als Dude — richte dein Setup ein.") and proceed. Greet, don't interrogate.

## Repos they need (derived from access)

Clone only what the user can access — the derivation above tells you which. Verify
per repo with `gh repo view performance-dudes/<repo>` if unsure (Org-Basis ist `none`,
also kommt Zugriff nur über Team-Grants).

| Derived role | Clone these |
|---|---|
| Founder (owner) | `trust`, `trust-keys`, `orga`, `pd`, `brand`, `culture`, `agent-sync` |
| Dude (`dudes`) | `trust`, `pd`, `brand`, `culture`, `agent-sync` |
| Partner (`partners`) | `trust`, `pd`, `agent-sync` |
| External | `trust`, `pd` (verify only) |

Plugins kommen **nicht** als geklonte Repos, sondern über User-Scope-Marketplaces
(siehe „Plugins") — `ai-plugins` / `ai-plugins-enterprise` / `ai-plugins-internal`
je nach Team-Zugriff.

`brand` is a private repo with ready-to-use brand assets (Teams backgrounds, logos, templates). Brand specs themselves live in the `brand-uix` skill (skills-private), which is the single source of truth.

All repos clone as siblings to this one: `../trust`, `../pd`, etc.
But since the user is IN this repo when running Claude, clone them INTO this repo's directory (they are gitignored here):
- `./trust/`, `./trust-keys/`, `./orga/`, `./pd/`, `./culture/`, `./brand/`, `./agent-sync/`

## Plugins

Claude-Code-Plugins kommen aus **Marketplace-Repos**. Modell: **Marketplace
einmal im User-Scope registrieren** (`~/.claude/settings.json` →
`extraKnownMarketplaces`, github source — maschinenweit, Code wird einmal
gecached), **Plugin pro Projekt enablen** (`<projekt>/.claude/settings.json` →
`enabledPlugins`, Eintrag `plugin@marketplace`).

Marketplace-Repos:

| Marketplace | Repo | Sichtbarkeit | Inhalt |
|---|---|---|---|
| `ai-plugins` | performance-dudes/ai-plugins | public | generische Infra (signing, workspace) |
| `ai-plugins-enterprise` | performance-dudes/ai-plugins-enterprise | privat | verkaufbare Produkt-Plugins |
| `ai-plugins-internal` | performance-dudes/ai-plugins-internal | privat | PD-intern (agent-sync, …) |

Migration aus den Alt-Plugins `pd`/`skills-private` läuft Plugin für Plugin;
bisher migriert: `agent-sync@ai-plugins-internal`.

**Setup (einmal pro Maschine) — Marketplaces im User-Scope registrieren:**

```bash
claude plugin marketplace add performance-dudes/ai-plugins
claude plugin marketplace add performance-dudes/ai-plugins-internal
claude plugin marketplace add performance-dudes/ai-plugins-enterprise   # nur mit Zugriff
```

Das landet in `~/.claude/settings.json` (`extraKnownMarketplaces`, github source)
— nicht pro Projekt, sondern einmal für alle. Private Marketplaces brauchen
Org-Mitgliedschaft (Team `dudes`/`partners`).

Welche Plugins **in diesem Workspace** aktiv sind, steht in
`<workspace>/.claude/settings.json` (`enabledPlugins`), z.B.:

```json
{ "enabledPlugins": { "agent-sync@ai-plugins-internal": true } }
```

`/reload-plugins` aktiviert. Plugins mit eigenem MCP-Server (z.B. `agent-sync`)
bringen ihre `.mcp.json` selbst mit — kein manuelles Wiring nötig.

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

For each repo the user has access to (derived above):

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

PKI-Onboarding (key/CSR/Cert, harden-signing) folgt [`pd/README.md`](pd/README.md). Pfad ergibt sich aus der oben abgeleiteten Rolle (nicht erfragen):

- **Founder**: signiert eigenen Cert (`issuer` = eigener GitHub-User), nach `pki-issue`-Workflow Cert-PR mergen.
- **Partner**: gleich wie Founder, aber ein Founder hat vorher `pki-onboard` für die Partner-Issuing-CA gefahren.
- **Dude**: kein `trust-keys`-Zugriff, CSR wird von einem Founder (oder Partner) mit dessen Issuing CA signiert (`pki-issue` mit dem Issuer-Username).
- **External**: nur Verify-Pfad, kein Key-Gen (`uv run scripts/verify.py <pdf> --trust ../trust` in `pd/`).

`harden-signing.sh` führt der Mensch selbst aus (Passphrase, siehe „Things you should NOT do").

## agent-sync channel (dudes + partners)

Dudes und Partner koordinieren über die geteilte Signal-Gruppe direkt aus
Claude-Code-Sessions (Agent↔Agent + Signal an Menschen). Der Kanal ist das Plugin
`agent-sync@ai-plugins-internal`. Voller Walkthrough:
`ai-plugins-internal/agent-sync/server/README.md` — hier die Schritte als
Onboarding-Checkliste:

1. **Marketplace + Plugin** (siehe „Plugins"): `ai-plugins-internal` im User-Scope
   registriert, `agent-sync@ai-plugins-internal` im Workspace enabled.
2. **Voraussetzungen** (ein Founder bestätigt sie): GitHub-Org-Mitglied (Team
   `dudes`/`partners` → Cloudflare Access lässt durch) **und** Signal-Account in
   der „Performance-Dudes"-Gruppe. Ohne 1. → jeder API-Call 401/403; ohne 2. →
   Receive-Pfad verwirft still.
3. **Tools**: `node` ≥ 20, `cloudflared` (`brew install cloudflared`), `signal-cli`
   (`brew install signal-cli`).
4. **Cloudflare Access**: `cloudflared access login https://agent-sync.performance-dudes.com`
   (Browser, GitHub-OAuth; erneuert sich danach ~monatlich selbst).
5. **signal-cli linken**: `signal-cli link -n "<mac-name>"` (QR mit Handy scannen),
   `signal-cli listAccounts` prüft.
6. **Config** `~/.config/pd/agent-sync.json` (Relay legt sie beim ersten
   `agent-sync start` an): `selfLabel`, `signal.account`, `identity`-Map
   (UUID→Label). `chmod 600`.
7. **Session mit Channel-Flag starten**:
   `claude --dangerously-load-development-channels plugin:agent-sync@ai-plugins-internal`.
8. **Verifizieren**: beim MCP-Start kommt eine `probe` → `confirm_channel` mit der
   nonce → `channel:on`. Smoke-Test: `POST http://127.0.0.1:8799/agent/send`.

`AGENT_SYNC_USER_ALLOWLIST` ist serverseitig bewusst leer (Autorisierungsgrenze =
Cloudflare Access). Architektur, Server-/Relay-Code, Deploy, ADRs: separates Repo
`performance-dudes/agent-sync`.

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
