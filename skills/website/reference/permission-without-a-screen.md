# A permission that is handed out and no screen exercises is a dead switch

Applies to any system with per-person permissions, capabilities, or feature flags — typically
split across a front end and its API.

## The failure mode

A permission grant lives in TWO places: the **view** (who sees the screen) and the **permission**
(who may execute). Moving one without the other opens a hole that **does not crash**:

    sees the screen      A, B    ← cannot execute
    holds the permission C, D    ← has no screen

Measured result: **nobody could perform the action**, and no typecheck, lint or test saw it.
Both halves are individually correct. Something simply stops being possible, silently.

The twin variant: a capability that was **never exercisable** — it is granted, the user record
translates it into "can do X", and no route or component ever reads it. The capability ran ahead
of the feature. And the inverse: the view is granted by role while the permission lives apart, so
revoking the permission **takes nothing away** — a switch that does nothing, and whoever hands it
out cannot tell.

## The gate that actually works

**Do not assert "role X has capability Y."** That re-states the table you already wrote: it goes
green while nobody can use it, because it measures the same fact twice. Assert these properties
instead:

1. **Every act is reachable** — at least one person both sees the screen AND holds the permission.
2. **Nobody opens a dead screen** — whoever sees the screen can execute something it offers.
3. **Nobody keeps an unusable permission** — whoever holds it has somewhere to use it.
4. **Every GRANTABLE capability appears in the code that consults it**, with an explicit exception
   list, each entry named with its reason (visible debt, not invisible). The check accuses in
   **both directions**: when an exception stops being one and nobody removes it from the list, it
   fails too.

**Grantable, not granted.** The set that matters is what *can* be handed out — what renders with a
switch on the permissions screen — not what is handed out today. Iterating only the granted ones
leaves a capability nobody carries yet invisible right up to the day someone enables it. Design
consequence: the catalogue must be a **list queryable at runtime**, not a union type / enum that
only exists for the compiler. Declare the list and derive the type from it
(`typeof CAPABILITIES[number]`), which also removes the need to regex the type.

## Three traps when writing that check

- **Reading the raw file makes it incapable of failing.** The permission's name survives in the
  comment explaining why the screen is gated, so the mutation "take the capability away from the
  screen" still passes. Strip comments — block, line, and JSX `{/* */}`.
- **But stripping comments and nothing else makes it far too permissive.** Mentioning, displaying
  and requiring are three different things and **only the third is the gate**: the capability also
  shows up in a translation map (`'inventory.manage': 'Move inventory'`) and that counts as a use.
  Discard presentation maps **by shape, not by filename** — appearance in **object-key position**
  (`/'<cap>'\s*:/`) is what all of them look like, wherever they live. Excluding three files by
  name is brittle: a new map in a fourth file walks straight through.
- **Do not anchor to ONE way of requiring.** A grep for `requireCapability(` misses capabilities
  required dynamically inside the handler, and those that travel through an intermediate constant.
  Filtering out what does NOT count (comments, map keys) is more robust than enumerating what does.

## Debt that spans two layers is declared WITH THE SIDE NAMED

The check measures reachability **within its own tree**, so structurally it cannot tell these two
apart, and they look identical in a debt list:

    screen missing, route already exists     ← a finish
    neither route nor screen                 ← a build

The difference is not academic: it changes the price of the decision for whoever receives it.
Declaring the first as if it were the second bills work that does not exist.

**How to measure it with no credentials and without writing anything** (probe a route's
EXISTENCE, not its behaviour): a POST with no token and an invalid body, plus a negative control
against a made-up route.

    POST /v1/<area>/transfers        → 401   exists, reached authentication
    POST /v1/<area>/made-up-control  → 404   this is what a missing one looks like

Without the negative control the 401 proves nothing: some servers authenticate before routing and
answer 401 to everything.

**What not to do** is let the gate call the other layer's API to cross the boundary on its own: a
gate that depends on the network stops being a gate the day there is no signal. Write the rule
into the debt declaration and measure that side once, by hand.

## Why the discovery method is worth copying

A gate written for one layer, handed to a second layer and run there against a completely
different implementation of the same property, found a real case the first one did not have — and
correcting the instrument with that lesson **broke it from the other side**. Two of three versions
answered wrong while all three ran fine. **A well-written gate is portable across layers**: it is
worth more to hand it over than to describe it.

The same failure mode appeared three times in one session wearing a different face each time:
**the scope of the search decided the answer, and the answer looked complete.** Three files
instead of the whole tree; the literal phrase instead of the idea; one tree instead of the two the
system has. Before signing a negative, the question is not "what did I find?" but **"where did I
not look?"**.
