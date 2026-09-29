# Skim Master: output rules for AI agents

Two layers that stop AI assistants from writing walls of text:

1. **Always-on output governance**, a rules block (Scope → Rule 0 → Rules →
   Pre-Send Check → Off-switches) that shapes *every* chat reply: answer first,
   plain English instead of system vocabulary, substance over format, no preamble,
   no recap, no closers.
2. **On-demand overlay**, end any prompt with the word `skim` and the agent
   reformats *that one answer* (kill the wall, number the steps, bold the
   load-bearing words).

Built from months of real corrections, every rule traces to a specific moment
where a default AI answer wasted the reader's attention. Runs in production
across Hermes Agent (core), Claude Code, and the Claude app.

## What's inside

| Path | What it is |
|---|---|
| `rules/skim-master-rules.md` | **Canonical rules block.** Drop-in file for any agent with standing instructions (Claude Code, Hermes, custom system prompts) |
| `claude-app/instructions-for-claude.md` | Slim version for the Claude app's account-level "Instructions for Claude" (~1,500-char cap) |
| `examples/before-after-explained.md` | The same answer before/after, with the rule firing on every line explained |
| `docs/enforcement.md` | Why prompt rules alone aren't enough for the ownership block, and the hook + gateway-check pattern that backs it in production |
| `docs/ownership-format.json` | Generic copy of the shared spec a Stop hook and a pre-send formatter both read, so the two can never drift apart |
| `SKILL.md` | The `skim`-suffix overlay as an installable skill (`adhd` still fires it) |
| `LICENSE` | MIT |

## Install

**Claude Code** (governs every session):

```bash
mkdir -p ~/.claude/rules
cp rules/skim-master-rules.md ~/.claude/rules/
```

Files in `~/.claude/rules/` auto-load at session start, no import line, no
CLAUDE.md edit. (You can also append the block to `~/.claude/CLAUDE.md`
directly.) New sessions only; a running session keeps its start-of-session
context.

**Claude app** (claude.ai):

Settings → General → Profile → "Instructions for Claude" → paste the slim
version from `claude-app/instructions-for-claude.md`. Applies to new chats.
For full governance on a work project, paste the complete block into that
Project's instructions instead (~8,000-char cap there).

**Hermes Agent**: shipped in core. The config key keeps its legacy name, `agent.adhd_output_rules: true`, in
config.yaml (default on); `output_style: broadcast` suppresses it for
report-shaped sessions. `rules/skim-master-rules.md` mirrors the production
constant.

**Any other agent**: paste the block into its system prompt / custom
instructions equivalent.

**Ad-hoc, one answer**: end your prompt with the word `skim`, see `SKILL.md`
for trigger rules and the formatting overlay.

## The shape of it

The "after" answer in `examples/before-after-explained.md` is 40% shorter and
answers the question in its first line:

```
Before:  "This usually happens when your starter is hungry and has run out of
          food, causing it to produce excess alcohol and acidic byproducts..."
After:   "Short answer: it's hungry, not dead. Here's why + the 4-step fix."
```

Same facts. Better delivery. Nothing summarized away, the rules restructure,
they never cut substance (that's Rule 0's job).

## Mechanism notes (read before trusting it)

- **Prompt layer, not code.** The rules live in context and the model judges
  them per message. On hosts with a config flag (Hermes) that's a real switch;
  on Claude Code and the Claude app there is no code gate, the prompt is the
  whole mechanism. Reliable, not infallible.
- **Why not make it shorter?** Distilling the rules saves tokens but each line
  exists because a default answer failed without it. The full block costs
  roughly 850 tokens, loaded once per session, cached.
- **Explicit off-switches** (in the block): user asks for a specific format →
  that wins; producing a document/artifact/code → rules step aside inside the
  artifact; broadcast-style reports with their own templates → exempt.
- **On-demand ≠ import.** Claude Code `@imports` expand at launch (same context
  cost, just organization). True on-demand loading is *skills*, wrong for
  rules that must shape every reply.

## Deterministic enforcement

Prompt rules have a non-zero failure rate, so the ownership block (the
MINE/YOURS close) is not left to the prompt alone in production. A shared spec
defines its format once - one block per turn, empty labels dropped rather than
padded with "nothing," a correction that replaces the block instead of
stacking a second one - and two things check a message against it: a Claude
Code Stop hook, and a pre-send formatter in the agent gateway used outside
Claude Code. See `docs/enforcement.md` for the full writeup and
`docs/ownership-format.json` for a generic copy of the spec.

## Credit

- Original 10-rule lineage: [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd)
  (MIT) by jjacky, the Claude Code CLAUDE.md workflow this grew from.
- Structure patterns drawn from Ed Leeman's CLAUDE.md workflow.
- Playbook v2 (Scope, Rule 0, Pre-Send Check, Off-switches) refined through
  production use across Hermes Agent + Claude Code (Jul-Sep 2026), including an
  independent model review pass that fixed five real wording bugs.
- Sample "before" text is representative of default model output, not a quote.

## License

MIT
