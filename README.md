# Fix Radar

Built for the **Agent Harness Hackathon** (Aug 24–30, 2026) — TrueForge +
Qodo.

Qodo reviews a pull request and flags a real issue. Fix Radar's agent picks
that finding up, reproduces the bug in an isolated sandbox, writes the
smallest correct fix, verifies it, and opens a PR for a human to approve
before anything merges. Qodo finds problems; the agent proves and fixes
them; a human stays the one who says yes.

This targets both prize tracks with one loop instead of two projects:
TrueForge provides the parts that make "reproduce → fix → verify → ask" a
real workflow (MCP tools, a sandbox, an approval gate) rather than a chat
reply, and Qodo is the thing that triggers the loop in the first place
rather than being bolted on for the submission.

## Architecture

```
Qodo review on a PR
        │  finding: file, lines, description
        ▼
TrueForge agent (this repo's saved agent + skill)
        │
        ├─ GitHub MCP connector ─── read the finding, the file, related code
        ├─ Sandbox (Daytona)    ─── check out the branch, reproduce the bug
        │                           with a failing test, apply the fix,
        │                           re-run tests
        └─ Approval gate        ─── human confirms before a branch is pushed
                                    or a PR is opened
        ▼
New PR: fix + before/after test output + link to the Qodo finding
```

The `skills/qodo-fix/SKILL.md` in this repo is the instruction pack the
agent loads for the actual investigate → reproduce → fix → verify → ask
loop. See that file for the exact steps and guardrails (smallest possible
fix, never merge automatically, say so if it's a false positive).

`../fix-radar-demo-target/` is the demo repository this agent runs against:
a tiny Flask API with two real, intentionally-left-in bugs (a SQL injection
and an unguarded division by zero) for Qodo to flag and the agent to fix on
camera.

## Setup checklist

Steps that need your own accounts/keys (can't be scripted):

- [ ] `npx @truefoundry/trueforge` → http://localhost:8790 (Node 22+
      confirmed installed)
- [ ] Settings → Models: add a provider + API key
- [ ] Settings → Connectors: add the GitHub MCP connector, scoped to the
      `fix-radar-demo-target` repo
- [ ] Settings → Skills: import this repo (or just `skills/qodo-fix/`) as a
      skill
- [ ] Settings → Sandbox providers: add a Daytona API key
- [ ] app.qodo.ai/signin → connect GitHub → install the Qodo app on
      `fix-radar-demo-target`
- [ ] Push `fix-radar-demo-target` to GitHub and open a PR against it (or
      just push to `main` — Qodo can review either) so Qodo has something to
      flag
- [ ] In TrueForge chat: enable the GitHub connector + sandbox + the
      `qodo-fix` skill, confirm it can read the Qodo finding, then **Save
      Agent**

## Demo script (for the submission video / blog post)

1. Show the two real bugs in `fix-radar-demo-target/app.py`.
2. Open a PR — Qodo flags the SQL injection and/or the ZeroDivisionError.
3. Fix Radar picks up the finding, reproduces it in the sandbox (show the
   failing test), fixes it, re-runs tests (show them passing).
4. Agent asks for approval before opening the fix PR — approve on camera.
5. Show the resulting PR: description, diff, before/after test output,
   link back to the Qodo finding.

## What actually happened (this run)

- Opened [PR #1](https://github.com/itjayesh/fix-radar-demo-target/pull/1)
  on `fix-radar-demo-target` adding a new `/expenses/total` endpoint.
- Commented `/agentic_review` — Qodo flagged it: the endpoint pulled every
  expense row into Python and summed them there instead of aggregating in
  SQL (medium-severity performance finding, `app.py` lines 66-69).
- Handed that finding to the saved TrueForge agent (GitHub MCP connector +
  Daytona sandbox + the `qodo-fix` skill). It read the PR and the file via
  GitHub, cloned the repo and installed dependencies in the sandbox,
  reproduced current behavior, rewrote the endpoint to use
  `SELECT COALESCE(SUM(amount), 0) FROM expenses`, added a regression test,
  ran `pytest -q test_app.py -k total` (passed), and correctly reported —
  without touching it — the pre-existing unrelated `ZeroDivisionError` bug
  in `average_amount([])`.
- **Stopped and asked for human approval** before pushing anything or
  opening a PR — the approval gate the `qodo-fix` skill's guardrails
  require.
- On approval, opened
  [PR #2](https://github.com/itjayesh/fix-radar-demo-target/pull/2) with
  the fix, the test, and a description linking back to the Qodo finding.

This is the whole loop end to end: Qodo finds a real issue → the agent
investigates and fixes it in a sandbox with its own tests → a human signs
off → the PR ships with the paper trail attached.
