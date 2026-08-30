# Fix Radar

Built for the **Agent Harness Hackathon** (Aug 24–30, 2026) — TrueForge +
Qodo.

Qodo reviews a pull request and flags a real issue. Fix Radar's agent picks
that finding up, reproduces the bug in an isolated sandbox, writes the
smallest correct fix, verifies it, and opens a PR for a human to approve
before anything merges. Qodo finds problems; the agent proves and fixes
them; a human stays the one who says yes.

**Live dashboard:** https://itjayesh.github.io/fix-radar/
**Blog post:** [BLOG.md](BLOG.md)
**The real run:** [Qodo's finding (PR #1)](https://github.com/itjayesh/fix-radar-demo-target/pull/1) → [the agent's fix (PR #2)](https://github.com/itjayesh/fix-radar-demo-target/pull/2)

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

[`fix-radar-demo-target`](https://github.com/itjayesh/fix-radar-demo-target)
is the demo repository this agent runs against: a tiny Flask API with two
real, intentionally-left-in bugs (a SQL injection and an unguarded division
by zero) for Qodo to flag and the agent to fix on camera.

## Setup (completed)

- [x] `npx @truefoundry/trueforge` → http://localhost:8790 — hit a
      Windows-only ESM crash ([upstream issue](https://github.com/truefoundry/trueforge/issues/427)),
      run inside WSL instead (see [BLOG.md](BLOG.md) for the detour)
- [x] Settings → Models: added a provider + API key
- [x] Settings → Connectors: added the GitHub MCP connector, scoped to
      `fix-radar-demo-target`
- [x] Settings → Skills: imported this repo's `skills/qodo-fix/` as a skill
- [x] Settings → Sandbox providers: added a Daytona API key
- [x] app.qodo.ai/signin → connected GitHub → installed the Qodo app on
      `fix-radar-demo-target`
- [x] Opened a PR on `fix-radar-demo-target`, commented `/agentic_review`
      to trigger Qodo
- [x] In TrueForge chat: enabled the GitHub connector + sandbox + the
      `qodo-fix` skill, handed it the real Qodo finding, approved the fix,
      **saved the agent**

`fix-radar-demo-target/app.py` also still has two bugs left in on purpose
(a SQL injection and an unguarded division by zero) that Qodo hasn't been
asked to review yet — untouched, available for anyone who wants to run the
loop again on a fresh finding.

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
