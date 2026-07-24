# performance-dudes

The workspace base repo for Performance Dudes partners and members.

**Clone this. Start Claude Code. The rest happens guided.**

## What this is

A base directory that contains all the Performance Dudes repositories as siblings. Claude Code uses the instructions in this repo to set you up — clone the right sub-repos, install and configure tools, set up the cockpit channel, and (on demand) generate keys and get your certificate signed — all based on what you can actually access on GitHub.

## Quick start

```bash
git clone git@github.com:performance-dudes/performance-dudes.git
cd performance-dudes
claude
```

Then in Claude Code:

> Set me up.

Claude reads [CLAUDE.md](CLAUDE.md), detects who you are from GitHub, and walks you through everything. You never pick a role.

## Want to read it yourself?

The full setup guide is **[docs/ONBOARDING.md](docs/ONBOARDING.md)** — read it top
to bottom if you'd rather see every step (discovering your repo access,
prerequisites, plugins, cloning, the cockpit channel).
[CLAUDE.md](CLAUDE.md) holds the rules that apply in *every* session — including
signing, the merge policy and commit conventions — and points Claude at the
onboarding guide when, and only when, it is actually needed. You
don't have to read either, though: **Claude can help with the entire setup** —
cloning repos, installing and configuring tools, the cockpit channel, signing, and
anything else. Just ask it, at any point.
