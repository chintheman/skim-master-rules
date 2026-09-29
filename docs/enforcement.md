# Deterministic enforcement

Prompt rules have a non-zero failure rate. A model can drift over a long
session and re-emit a second MINE/YOURS block, pad an empty label with
"nothing pending" instead of dropping it, or append a corrected block below
the wrong one instead of replacing it. None of that is a reasoning failure the
rules text can argue its way out of - it is a formatting contract, and a
formatting contract is cheap to check in code. That is the general principle:
where a rule, sequence, or output shape genuinely matters, enforce it
deterministically instead of asking the model to remember it every time.

The ownership block (see "The ownership block is an exception, not a footer"
in `rules/skim-master-rules.md`) is the one rule in this project worth that
treatment, because it is narrow, mechanical, and has a clear pass/fail check.

## The reference approach

One spec file defines the ownership block's format once. `ownership-format.json`
in this folder is a generic copy of it: the label names (MINE/YOURS), the rule
that a turn carries at most one block, the list of "empty" values a label is
never allowed to be padded with (`nothing`, `n/a`, `nothing pending`, and
similar), and the rule that a correction replaces the block rather than
appending a second one.

Two things read that same spec in the production setup this repo is mirrored
from, so they can never drift apart:

- **A Stop hook in Claude Code.** Runs after the assistant's final message for
  the turn, before the turn is allowed to end. It checks only that last
  message - earlier status updates in a multi-part turn are never judged -
  for two failure modes: two labels of the same kind stacked in one message,
  or a label padded with a no-op value from the spec's empty list. On a hit it
  blocks the turn and asks for the corrected two-line block only, never a
  full re-send of the message; re-sending the whole thing would just repeat
  the duplication the check exists to stop.
- **A pre-send formatter in the agent gateway.** A non-interactive agent
  framework runs the same check against the same spec before a message goes
  out over its own delivery channel, since a Stop hook is a Claude Code
  concept and has no equivalent there.

Neither piece decides *whether* a turn needs a block - that is still a
judgment call for the model, and a regex has no business making it. They only
catch the block being malformed once the model has already decided to send
one. That is the proportionate split: enforce the deterministic part (shape,
one instance, no padded values) in code, leave the judgment part (does this
turn need a block at all) to the prompt.

## Reuse it

Point your own Stop hook, or your own agent framework's pre-send check, at
`ownership-format.json` (or a copy of it) and it enforces the same four rules
without any code change on your side beyond reading the spec: one block per
turn, empty labels dropped rather than shown, no padded "nothing" values, and
a correction that replaces the block instead of stacking a second one below
it.

This repo does not ship the hook script itself - the reference implementation
lives in a private production setup and its source has internal paths and
history baked into its comments that do not belong in a public repo. The spec
file is enough to reimplement it: read the last assistant message of the
turn, apply `label_pattern` for each label in `labels`, block on more than
`max_blocks_per_turn` instances of the same label, and block when a label's
value (after stripping markdown) matches an entry in `empty_values`.
