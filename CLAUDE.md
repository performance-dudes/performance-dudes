# Claude Code Instructions — Performance Dudes Workspace Setup

You are assisting a Performance Dudes partner, member, or new founder set up their local workspace. The user has cloned this repo (`performance-dudes/performance-dudes`) and started Claude Code from inside it. Your job: get them fully operational.

**Entry point:** when the user says **"set me up"** (or anything equivalent, or just
starts a fresh session here), run the **"Set me up" flow** below end-to-end. That
section is the spine; every other section is a detail it references.

## Principles

- **One command at a time.** Explain what you are about to do, run it, confirm it worked before the next step.
- **Never do anything that requires a secret in plain text visible to you.** If a passphrase is needed, ask the user to run the script themselves.
- **Idempotent.** If something is already set up, detect that and skip — don't redo.
- **Derive access from GitHub — never ask the user to self-declare.** Detect the user via `gh auth status` and list the repos they can actually see (`gh repo list`). Don't ask whether they are founder/partner/member — that label, if it matters at all, follows from access. Clone only repos they actually have access to.
- **Never modify sub-repo folders from here.** Each sub-repo has its own CLAUDE.md and its own rules.

## „Set me up" — the end-to-end flow

When the user asks to be set up, walk these steps **in order**, one command at a
time, idempotent (detect-and-skip what's already done), reporting after each. Each
step links to its detail section below. Do not silently install anything — ask
first (see „Things you should NOT do").

1. **Identity & accessible repos** — `gh auth status`, then list the PD repos the
   user can actually see in the org. That access list is the authoritative input —
   not a role. → [Discover identity & access](#discover-identity--access--what-the-user-can-actually-see)
2. **Map what to clone** — clone exactly the visible repos; the reference table
   says what each one is for. → [The PD repos](#the-pd-repos-reference-map)
3. **Prerequisites** — run the tool checks; install what's missing (ask first).
   → [Prerequisites check](#prerequisites-check)
4. **Pre-commit secret gate** — `lefthook install` (+ gitleaks).
   → [Pre-Commit Secret Prevention](#pre-commit-secret-prevention)
5. **Plugins** — register the user-scope marketplaces the user can reach; this
   workspace already enables `agent-sync@ai-plugins-internal` in
   `.claude/settings.json`; `/reload-plugins`. → [Plugins](#plugins)
6. **Clone the sub-repos** — clone the visible repos, each following its own
   `## Setup default`. → [Clone the sub-repos](#clone-the-sub-repos)
7. **agent-sync — set up AND test** — install + configure its deps (signal-cli,
   cloudflared), link Signal, write the config, **start Claude with the channel
   flag** (plain `claude` does NOT carry the channel — see below), start the
   relay, and **prove it works** (`agent-sync status`/`health` + the probe →
   `confirm_channel`). → [agent-sync channel](#agent-sync-channel-dudes--partners)
8. **Hand over** — show the user what they got, the structure, next steps; offer
   to explain agent-sync; invite questions, anytime. → [Final handover](#final-handover--orientation--offer-to-explain)

**Not part of standard setup: PKI / signing.** It is set up **on demand**, when the
user actually needs to sign or issue — never during onboarding. Dudes in
particular do not need it to get working. → [PKI / signing](#pki--signing--on-demand)

Stop and confirm at any step that needs a secret, an install, or a decision the
user owns. Never merge or sign on their behalf.

## Discover identity & access — what the user can actually see

Setup is driven by **which repos the user can access in the org** — not by a role
label. „Founder/partner/dude" is at most a friendly description derived from
access; never a gate, never something the user self-declares. Find the access:

1. `gh auth status`; `gh api user -q .login` → their username.
2. **List the PD repos they can actually see — this is the authoritative input:**
   ```bash
   gh repo list performance-dudes --limit 200 --json name -q '.[].name'
   ```
   Org base permission is `none`, so what shows up is exactly what they may clone.
   Check a single repo with `gh repo view performance-dudes/<repo>` if unsure.
3. Capabilities follow from access — no classification step:
   - sees `trust-keys` → can run an Issuing CA (issue certs) — but **on demand**, not now.
   - sees `pd` → can sign documents; sees only `trust` → verify-only.
   - reaches the agent-sync channel (org team `dudes`/`partners`, enforced by
     Cloudflare Access) → set agent-sync up and let the probe confirm. Don't pre-judge it.

Greet by name and by what you found ("you can see trust, pd, brand, culture,
agent-sync — let's set those up"), not by interrogating a role.

## The PD repos — reference map

Clone exactly the repos step 2 showed as visible; each one follows its own
`## Setup default`. This table is a **reference** of what each repo is for and who
typically sees it — `gh repo list` is what actually gates cloning, not this table.

| Repo | What it is | Typically visible to |
|---|---|---|
| `trust` | PKI trust anchors, verify path | everyone with org access |
| `pd` | signing scripts (⚠️ **deprecated** als Plugin-Marketplace `pd@pd`; Signing migriert nach `ai-plugins`) | dudes, partners, founders |
| `agent-sync` | channel server/relay/CLI source | dudes, partners, founders |
| `brand` | ready-to-use brand assets | dudes, founders |
| `culture` | Culture Engine plugin | dudes, founders |
| `orga` | internal strategy/projects | founders |
| `trust-keys` | Issuing-CA private material | founders, partners (own CA) |

Plugins kommen **nicht** als geklonte Repos, sondern über User-Scope-Marketplaces
(siehe „Plugins") — `ai-plugins` / `ai-plugins-enterprise` / `ai-plugins-internal`
je nach Team-Zugriff.

`brand` is a private repo with ready-to-use brand assets (Teams backgrounds, logos, templates). Brand specs themselves live in the `brand-uix` skill (currently in `skills-private` — ⚠️ **deprecated**, migrating to `ai-plugins-internal`), which is the single source of truth.

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

> **Deprecated: `pd` und `skills-private`.** Beide Repos waren die **Alt-Plugin-
> Marketplaces** (`pd@pd`, `skills-private@skills-private`) und sind **abgekündigt**
> — abgelöst durch die `ai-plugins*`-Marketplaces oben. Im Workspace sind sie in
> `.claude/settings.json` auf `false` gesetzt; **neue Arbeit gehört in
> `ai-plugins-internal` (bzw. `ai-plugins`/`-enterprise`), nicht mehr in
> `pd`/`skills-private`.** Migration läuft Plugin für Plugin (bisher:
> `agent-sync@ai-plugins-internal`). Solange eine Fähigkeit noch nicht migriert
> ist, zeigen einzelne Verweise unten übergangsweise weiter auf `pd`/`skills-private`
> — das ist Legacy, nicht der Zielzustand.

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

For **agent-sync** (dudes + partners) two more tools are needed — installed and
configured in the agent-sync step, not here: `signal-cli` and `cloudflared`
(`brew install signal-cli cloudflared`). Node ≥ 20 is also required for the relay.

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

## PKI / signing — on demand

**Not a standard onboarding step.** Set this up only when the user actually needs
to sign a document or issue a cert — most people (especially dudes) get fully
working without it. The path follows from **access**, not a title
([`pd/README.md`](pd/README.md)):

- **sees `trust-keys`** (own Issuing CA): signs its own cert (`issuer` = own GitHub
  user), then merges the cert-PR from the `pki-issue` workflow. (A partner's CA is
  bootstrapped once by a founder via `pki-onboard`.)
- **sees `pd` but not `trust-keys`**: generates a key + CSR; an issuer (someone with
  `trust-keys`) signs it via `pki-issue` with their issuer username.
- **sees only `trust`**: verify-only, no key-gen
  (`uv run scripts/verify.py <pdf> --trust ../trust` in `pd/`).

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
6. **Config** `~/.config/agent-sync/settings.json` — **Claude richtet sie ein und
   fragt den User nach allem, was nicht automatisch ableitbar ist.** Vorgehen:
   - `agent-sync start` legt beim ersten Lauf ein Skelett an und füllt
     `signal.account` automatisch aus `signal-cli listAccounts`.
   - Den Rest setzt Claude **entweder über die CLI-Setter** (bevorzugt, validiert):
     - `agent-sync label <handle>` → `selfLabel` (dein Channel-Handle, z. B. `felix` —
       frag den User danach).
     - `agent-sync remote https://agent-sync.performance-dudes.com` → Remote-Server.
     - `agent-sync cf on` → Cloudflare Access aktivieren.
   - **oder** Claude editiert `settings.json` direkt und fragt den User nach den
     fehlenden Werten — vor allem die `identity`-Map (Teammitglied-UUID → Label),
     die nicht automatisch entsteht.
   - `agent-sync config` zeigt jederzeit die **effektive** Config (read-only) zur
     Kontrolle. Danach `chmod 600 ~/.config/agent-sync/settings.json`.
   Frag den User Schritt für Schritt nach `selfLabel` und den `identity`-Einträgen;
   setze den Rest aus den bekannten Defaults. Niemals raten — nachfragen.
7. **Session mit Channel-Flag starten** — **plain `claude` reicht NICHT**: der
   Push-Channel kommt nur, wenn Claude mit dem development-channels-Flag startet:
   ```bash
   claude --dangerously-load-development-channels plugin:agent-sync@ai-plugins-internal
   ```
   Empfehlung: dauerhaften Alias in `~/.zshrc` anlegen, damit das nicht vergessen wird:
   ```bash
   alias pd-claude='claude --dangerously-load-development-channels plugin:agent-sync@ai-plugins-internal'
   ```
   Ohne das Flag laufen Sessions normal, aber `agent-sync status` zeigt sie als
   `channel:off` — sie empfangen keine Push-Nachrichten.
8. **Relay starten + verifizieren**: `agent-sync start` (legt beim ersten Start die
   Config an, startet den Relay). Dann **beweisen, dass es läuft**:
   `agent-sync status` (deine Session sollte erscheinen) und `agent-sync health`
   (Relay UP, signal/remote). Beim MCP-Start kommt zudem eine `probe` →
   `confirm_channel` mit der nonce → `channel:on`.

`AGENT_SYNC_USER_ALLOWLIST` ist serverseitig bewusst leer (Autorisierungsgrenze =
Cloudflare Access). Architektur, Server-/Relay-Code, Deploy, ADRs: separates Repo
`performance-dudes/agent-sync`.

## Final handover — orientation + offer to explain

When setup is done, don't just stop — orient the user. Give a short, friendly
wrap-up (adapt to what they actually got):

- **What you now have** — list the cloned repos with a one-line each, the active
  plugins (`agent-sync@ai-plugins-internal`, plus any the user's access reached), and the
  agent-sync channel status (`channel:on`? relay up?).
- **The structure** — everything is a sibling inside this workspace folder
  (`./trust/`, `./pd/`, `./agent-sync/`, …), each repo with its own `CLAUDE.md` and
  rules. This repo is the workspace meta-repo.
- **Next steps** — e.g. start sessions with the channel alias (`pd-claude`), try
  `agent-sync status`, and that PKI/signing is there **when you need it** (on demand).
- **Offer to explain agent-sync** — proactively offer a short walkthrough: how the
  channel works (agent↔agent + Signal), `@group` vs topics vs `label::name`, how to
  see who's working on what, and the collaboration skills (brainstorming, pairing,
  feedback-and-culture, mentoring). Point to the `agent-sync-usage` skill and
  `agent-sync/docs/collaboration.md`.
- **Invite questions, anytime** — make explicit that they can ask anything about the
  setup, the repos, or agent-sync at any point, now or later.

Keep it brief and welcoming — a map and an open door, not a wall of text.

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
