---
name: qodo-fix
description: Investigate a Qodo code-review finding, reproduce it, propose a minimal fix, and open a PR for human approval.
---

# Qodo Finding → Fix Loop

You are handed one Qodo finding on a pull request or repository: a file, a
line range, and a description of the problem. Your job is to turn that
finding into a safe, minimal, verified fix — not to redesign the file.

## Steps

1. **Read the finding and the surrounding code.** Pull the flagged file plus
   enough surrounding context (calling code, tests, related modules) via the
   GitHub connector to understand what the finding actually means. Do not
   trust the one-line description alone — confirm the defect yourself.

2. **Reproduce it before fixing it.** Open a sandbox, check out the repo (or
   the PR branch), and write or run a test that demonstrates the bug failing.
   If a test already exists that should have caught this, run it and show it
   failing. Never propose a fix for something you haven't reproduced.

3. **Propose the smallest correct fix.** Fix only what the finding
   identifies. Do not refactor unrelated code, do not rename things, do not
   add speculative error handling elsewhere in the file. If fixing it
   properly requires touching more than the flagged lines, say so explicitly
   in the PR description rather than silently expanding scope.

4. **Verify the fix in the sandbox.** Re-run the reproduction test (it should
   now pass) and the existing test suite (it should not newly fail anything).
   Include both results in the PR description.

5. **Stop and ask before opening or merging anything.** Opening a PR is the
   sensitive action here — request human approval before pushing a branch or
   opening the PR, and never merge automatically. If the finding is a
   security issue (e.g. injection, secrets, auth), say so plainly in the
   approval request so the human knows to look closely before approving.

6. **Write the PR description for a human reviewer**, not for yourself:
   what Qodo flagged, what you confirmed, what changed, and the before/after
   test output. Link back to the Qodo finding.

## Guardrails

- If you cannot reproduce the finding, report that back instead of guessing
  at a fix — a fix for an unconfirmed bug is worse than no fix.
- If the finding is a false positive, explain why in your response instead
  of forcing a code change.
- Never touch files outside the ones the finding implicates without calling
  that out for approval first.
