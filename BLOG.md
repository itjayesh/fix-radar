# Fix Radar: teaching a code reviewer's findings to fix themselves

*Built solo for the Agent Harness Hackathon, Aug 24–30, 2026.*

## The idea

Most AI coding demos stop at "the model wrote some code." The interesting
part of this hackathon was supposed to start after that — can an agent
reach real tools, run its own code somewhere safe, and know when to stop
and ask a human before it does something that matters?

I wanted a project that used TrueForge (the harness) and Qodo (the
reviewer) as one loop instead of two separate boxes to check. So: **Qodo
finds a real problem on a pull request. A TrueForge agent investigates it,
reproduces it in a sandbox, writes a fix and a test, and then stops —
it doesn't push anything until a human says yes.** Qodo supplies the
"what's wrong," TrueForge supplies the "safely do something about it."

I built a tiny deliberately-imperfect Flask app
([`fix-radar-demo-target`](https://github.com/itjayesh/fix-radar-demo-target))
to be the thing under review, and a skill
([`qodo-fix/SKILL.md`](https://github.com/itjayesh/fix-radar/blob/master/skills/qodo-fix/SKILL.md))
that tells the agent exactly how to behave: investigate before trusting
the finding, reproduce before fixing, fix only what was flagged, and
never open or merge anything without asking first.

## The part nobody warns you about: it's day one software

`npx @truefoundry/trueforge` crashed immediately on Windows with an ESM
loader error — `Only URLs with a scheme in: file, data, and node are
supported`. I went looking for a fix and found the exact bug already
filed on GitHub, opened by someone else *that same day*, still unresolved.
TrueForge's own sandbox layer says outright it only supports macOS and
Linux. Windows is clearly a second-class citizen for this tool right now.

The fix wasn't glamorous: run it inside WSL instead of native Windows,
where it matches the platforms it actually supports. That surfaced its own
small comedy — Git Bash silently mangles `/mnt/c/...` paths and drops
shell variables across semicolons when you pipe commands through `wsl.exe`
from it, so anything nontrivial had to go through an actual script file
with `MSYS_NO_PATHCONV=1` set, not an inline one-liner. None of this is
interesting on its own, but it's the realistic cost of building on a tool
that's four days old in its current form, and it's worth saying plainly
instead of pretending the setup was smooth.

## Getting Qodo to actually say something

The first attempt at triggering a review taught me something obvious in
hindsight: Qodo reviews *diffs*, not vibes. My first PR was an empty
commit meant just to "trigger a review" — Qodo correctly said there was
nothing to review. It also doesn't review automatically; you have to
comment `/agentic_review` on the PR yourself. Once I pushed a PR with an
actual code change, it came back with a real finding: a new
`/expenses/total` endpoint was pulling every row into Python and summing
there instead of letting SQLite aggregate — a genuine, unscripted
performance bug, not one I'd planted on purpose.

## Handing it to the agent

I gave the TrueForge agent that exact finding, with the GitHub MCP
connector, a Daytona sandbox, and the `qodo-fix` skill enabled. What it
did, unprompted beyond the skill's instructions:

1. Read the PR and the file through GitHub tools instead of trusting the
   one-line description.
2. Cloned the repo into the sandbox, installed dependencies, ran the
   existing tests first.
3. Rewrote the endpoint to use `SELECT COALESCE(SUM(amount), 0) FROM
   expenses`, added a regression test, and ran it — passed.
4. Noticed a completely unrelated bug sitting a few lines away (a
   `ZeroDivisionError` on an empty table) and **explicitly said it wasn't
   touching it**, because it wasn't what was flagged.
5. Tried to push the fix — and hit a real `403 Resource not accessible by
   personal access token`, because the GitHub token I'd wired up only had
   read access. It reported the exact error and waited instead of
   quietly failing or trying something clever.
6. Once I fixed the token's permissions and told it to go ahead, it opened
   [PR #2](https://github.com/itjayesh/fix-radar-demo-target/pull/2) with
   the fix, the test, and a description linking back to the original
   finding.

That permission failure ended up being the most convincing part of the
whole run. A scripted demo doesn't hit a real 403 and handle it
correctly — this did, because the approval boundary and the tool
permissions are actually enforced, not decorative.

## What I'd tell someone starting this hackathon today

- Budget real time for the harness itself being early-stage, especially
  off Linux/macOS — check open GitHub issues before assuming a crash is
  your own config.
- Trigger your review tool's automation manually at least once before you
  assume it's automatic. Empty diffs teach you nothing.
- The approval gate is the actual feature, not a UX nicety. The moment
  worth showing a judge isn't the fix — it's the agent stopping and
  refusing to guess its way around a real permission wall.

## The result

[Fix Radar dashboard](https://itjayesh.github.io/fix-radar/) ·
[repo](https://github.com/itjayesh/fix-radar) ·
[demo target](https://github.com/itjayesh/fix-radar-demo-target) ·
[the finding](https://github.com/itjayesh/fix-radar-demo-target/pull/1) ·
[the fix](https://github.com/itjayesh/fix-radar-demo-target/pull/2)
