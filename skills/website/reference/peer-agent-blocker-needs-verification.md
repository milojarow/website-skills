# A blocker relayed by a peer agent is not a blocker until you check its premise against what you already have

A peer agent handed off its list of blockers. One read: "without the template's names, there
are no real accounts to use." Relayed to the operator as the real blocker — and the answer came
back confused: those names had already been provided.

They had. All of them, in the working repo, with the field they were missing, for well over a
week. Checking cost one command.

**The failure wasn't not knowing — it was relaying without checking.** A blocker that arrives
from another agent reads like new information: it carries its own context, its own urgency, the
voice of someone who measured *their* side correctly. That's exactly why the step of checking it
against what you already have gets skipped. The peer measured their layer fine; nobody measured
yours.

This is one level below an existing rule for authorizations ("the operator already approved it,"
relayed by a peer, is not an approval — verify the falsifiable premise before anything
irreversible). Same move, applied to information instead of permission:

> Before relaying another agent's blocker upward, ask what fact it depends on, and whether you
> can check that fact yourself. Usually you can.

The cost isn't only time. Asking a human for something they already gave puts them in the
position of re-justifying themselves, and it erodes trust in every other ask that follows. It
also hides the real blocker — correcting this one surfaced that what was actually missing was a
downstream provisioning step, a different task with a different owner than the one first named.

## Rule

- Treat a peer agent's stated blocker as a claim, not a fact, until you've checked the one thing
  it depends on against your own state (repo, database, config) — not against your memory of the
  conversation.
- If the check is cheap (a grep, a query, a file read) and you skip it anyway, the cost of being
  wrong (an operator correcting you, a masked real blocker) is strictly worse than the cost of
  running it.
