# Claude Code Instructions — Performance Dudes Workspace Setup

You are assisting someone setting up their Performance Dudes workspace. The user has cloned this repo (`performance-dudes/performance-dudes`) and started Claude Code from inside it. Your job: get them fully operational. **There are no roles here** — nobody is boxed or limited by a "founder/partner/member" label. What someone can do follows entirely from which repos they can actually see on GitHub.

**Entry point:** when the user says **"set me up"** (or anything equivalent, or just
starts a fresh session here), run the **"Set me up" flow** below end-to-end. That
section is the spine; every other section is a detail it references.

## Principles

- **One command at a time.** Explain what you are about to do, run it, confirm it worked before the next step.
- **Never do anything that requires a secret in plain text visible to you.** If a passphrase is needed, ask the user to run the script themselves.
- **Idempotent.** If something is already set up, detect that and skip — don't redo.
- **Access from GitHub, not roles.** Detect the user via `gh auth status` and list the repos they can actually see (`gh repo list`). There is no role to ask about or assign — every person sees a different set of repos, and that set is the whole truth. Clone what they can see; do what each repo's own CLAUDE.md says.
- **Never modify sub-repo folders from here.** Each sub-repo has its own CLAUDE.md and its own rules.

## „Set me up" — the end-to-end flow

When the user asks to be set up, walk these steps **in order**, one command at a
time, idempotent (detect-and-skip what's already done), reporting after each. Each
step links to its detail section below. Do not silently install anything — ask
first (see „Things you should NOT do").

1. **Identity & accessible repos** — `gh auth status`, then list the PD repos the
   user can actually see in the org (`gh repo list`). That access list is the
   whole input — there is no role. → [Discover what the user can access](#discover-what-the-user-can-access)
2. **Clone what's visible** — clone exactly the repos `gh repo list` returned;
   only the public ones are known up front. → [The PD repos](#the-pd-repos)
3. **Prerequisites** — run the tool checks; install what's missing (ask first).
   → [Prerequisites check](#prerequisites-check)
4. **Pre-commit secret gate** — `lefthook install` (+ gitleaks).
   → [Pre-Commit Secret Prevention](#pre-commit-secret-prevention)
5. **Plugins** — register the user-scope marketplaces the user can reach; the
   workspace root enables `agent-sync@ai-plugins-internal` in
   `performance-dudes/.claude/settings.json`, and each sub-repo enables it in its
   own `<repo>/.claude/settings.json` (z.B. `be-plus/.claude/settings.json`) —
   Settings werden **nicht** aus Eltern-Verzeichnissen vererbt; `/reload-plugins`.
   → [Plugins](#plugins)
6. **Clone the sub-repos** — clone the visible repos, each following its own
   `## Setup default`. → [Clone the sub-repos](#clone-the-sub-repos)
7. **agent-sync — set up AND test** — install + configure its deps (signal-cli,
   cloudflared), link Signal, write the config, **start Claude with the channel
   flag** (plain `claude` does NOT carry the channel — see below), start the
   relay, and **prove it works** (`agent-sync status` + the probe →
   `confirm_channel`). → [agent-sync channel](#agent-sync-channel)
8. **Hand over** — show the user what they got, the structure, next steps; offer
   to explain agent-sync; invite questions, anytime. → [Final handover](#final-handover--orientation--offer-to-explain)

Stop and confirm at any step that needs a secret, an install, or a decision the
user owns. Never merge or sign on their behalf.

## Discover what the user can access

Setup is driven entirely by **which repos the user can see in the org**. Find them:

1. `gh auth status`; `gh api user -q .login` → their username.
2. **List the PD repos they can actually see — this is the whole input:**
   ```bash
   gh repo list performance-dudes --limit 200 --json name,visibility -q '.[].name'
   ```
   Org base permission is `none`, so what shows up is exactly what they may clone.
   Check a single repo with `gh repo view performance-dudes/<repo>` if unsure.
3. What someone can do follows from what they can see — no roles, no classification.
   Clone what's there and read each repo's own `CLAUDE.md` for the rest.

Greet by name and by what you found ("you can see these repos — let's set those
up").

## The PD repos

**Clone exactly what `gh repo list` returned** — every person sees a different
set, so there is no fixed clone list. Each repo follows its own `## Setup default`.

The **public** repos are the only ones knowable up front (everyone with org
access sees them):

| Public repo | What it is |
|---|---|
| `performance-dudes` | this workspace base repo (you're already in it) |
| `trust` | PKI trust anchors + verify path |
| `pd` | document signing scripts |
| `ai-plugins` | public Claude-Code plugins — a **marketplace**, registered not cloned (see „Plugins") |
| `website` | the Performance Dudes website |

Everything **else** the user sees in `gh repo list` is private and **varies per
person**. Don't assume which private repos exist; clone whatever the list shows
and read each one's `CLAUDE.md`. (Plugins like `agent-sync@ai-plugins-internal`
come via marketplaces, not as clones — see „Plugins".)

Clone everything as a sibling **inside** this repo's directory (they are
gitignored here): `./<repo>/` for each repo from the list, e.g. `./trust/`,
`./pd/`.

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
> — abgelöst durch die `ai-plugins*`-Marketplaces oben. **Neue Arbeit gehört in
> `ai-plugins-internal` (bzw. `ai-plugins`/`-enterprise`), nicht mehr in
> `pd`/`skills-private`.**
>
> `skills-private` ist **vollständig migriert und archiviert**: die Skills leben
> jetzt in `ai-plugins-internal` als `ai-first` (linniks-ai-first, ai-ready),
> `engineering` (linniks-coding-philosophy, linniks-vibe-engineering), `sales`
> (enterprise-angebot), `brand` (brand-uix) und `catchup` (im `workspace`-Plugin).
> `pd` wird weiterhin Plugin für Plugin migriert; solange eine Fähigkeit dort noch
> nicht migriert ist, zeigen einzelne Verweise unten übergangsweise weiter auf
> `pd` — das ist Legacy, nicht der Zielzustand.

**Setup (einmal pro Maschine) — Marketplaces im User-Scope registrieren:**

```bash
claude plugin marketplace add performance-dudes/ai-plugins
claude plugin marketplace add performance-dudes/ai-plugins-internal
claude plugin marketplace add performance-dudes/ai-plugins-enterprise   # nur mit Zugriff
```

Das landet in `~/.claude/settings.json` (`extraKnownMarketplaces`, github source)
— nicht pro Projekt, sondern einmal für alle. Private Marketplaces brauchen
GitHub-Org-Mitgliedschaft (wer keinen Zugriff hat, sieht sie schlicht nicht).

Welche Plugins in einem Projekt aktiv sind, steht in dessen **eigenem**
`<projekt-root>/.claude/settings.json` (`enabledPlugins`) — die Workspace-Wurzel
in `performance-dudes/.claude/settings.json`, ein Sub-Repo in
`be-plus/.claude/settings.json`. Es gibt **keine Vererbung aus
Eltern-Verzeichnissen**: startet Claude in `be-plus/`, liest es nur
`be-plus/.claude/…` plus das User-Scope `~/.claude/settings.json`, nicht das
`.claude/settings.json` des Eltern-Workspace. Beispiel:

```json
{ "enabledPlugins": { "agent-sync@ai-plugins-internal": true } }
```

`/reload-plugins` aktiviert. Plugins mit eigenem MCP-Server (z.B. `agent-sync`)
bringen ihre `.mcp.json` selbst mit — kein manuelles Wiring nötig.

**Externe Dependency: `context-mode`.** Das Plugin `context-aware@ai-plugins`
hängt von der **context-mode-MCP** ab (Retrieval über `ctx_*`, hält Roh-Bytes aus
dem Fenster). Die kommt aus einem **Drittanbieter-Marketplace** (`mksglu/context-mode`),
nicht aus den PD-Repos — sie muss separat registriert und installiert werden, sonst
läuft ein `context-mode`-Enable ins Leere und `ctx_*` fehlt. Nach der
PD-Plugin-Regel (plugin-hygiene): **im User-Scope registrieren + installieren,
dort deaktivieren, pro Projekt enablen.**

```bash
claude plugin marketplace add mksglu/context-mode
claude plugin install context-mode@context-mode   # auto-enabled im User-Scope -> danach disablen
```

Dann pro Projekt aktivieren (z.B. Workspace-Wurzel):
`{ "enabledPlugins": { "context-mode@context-mode": true, "context-aware@ai-plugins": true } }`,
`/reload-plugins`. Check: `/context-aware-doctor`.

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

For **agent-sync** (anyone with channel access) two more tools are needed — installed and
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

## Signing

When the user wants to sign a document or issue a certificate, follow
[`pd/README.md`](pd/README.md). `harden-signing.sh` führt der Mensch selbst aus
(Passphrase, siehe „Things you should NOT do").

## agent-sync channel

Wer Channel-Zugriff hat, koordiniert über die geteilte Signal-Gruppe direkt aus
Claude-Code-Sessions (Agent↔Agent + Signal an Menschen). Der Kanal ist das Plugin
`agent-sync@ai-plugins-internal`. Voller Walkthrough:
`ai-plugins-internal/agent-sync/server/README.md` — hier die Schritte als
Onboarding-Checkliste:

1. **Marketplace + Plugin** (siehe „Plugins"): `ai-plugins-internal` im User-Scope
   registriert, `agent-sync@ai-plugins-internal` im Workspace enabled.
2. **Voraussetzungen**: GitHub-Org-Mitgliedschaft (lässt Cloudflare Access durch)
   **und** Signal-Account in der „Performance-Dudes"-Gruppe.
3. **Tools**: `node` ≥ 20, `cloudflared` (`brew install cloudflared`), `signal-cli`
   (`brew install signal-cli`).
   - **`agent-sync`-CLI installieren** — als **Self-Update-Wrapper**, damit das CLI
     dem selbst-aktualisierenden Relay folgt (`~/.local/share/agent-sync/current`,
     das `agent-sync update` umlegt) statt an einem Git-Checkout zu lagen. Über
     `node` starten (die gebündelte `cli.mjs` trägt kein +x):
     ```bash
     printf '#!/bin/sh\nexec node "$HOME/.local/share/agent-sync/current/src/cli.mjs" "$@"\n' \
       > "$(brew --prefix)/bin/agent-sync" && chmod +x "$(brew --prefix)/bin/agent-sync"
     ```
     Der Install `~/.local/share/agent-sync/current` entsteht beim ersten
     Relay-Start (durch die MCP oder `agent-sync start`). **Nicht** `ln -s` auf einen
     Dev-Checkout — das lagt, weil `agent-sync update` nur den Relay aktualisiert.
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
7. **Claude mit dem Channel-Flag starten** — der Push-Channel braucht das
   development-channels-Flag. Leg dir **`cl` als deinen Claude-Alias** an
   (`~/.zshrc`), dann startest du Claude immer mit Channel:
   ```bash
   alias cl='claude --dangerously-load-development-channels plugin:agent-sync@ai-plugins-internal'
   ```
   Ab dann einfach `cl` statt `claude`.
8. **Relay starten + verifizieren**: `agent-sync start` (legt beim ersten Start die
   Config an, startet den Relay). Dann **beweisen, dass es läuft**: `agent-sync
   status` zeigt alles — Relay UP, signal/remote, deine Session, Channel/Flag.
   Beim MCP-Start kommt zudem eine `probe` → `confirm_channel` mit der nonce →
   `channel:on`.

**Self-Update:** der Relay hält sich selbst aktuell — `agent-sync start` zieht
vorher die zum Server passende Version (Config `autoUpdate`, **Default on**). Du
musst nichts pullen. Manuell: `agent-sync update`; abschalten:
`agent-sync autoupdate off`. Details: `agent-sync/CLAUDE.md` → „Versionen & Update".

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
- **Next steps** — e.g. start Claude with your `cl` alias (Claude + channel), try
  `agent-sync status`, and that PKI/signing is there **when you need it** (on demand).
- **Offer to explain agent-sync** — proactively offer a short walkthrough: how the
  channel works (agent↔agent + Signal), `@group` vs topics vs `label::name`, how to
  see who's working on what, and the collaboration skills (brainstorming, pairing,
  feedback-and-culture, mentoring). Point to the `agent-sync-usage` skill and
  `agent-sync/docs/collaboration.md`.
- **Invite questions, anytime** — make explicit that they can ask anything about the
  setup, the repos, or agent-sync at any point, now or later.

Keep it brief and welcoming — a map and an open door, not a wall of text.

## Projekt-Workspace-Repos (PWR)

Die PD-Produkte — Runway, Roger, cockpit, design-system, help-center — arbeiten alle auf
**Projekt-Workspace-Repos**. Welcher Ordner welchem Produkt gehört und wer liest oder schreibt,
steht in [`docs/pwr-contract.md`](docs/pwr-contract.md). Die harte Regel daraus:

> Kein Produkt schreibt Dateien in ein PWR. Dateien entstehen ausschliesslich über einen
> Merge Request — von Menschen oder Entwicklungsagenten.

Zugriff regelt der Vertrag **nicht**: es gilt weiter „Access from GitHub, not roles".

## Things you should NOT do

- Run `harden-signing.sh` or anything that captures the passphrase (user must run it themselves)
- Modify the internal state of sub-repos (`./trust`, `./pd`, etc. each have their own CLAUDE.md)
- Install tools without asking first (especially if requiring sudo)
- Approve workflow runs on the user's behalf unless explicitly asked (you *can* approve via `gh api`, but ask first)

## German language

In allen tracked Files dieses Repos und der Sub-Repos: echte Umlaute (`ä`, `ö`, `ü`, `Ä`, `Ö`, `Ü`, `ß`) statt ASCII-Ersatzschreibung (`ae`, `oe`, `ue`, `ss`). Gilt für Doku, Specs, Commit-Messages, PR-Bodies. Englische Fachbegriffe und Eigennamen bleiben unverändert (Maven, Auto-Merge, false). Wenn ein bestehendes Dokument ASCII-Schreibweise nutzt, im Rahmen einer regulären Änderung mit-fixen.

## Commits

- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `style:`.
- **Header:** What changed. Short, conventional commit format.
- **Body:** Why it changed. The broader context, intention, and motivation that cannot be derived from the diff. What problem was solved, what feedback triggered the change, what trade-off was made. Do not repeat what the diff shows. Write the context that would be lost without this message.

## Reference

- [trust README](https://github.com/performance-dudes/trust) — PKI setup, workflows
- [pd README](https://github.com/performance-dudes/pd) — signing scripts
- [cooperative story](https://github.com/performance-dudes/trust/blob/main/docs/cooperative.md) — why Performance Dudes runs a PKI
