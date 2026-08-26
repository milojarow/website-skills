# A counter over the RETIRED vocabulary reads as "nothing to do" while the real work hides on the other side

When a product migrates from an invented vocabulary — a catalogue/list/enum the team built
before having the client's real data — to the real one, the old code does not break. It keeps
working perfectly over the dead set. Nothing looks "wrong": a grep for errors finds nothing,
because every statement was true the day it was written and only expired when the world under
it changed. This sweeps differently from a bug.

Measured (an internal capture app, 156 catalogue items total):

    counter shown on screen        194 "fields still to ask"   ← counted over 47 RETIRED items
    the same computation over        0                          ← counted over the 112 LIVE items
    true figure                    889
    items actually listed          47 of 156 possible

The operator read "194 to ask" and it was all dead-catalogue debt. **A zero produced by looking
at the wrong universe is indistinguishable from "nothing to do."**

## How to detect it

The offending functions all share one shape: resolve the object by looking it up in a single
universe's collection, and return empty if it is not there.

```js
const item = oldCollection().find((c) => c.id === id);
if (!item) return [];          // ← every item from the new universe lands here, silently
```

Sweep: grep the old collection by name and classify every use as **read / count / write**. All
three need to know about both universes, and each breaks differently:

- **read** → the screen never lists the object
- **count** → the number lies in the opposite direction (reports a lot of dead work, zero of
  the live work)
- **write** → rejects with "that object doesn't exist" **while the object is on screen**

## The fix is a bridge, not an OR

Don't patch it with a `||` that queries both collections. Find the attribute **both universes
actually share** (here, the category) and resolve through it. An object that doesn't map stays
out **on purpose and by name** (here: a handful of items that are not furniture for that kind
of room), never by accident.

## The arbiter can carry the same bug

The gate script counted over the dead universe too: it reported "1 object missing attributes"
and **passed green** while 112 of 112 live objects had none. The instrument measured a dead
universe and its silence read as health.

**Mandatory positive control before trusting the new count:** seed ONE record in the live
universe and confirm the number drops by exactly one. If it doesn't move, the check is a
tautology.
