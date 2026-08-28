# A test suite that exercises a REPLACE endpoint can wipe production, and stay invisible for months

A sync endpoint's contract is legitimately **replace, not upsert** — it deletes a mirror
table and reinserts the current projection, because a row that stops qualifying must
disappear (an upsert would keep it published forever). That is the correct design for the
endpoint. It is not a safe thing to call from a test suite pointed at the real database.

## The mechanism

The test suite called that endpoint against production. No bug was required — **exercising
the endpoint exactly as documented was enough**, and its cleanup step then left the table
empty. Every other test file in the same suite only cleaned up its own rows
(`DELETE ... WHERE name LIKE 'TEST-%'`), which felt safe and cannot work here: the
destructive step belongs to the endpoint itself, not to a fixture that could be scoped.

## Why it hid for months

Something else re-ran the same sync shortly after, for an unrelated reason, and the mirror
repopulated before anyone looked. The damage was real every time and healed on the usual
path — it only surfaced the one time nobody re-ran it. **A defect that self-heals on the
usual path is not rare, it's permanent with an intermittent symptom.**

## Two rules

1. **Any suite that can call a destructive/replace endpoint gets its own database**, not
   just its own row-namespace. Namespaced cleanup protects a suite from its own fixtures —
   it does nothing against an endpoint's own semantics.
2. **A test that requires pre-existing state is coupled to production**, not tested against
   a clean baseline. If a test needs an active record to exist, have it create one and
   delete it after; a suite that only passes against the live system is a description of
   that system, not a test of the code.

## The diagnostic trap this produces

A success log for the process that repopulated the table ("inserted 45 · success") is
telling the truth about what *that* run did — it is not evidence about the table's current
state, because something else emptied it a minute later by a different path. Check the
table. A log that says the table should be fine is not the same claim.
