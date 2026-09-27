# Handoff — set up relay-bridge so this repo can receive automated handoffs

**Date:** 27 September 2026
**From:** Frontend team
**To:** Saeed / backend team

---

## What this is

We've built a small tool, **relay-bridge**, that lets the frontend and backend
teams hand work off to each other's Claude Code session automatically —
no more manually copying a handoff doc into a chat. When we finish something
you need to act on, we run one command; a GitHub Action in this repo picks it
up, runs your Claude Code against it, and opens a PR here for you to review.
You never have to be at your laptop for it to happen.

Full design + source: <https://github.com/SAQIBJH/relay-bridge> (private repo —
tell us your GitHub username if you want access, or just read this handoff,
it's self-contained).

**This PR already contains the one piece we could set up ourselves:**
`.github/workflows/handoff.yml`. Everything below is what's left, and it's
written so your own Claude Code — given permission — can run almost all of it
itself. Paste this file's path to your Claude Code session and say "do this,"
and it can execute every command below; the only things that need *you*
specifically are two browser approval clicks, which are called out clearly.

## What we could NOT do from our side, and why

We (SAQIBJH) only have **read** access to this repo, not write — verified via
the GitHub API, not assumed. That's exactly right and we didn't try to work
around it. So we opened this as a pull request from a fork instead of pushing
directly. You'll need to review and merge it, same as any other PR.

Two credentials in the steps below fundamentally require *your* subscription
or *your* click — nobody else can generate them for you:
- `CLAUDE_CODE_OAUTH_TOKEN` must come from **your own** Claude Pro/Max
  subscription login.
- Installing the Claude GitHub App and generating a fine-grained PAT both
  require a one-time browser approval — GitHub doesn't expose an API for
  either, by design.

## Step-by-step (run these in order; your Claude Code can execute all the commands, you just approve in the browser when one opens)

### 1. Review this PR

Look at `.github/workflows/handoff.yml`. It's configured with:
- `CHECKOUT_SOURCE_REPO: "true"` and `SOURCE_CHECKOUT_LINK: "../SMS-UI"` —
  matches this repo's own `CLAUDE.md` rule #1 (pull the frontend checkout
  before doing anything).
- `--allowedTools "Read,Write,Edit,Glob,Grep,Bash(git:*),Bash(gh:*)"` and
  `--permission-mode dontAsk` — this is **your** autonomy dial. Tighten or
  loosen it to whatever you're comfortable granting an unattended run. If you
  want it to also run your remote test suite (`make test` via
  `scripts/remote-test.sh`), add whatever `Bash(...)` pattern that needs, and
  see step 5 about the SSH key that needs.
- It never pushes to `main`/`master` directly — only ever opens a PR. That's
  enforced by the prompt, not by GitHub config, so **please add a branch
  protection rule on `main`** as the real backstop (Settings → Branches →
  Add rule). This was a deliberate, reviewed trade-off on our side, not an
  oversight.

### 2. Install the Claude GitHub App on this repo

```
/install-github-app
```

Run this in your Claude Code. It'll open a browser — **this is the one step
that needs you personally**: pick this repo (`saeedafri/SMS-BE`) and approve
the installation. `claude-code-action` (used in the workflow) authenticates
as this app by default, and it has to already be installed before the first
dispatch arrives.

### 3. Generate your subscription token

```bash
claude setup-token
```

This also opens a browser for you to approve — **second and last manual
click**. It prints a long-lived token in the terminal afterward. Your Claude
Code can capture that output and immediately run:

```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo saeedafri/SMS-BE
```

(paste the printed token when prompted, or pipe it in) — this needs you to
have `admin` on this repo, which as the repo owner you do.

### 4. `SOURCE_REPO_READ_TOKEN` — we'll send you this one, you just store it

**Correction from an earlier version of this doc:** this can't be something
you generate yourself — a fine-grained PAT can only be scoped to a repo the
*creator* has access to, and you don't have access to our frontend repo. So
**we're generating it** (Contents: Read-only, scoped only to
`SAQIBJH/sms-platform-frontend`) and will send you the value directly
(not over this PR — somewhere private, like Slack/DM).

Once you have it, either paste it to your Claude Code and let it run:

```bash
gh secret set SOURCE_REPO_READ_TOKEN --repo saeedafri/SMS-BE
```

or add it yourself under Settings → Secrets and variables → Actions → New
repository secret.

### 4b. What we need back from you: a dispatch token for this repo

This is the other half of the same problem, mirrored: for us to trigger this
workflow at all (fire `repository_dispatch` on `saeedafri/SMS-BE`), we need a
token with write access to *this* repo — and we don't have that access
ourselves (we only have read/pull, verified via the API). Only you can create
this one, the same way, in reverse:

1. Go to <https://github.com/settings/personal-access-tokens/new>
2. Token name: anything, e.g. `relay-bridge-dispatch`
3. Repository access → "Only select repositories" → `saeedafri/SMS-BE`
4. Permissions → Repository permissions → **Contents: Read and write**
5. Generate token, copy the value, send it to us the same private way (not
   as a PR comment)

We'll set it locally as `RELAY_BRIDGE_DISPATCH_TOKEN` when we run
`send_handoff.py`. Nothing works end-to-end until we have this.

### 5. (Optional) test-tunnel SSH key, only if you want the remote suite in CI

If you widen `--allowedTools` in step 1 to let the agent run
`scripts/remote-test.sh` / `make test`, it'll need SSH access to the AWS box
to reach Postgres/ClickHouse/Redis. **Don't reuse `deploy.yml`'s
`AWS_SSH_KEY`** — mint a separate key/user scoped to read/tunnel access only,
so an autonomous run can never hold the credential that restarts
`relay-api` in production. Add it as its own secret and reference it in the
workflow.

### 6. (Optional) `PR_TOKEN`

GitHub doesn't run this repo's own `pull_request`-triggered CI/review
workflows on a PR opened with the default `GITHUB_TOKEN`. If you want your
usual CI/Codex review to fire automatically on the PRs this workflow opens,
add a PAT with contents+PR write on this repo as secret `PR_TOKEN`. If you
skip this, the PRs still get opened fine — your CI just won't auto-run on
them; you can trigger it manually.

### 7. Merge this PR

Once secrets are in place, merge it. The workflow is live from that point.

## How to test it

Ask the frontend team (us) to send a trivial test handoff once you've merged
and added the secrets — we'll run `send_handoff.py` and you should see a PR
appear here within a minute or two of the dispatch firing. Check the Actions
tab if it doesn't.

## Once this direction works, the reverse direction is symmetric

If you also want to send handoffs from here to the frontend repo, the same
toolkit does it — you'd need `send_handoff.py` from
<https://github.com/SAQIBJH/relay-bridge> on your machine, a PAT with write
access to `SAQIBJH/sms-platform-frontend` to fire the dispatch, and the
frontend repo would need its own `handoff-workflow.yml` + secrets (mirroring
steps 2–6 above, but on their side). Ask us when you're ready and we'll set
up our end.
