# A retry is not an error, and `already_had_it` is read from a FIELD, never from the HTTP status

Contract for a route that ASSIGNS something that may already exist (attaching an
already-uploaded image to another object, adding a member to a group, tagging): the route
answered **422 `already_has_it`** when the relationship already existed.

**That's wrong, and the reason generalizes to every assignment route:** whoever retries
already did their job. The relationship is exactly where they asked for it, and the state of
the world is exactly the one they requested. Answering with an error forces the client (and any
offline queue) to distinguish "it failed" from "it was already there" — and that distinction
gets built wrong the one day nobody is watching it: **either it retries forever, or it marks as
failed something that actually happened.**

Correct shape:

    201  first time         { ok:true, already_had_it:false, ... }
    200  already had it     { ok:true, already_had_it:true,  ... }
    422  invalid target     (retired / not applicable)
    404  does not exist

## 🔴 `already_had_it` travels as a FIELD in both cases

**Don't discriminate by the HTTP status.** 200 vs 201 is exactly what a proxy, a CDN, or a
gateway flattens without telling anyone. The explicit field survives; the status doesn't.

## The identity of an assignment is not the identity of the file

Related trap: the uploaded file's uuid identifies a **FILE**, and the same file can hang off
several objects (three products sharing one photo). So the uuid **can never identify an
assignment** — the route must take the `id` of the relationship's own ROW, not the file's. If a
public listing exposes both together, the mapping doesn't cost a credential.

## `assign` ADDS, it does not MOVE

If the response carries something like `previous_location`, the UI **cannot say "moved"** —
that's a lie. Actually moving is a different route, and that one requires a **reason**, because
undoing a claim about the world is itself a claim about the world.

## And whichever side writes first

When the same button has two paths (upload a new file vs pick an existing one), the second one
tends to end up unwired, because **both paths end in the same local function** and the local
records look identical. The difference only exists on the server side.

    of 38 recorded actions: 4 reached the server on their own, 34 had to be migrated by hand

Rule: **it goes to the server FIRST, and nothing is written locally if the server didn't keep
it.** Writing local state when the server call failed is exactly how a record nobody can find
later gets born.

⚠️ And: **a migration over a list frozen while someone keeps working always leaves a gap.** An
action logged between the list's snapshot and the day the wiring landed falls through the hole
and only turns up by cross-checking record by record. Cross-check: the scaffold's own log
against the real table; whatever is left over in the first one is human work that never
arrived.
