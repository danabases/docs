# AI Model Recommendation for the SQL Audit Deployment Project

## Current model landscape for GitHub Copilot in VS Code (April 2026)

Copilot Chat in VS Code currently offers, among others, Claude Opus 4.7
(at a promotional 7.5x request multiplier until April 30, 2026), Claude
Opus 4.6 and Sonnet 4.6, various GPT-5 variants, and Gemini. The VS Code
docs themselves suggest using a fast model like GPT-5 Mini for quick
edits and simple questions, and a reasoning model like Claude Opus for
complex refactoring and architectural decisions. That aligns with how
I’d split this project’s workload.

## My specific recommendation for this project

For the architecture work, compliance reasoning, and anything involving
the `xp_cmdshell` restoration logic or AG coordination — use
**Claude Opus 4.7**. The scenarios where one wrong assumption costs
you a production incident (leaving `xp_cmdshell` enabled, mismatched
AG GUIDs surfacing on failover six months from now) are exactly where
a reasoning model earns its keep. The 7.5x request multiplier stings
but the promotional rate runs through April 30; after that it’ll drop
to standard pricing.

For the iterative implementation and debugging — **Claude Sonnet 4.6**.
Solid code generation, faster responses, reasonable cost. This is your
daily driver for Tasks 1–6 in the prompt file. Set thinking effort to
“High” for the harder bits.

For inline autocomplete (as-you-type suggestions) — whatever your
Copilot plan gives you by default is probably fine. Inline suggestions
benefit less from frontier reasoning since the context is narrow by
design. Don’t burn premium request multipliers on inline.

## A couple of alternatives worth knowing about

**Claude Code** — Anthropic’s terminal-based coding agent, separate
from VS Code Copilot. Runs from your command line, can read your entire
workspace, plan multi-file changes, and execute them with your approval.
Strong fit for this project because it respects architectural decisions
across long sessions and can run your scripts directly to validate
syntax. If you’d rather work in a terminal alongside VS Code than
entirely inside Copilot Chat, this is worth trying. Available via
Anthropic subscription or API.

**The direct API route** — If your org has restrictions on what data
goes through GitHub’s Copilot infrastructure, you can hit the Anthropic
API directly from VS Code via community extensions, or use Claude.ai’s
web interface with files uploaded. Slower workflow than Copilot inline
suggestions but gives you full control over what leaves your network.
For a compliance-sensitive project where the code itself references
production server names and infrastructure details, this is worth
considering.

## Practical workflow suggestion

Keep three windows open when working on this:

1. **VS Code with Copilot Chat set to Sonnet 4.6** — primary
   implementation, inline tweaks
1. **A separate Claude chat (claude.ai or Opus 4.7 in Copilot)** —
   for architecture questions, code reviews against the prompt file’s
   “do not” list, and design decisions. Start each session by pasting
   the prompt file
1. **A terminal** — for actually running the scripts against your test
   CMS group and iterating on real failures

The key trick: paste the Copilot prompt file as the first message in
every new Claude session. Long-running projects like this one drift
without it — the model starts forgetting why StrictMode is 2.0 not
Latest, or suggests parallelization you already rejected, or reaches
for `STRING_AGG` because it’s cleaner. The prompt file is your hedge
against that drift.

## One honest caveat about me specifically

I’m obviously not a neutral voice here — I’m Claude recommending Claude.
Take the recommendation with that grain of salt. What I can say with
more confidence is the split strategy: use a reasoning-heavy model for
architecture and a faster model for implementation, regardless of whose
model each turns out to be. If you want to sanity-check my
recommendation, run the same architecture question (say, “review the
`xp_cmdshell` restore logic in v3 for failure modes”) through GPT-5.4
and Claude Opus 4.7 and see which response catches more real issues.
The one that finds things the other missed is the one worth using for
hard problems.