# performance-dudes

The workspace base repo for Performance Dudes partners and members.

**Clone this. Start Claude Code. The rest happens guided.**

## What this is

A base directory that contains all the Performance Dudes repositories as siblings. Claude Code uses the instructions in this repo to set you up — clone the right sub-repos, install tools, generate keys, get your certificate signed, configure signing with Touch ID — all according to your role.

## Quick start

```bash
git clone git@github.com:performance-dudes/performance-dudes.git
cd performance-dudes
claude
```

Then in Claude Code:

> Set me up.

Claude reads [CLAUDE.md](CLAUDE.md), detects who you are from GitHub, and walks you through everything. You never pick a role.

## The repo landscape

| Repo | Visibility | Contents |
|---|---|---|
| [`performance-dudes/performance-dudes`](https://github.com/performance-dudes/performance-dudes) (this one) | public | Workspace base, Claude setup instructions |
| [`performance-dudes/trust`](https://github.com/performance-dudes/trust) | public | PKI workflows, Root CA, Issuing CAs (certs only) |
| [`performance-dudes/pd`](https://github.com/performance-dudes/pd) | public | Document signing plugin and scripts |
| [`performance-dudes/trust-keys`](https://github.com/performance-dudes/trust-keys) | private | Encrypted CA key audit trail (founders + partners) |
| [`performance-dudes/orga`](https://github.com/performance-dudes/orga) | private | Strategy, decisions, concepts (founders) |

## Roles — derived from GitHub, not asked

You never declare a role. Claude detects your GitHub identity and org team
membership (owner / `dudes` / `partners`) and sets up exactly what you can access.

- **Founder** (org owner) — full access. Root CA ceremonies, 2-of-2 operations, partner onboarding.
- **Dude** (`dudes` team) — internal team. Personal signing certificate, full working access.
- **Partner** (`partners` team) — operates their own Issuing CA. Signs on behalf of Performance Dudes. Can issue certs to their team.
- **External / customer** — just verifies signatures. Needs no setup at all — public certs in `trust/` are enough.

## Layout after setup

```
performance-dudes/                 ← this repo
├── CLAUDE.md
├── README.md
├── orga/                          ← cloned sub-repo (if you have access)
├── trust/                         ← cloned sub-repo
├── trust-keys/                    ← cloned sub-repo (if partner or founder)
└── pd/                            ← cloned sub-repo
```

The sub-repos are `.gitignored` in this repo — they live here for convenience but are managed independently.
