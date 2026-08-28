# A bug's saved diagnosis expires like a comment — re-measure before working it

An open bug carried a measured, correct root cause: "the blocker is missing photos — only
2 of 112 catalog items have one; migrating now would ship an empty storefront. Order:
photos first, migration second."

Three days later, re-measured against the source: **66 of 111 already had a photo**.
Another session had migrated photos in the meantime. And the real blocker — an
unassigned taxonomy field, 0 of 111 — was never written down, because the day the bug was
filed nobody had looked at that axis yet.

Working the bug from its saved root cause would have produced correct work on an
already-solved problem, and left the actual blocker untouched.

## The rule

**A bugs/blockers file is a log of DATED OBSERVATIONS, not facts.** Each entry captures
the state of the world on the day someone looked. In a repo worked across multiple
sessions or agents, that world keeps moving between when the bug is written and when it is
picked up.

**Before touching a bug more than a day old, re-run its measurement** and record the delta
in the bug itself — don't overwrite the old entry, since it still documents what was true
and when. If the number moved, re-read the proposed fix too: it is usually anchored to the
old number.
