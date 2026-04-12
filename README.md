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

> Set me up as a new Performance Dudes partner (or member, or founder).

Claude reads [CLAUDE.md](CLAUDE.md) and walks you through everything.

## The repo landscape

| Repo | Visibility | Contents |
|---|---|---|
| [`performance-dudes/performance-dudes`](https://github.com/performance-dudes/performance-dudes) (this one) | public | Workspace base, Claude setup instructions |
| [`performance-dudes/trust`](https://github.com/performance-dudes/trust) | public | PKI workflows, Root CA, Issuing CAs (certs only) |
| [`performance-dudes/pd`](https://github.com/performance-dudes/pd) | public | Document signing plugin and scripts |
| [`performance-dudes/trust-keys`](https://github.com/performance-dudes/trust-keys) | private | Encrypted CA key audit trail (founders + partners) |
| [`performance-dudes/orga`](https://github.com/performance-dudes/orga) | private | Strategy, decisions, concepts (founders) |

## Roles

- **Founder** — full access. Root CA ceremonies, 2-of-2 operations, partner onboarding.
- **Partner** — operates their own Issuing CA. Signs on behalf of Performance Dudes. Can issue certs to their team.
- **Member** — works for a partner. Has a personal signing certificate, signs documents.
- **Customer / external** — just verifies signatures. Needs no setup at all — public certs in `trust/` are enough.

Claude picks the right setup path based on your role.

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
