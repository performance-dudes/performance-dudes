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
| Founder | `trust`, `trust-keys`, `orga`, `pd` |
| Partner | `trust`, `trust-keys`, `pd` |
| Member | `trust`, `pd` |
| Exploring | `trust`, `pd` |

All repos clone as siblings to this one: `../trust`, `../pd`, etc.
But since the user is IN this repo when running Claude, clone them INTO this repo's directory (they are gitignored here):
- `./trust/`, `./trust-keys/`, `./orga/`, `./pd/`

## Plugin

This workspace ships with the `pd` plugin enabled at project scope (see `.claude/settings.json`). The marketplace is also declared there via `extraKnownMarketplaces`, so Claude Code discovers the plugin automatically on first launch from this directory — no manual `claude plugin marketplace add` step needed. The user will see a consent prompt when they first trust this workspace.

## Prerequisites check

Before cloning anything, verify the user has:

```bash
command -v git && git --version
command -v gh && gh --version
command -v uv && uv --version
command -v openssl && openssl version
gh auth status
```

If any are missing:
- `git`: `brew install git`
- `gh`: `brew install gh` then `gh auth login`
- `uv`: `brew install uv` (or `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- `openssl`: usually pre-installed on macOS

Run them via the Bash tool and report back.

## Clone the sub-repos

For each repo in the user's role list:

```bash
git clone git@github.com:performance-dudes/<repo>.git
```

If `git clone` fails for a private repo, the user doesn't have access — report clearly and move on.

## Setup by role

### Founder or Partner

Follow the setup in [`pd/README.md`](pd/README.md):

1. `cd pd && uv run scripts/setup.py --username <gh-user> --email <email> --trust ../trust`
2. Commit and push the CSR: `cd ../trust && git add pki/csrs/<user>.csr && git commit -m "feat: add CSR for <user>" && git push`
3. Trigger `pki-issue` workflow:
   ```bash
   gh workflow run pki-issue.yml --repo performance-dudes/trust \
     -f issuer=<ISSUING_CA_USERNAME> \
     -f csr_path=pki/csrs/<user>.csr
   ```
   - For a founder signing their own cert, `issuer` = their own GitHub username.
   - For a partner being onboarded, a founder has already run `pki-onboard` and issued the partner's Issuing CA. The partner then uses their own username as issuer.
4. After the workflow runs, approve the environment gate in the GitHub UI, merge the cert PR.
5. Pull the cert: `cd ../trust && git pull`
6. Extract a signature image: `uv run scripts/extract-signature.py <signed-pdf>` (user creates it with Preview first)
7. Run `./scripts/harden-signing.sh` — user runs this themselves (passphrase handling).

### Member

Same as partner but:
- No `trust-keys` access
- CSR must be signed by their partner's Issuing CA, not their own (the partner — or a founder — runs the `pki-issue` workflow with the partner's username as `issuer`)

### Just exploring

- Clone `trust` and `pd` (both public)
- Show them how to verify signed PDFs: `cd pd && uv run scripts/verify.py some-signed.pdf --trust ../trust`
- No key generation needed

## Things you should NOT do

- Run `harden-signing.sh` or anything that captures the passphrase (user must run it themselves)
- Modify the internal state of sub-repos (`./trust`, `./pd`, etc. each have their own CLAUDE.md)
- Install tools without asking first (especially if requiring sudo)
- Approve workflow runs on the user's behalf unless explicitly asked (you *can* approve via `gh api`, but ask first)

## Reference

- [trust README](https://github.com/performance-dudes/trust) — PKI setup, workflows
- [pd README](https://github.com/performance-dudes/pd) — signing scripts
- [cooperative story](https://github.com/performance-dudes/trust/blob/main/docs/cooperative.md) — why Performance Dudes runs a PKI
