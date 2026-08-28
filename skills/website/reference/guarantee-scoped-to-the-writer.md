# A documented guarantee belongs to the code PATH that implements it, not to the table

A dedup/validation rule described in a doc ("the insert dedups by X") is easy to read as a
property of the resource (the table, the collection) when it is actually a property of
**one writer** against that resource. A second writer touching the same table does not
inherit it.

## The failure shape

    doc says:   "the INSERT dedups by commit, or by (project, date, title)"
    true for:   one hand-written SQL script
    NOT true for: POST /api/changes — a bare INSERT, no dedup at all

An agent reads the doc, tells the user "running this twice is harmless, the backend
dedups," and ships a route that duplicates every row on a retry. It never opened the
handler it was actually calling — the sentence described a code path, and got read as a
property of the table.

Two writers on the same table share exactly the guarantees the **table itself** enforces
(`UNIQUE`, `ON CONFLICT`, a trigger, a `CHECK` constraint) — and none of the guarantees any
single *writer* enforces in its own code.

## Write the doc so it cannot be misread

Name the writer, always:

    bad   the INSERT dedups by commit
    good  the admin panel's manual writes dedup by commit. POST /api/changes does NOT —
          it is a bare INSERT (routes/changes.ts:84). Any new HTTP writer has no net and
          must check first.

And when a doc has already misled someone, say so in the doc itself — the correction is
cheap; the same misreading twice is not.

## The check before relying on it

Before assuming a documented guarantee protects a new caller, open the handler that new
caller actually goes through and look for the enforcement there. If it is not in your
path, it does not protect you, no matter how confidently the doc states it for some other
path. Same family as: patching client-side validation is not the invariant when a second
route writes to the same table without passing through it — both are "I fixed the door I
could see."
