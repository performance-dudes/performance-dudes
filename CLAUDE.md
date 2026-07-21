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
5. **Plugins** — register the user-scope marketplaces the user can reach.
   `agent-sync@ai-plugins-internal` bleibt im Workspace-Root **`false`**
   (Team-Entscheidung, siehe „Plugins" und „agent-sync channel") — jede
   Session entscheidet selbst per `cl`-Alias-Flag über Kanal-Teilnahme. Andere
   Plugins enablet jedes Sub-Repo in seiner eigenen
   `<repo>/.claude/settings.json` (z.B. `be-plus/.claude/settings.json`) —
   Settings werden **nicht** aus Eltern-Verzeichnissen vererbt; `/reload-plugins`.
   → [Plugins](#plugins)
6. **Clone the sub-repos** — clone the visible repos, each following its own
   `## Setup default`. → [Clone the sub-repos](#clone-the-sub-repos)
7. **agent-sync — currently on hold, cockpit ist der Nachfolger** — das
   Backing-Repo von `agent-sync` existiert nicht mehr; `cockpit` ist noch im
   Bootstrap-Status (kein Hub deployed, keine Binaries released). Prerequisites
   (signal-cli, cloudflared, Cloudflare Access) kann man trotzdem vorbereiten,
   der eigentliche Channel-Bootstrap ist aber aktuell nicht durchführbar.
   → [agent-sync channel](#agent-sync-channel)
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
{ "enabledPlugins": { "example-plugin@ai-plugins-internal": true } }
```

`/reload-plugins` aktiviert. Plugins mit eigenem MCP-Server (z.B. `agent-sync`)
bringen ihre `.mcp.json` selbst mit — kein manuelles Wiring nötig.

**Ausnahme `agent-sync@ai-plugins-internal`:** bleibt im Workspace-Root bewusst
`false` (PR #32, Felix) — der Channel soll nicht automatisch in jeder Session
mitlaufen, sondern pro Session gezielt über den `cl`-Alias geladen werden (siehe
„agent-sync channel"). Beim `claude plugin install` wird ein Plugin i.d.R.
automatisch im User-Scope (`~/.claude/settings.json`) auf `true` gesetzt — das
für `agent-sync@ai-plugins-internal` **danach wieder auf `false` korrigieren**,
sonst läuft der Channel künftig in jedem Projekt ohne eigene Override
automatisch mit.

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

> **Status (Stand 2026-07-21): pausiert, Nachfolger `cockpit`.** Das
> Backing-Repo `performance-dudes/agent-sync` (Relay-/CLI-/Server-Code) existiert
> nicht mehr — `gh repo view performance-dudes/agent-sync` scheitert, das Repo
> taucht auch nicht mehr in `gh repo list` auf. Nachfolger ist
> [`cockpit`](https://github.com/performance-dudes/cockpit) („enterprise-grade
> Rewrite von agent-sync"), aber `cockpit/README.md` zeigt **Status: Bootstrap**
> — Repo, Specs und Plan stehen, der Rewrite wird noch gebaut. Kein Hub deployed,
> keine Binaries released, keine End-User-Onboarding-Anleitung.
>
> **Praktisch heißt das:** Die Schritte unten bis einschließlich Cloudflare
> Access + signal-cli-Link funktionieren weiterhin (der alte Remote-Server unter
> `agent-sync.performance-dudes.com` läuft offenbar noch), aber der lokale
> CLI-Wrapper/Relay-Bootstrap (`agent-sync start`, `~/.local/share/agent-sync/current`)
> läuft ins Leere, weil es keine Quelle mehr gibt, von der das Relay-Bundle
> gezogen werden könnte. **Vor einem neuen Versuch immer zuerst
> `cockpit/README.md` auf einen aktuelleren Status prüfen** — sobald cockpit
> released ist, ersetzt es diesen ganzen Abschnitt.

Wer Channel-Zugriff hat, koordiniert über die geteilte Signal-Gruppe direkt aus
Claude-Code-Sessions (Agent↔Agent + Signal an Menschen). Der Kanal ist das Plugin
`agent-sync@ai-plugins-internal`. Solange kein cockpit-Release vorliegt, hier nur
die Schritte, die tatsächlich noch funktionieren (Vorbereitung, kein
vollständiger Bootstrap):

1. **Marketplace + Plugin registrieren** (siehe „Plugins"): `ai-plugins-internal`
   im User-Scope registrieren, Plugin explizit installieren
   (`claude plugin install agent-sync@ai-plugins-internal` — cached den Code,
   damit der Channel-Flag es überhaupt findet), **danach im User-Scope wieder
   auf `false` korrigieren** (siehe Ausnahme-Hinweis in „Plugins").
2. **Voraussetzungen**: GitHub-Org-Mitgliedschaft (lässt Cloudflare Access durch)
   **und** Signal-Account in der „Performance-Dudes"-Gruppe.
3. **Tools**: `node` ≥ 20, `cloudflared`, `signal-cli` (auf Linux ohne Homebrew:
   Distro-Paketmanager, z.B. `pacman`/AUR — `signal-cli` braucht zum Bauen UND
   Laufen mindestens Java 25, `archlinux-java set java-25-openjdk` falls der
   Build mit `UnsupportedClassVersionError` scheitert).
4. **Cloudflare Access**: `cloudflared access login https://agent-sync.performance-dudes.com`
   (Browser, GitHub-OAuth; erneuert sich danach ~monatlich selbst). Funktioniert
   aktuell noch.
5. **signal-cli linken**: `signal-cli link -n "<geraete-name>"` gibt einen
   `sgnl://linkdevice?...`-Link aus, **keinen** fertigen QR-Code im Terminal —
   selbst mit z.B. `qrencode` in ein Bild umwandeln und zeitnah scannen (das
   Link-Fenster ist kurz, ca. 1–2 Minuten); `signal-cli listAccounts` prüft
   danach die Verlinkung.
6. **Ab hier pausiert:** Config schreiben, `agent-sync start`, Channel-Flag-Test
   (`cl`-Alias) — der Relay-Bootstrap findet keine Quelle mehr und schlägt mit
   `Cannot find module '.../current/src/cli.mjs'` fehl. Nicht weiter versuchen,
   bis `cockpit` released ist; stattdessen den Stand dieses Abschnitts sowie
   `cockpit/README.md` erneut prüfen.

`AGENT_SYNC_USER_ALLOWLIST` war serverseitig bewusst leer (Autorisierungsgrenze =
Cloudflare Access). Architektur, Server-/Relay-Code, Deploy, ADRs lagen im
separaten Repo `performance-dudes/agent-sync` (jetzt nicht mehr vorhanden) —
für den Nachfolger siehe `cockpit/docs/architecture/0001-overview-and-decisions.md`.

## Final handover — orientation + offer to explain

When setup is done, don't just stop — orient the user. Give a short, friendly
wrap-up (adapt to what they actually got):

- **What you now have** — list the cloned repos with a one-line each and the
  active plugins the user's access reached. The agent-sync channel is on hold
  (see „agent-sync channel") — say plainly whether prep steps (signal-cli
  linked, Cloudflare Access logged in) succeeded, and that the actual channel
  waits on the `cockpit` release.
- **The structure** — everything is a sibling inside this workspace folder
  (`./trust/`, `./pd/`, `./cockpit/`, …), each repo with its own `CLAUDE.md` and
  rules. This repo is the workspace meta-repo.
- **Next steps** — e.g. that PKI/signing is there **when you need it** (on
  demand), and that the agent-sync channel returns once `cockpit` ships (check
  `cockpit/README.md`'s status line periodically).
- **Invite questions, anytime** — make explicit that they can ask anything about
  the setup or the repos at any point, now or later.

Keep it brief and welcoming — a map and an open door, not a wall of text.

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
