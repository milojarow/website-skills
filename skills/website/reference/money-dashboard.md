# Screens that show money — six rules, and one that isn't about code

A dashboard of income and spend is not a normal data screen. Its numbers get carried into decisions — a valuation, a budget, a "can we afford this" — and a figure nobody can reproduce is worse than no figure at all. These are the failure shapes that keep recurring.

## 1. Never add two currencies, and never convert without a dated rate

If the data holds more than one currency, each gets its **own row**. Converting requires an exchange rate **with a date attached**; if the rate table doesn't cover the period, any conversion is an invented number. **A figure nobody can reproduce is useless for the decision the screen exists to support.**

**The trap that looks like a bug and isn't:** the same charge appearing in USD on the vendor's receipt and in MXN on the account statement is **normal** — the vendor bills in one currency and the card settles in another. A detector that flags "currency divergence" as suspicious fires on nearly everything. The discriminator that actually works is the **ratio** between the two numbers: a ratio near ~1.0 against a ledger denominated in MXN is the only signature with no innocent explanation.

## 2. Sum in integer cents, never in floats

`0.1 + 0.2 !== 0.3`. Across dozens of charges the error accumulates until the header total disagrees with what you get adding up the table underneath it by hand. On a money screen that is not a rounding artifact — it is a user who can't make the numbers work and therefore **stops trusting the whole panel**.

```js
const toCents = (m) => Math.round(parseFloat(m) * 100) || 0
```

And have the backend send amounts as **strings** for the same reason: a Postgres `numeric` serialized into a JSON `number` has already lost precision before it reaches the front. Same family as [a `date` column that arrives as an instant](calendar-days-vs-instants.md) — the driver's default serialization is not the shape you assumed.

## 3. Committed ≠ collected. Two figures, never one

What active customers *should* pay per month and what actually came in are different quantities. Fusing them into a single number is the most common way a dashboard ends up looking healthier than the business is. **Cancelled** subscriptions stay out of the monthly figure — counting them inflates it with customers who no longer pay.

**Corollary about filters, and it cuts both ways:**

- A payment is a **dated fact** → it *is* sliced by the reporting period.
- An active subscription is a **state as of today** → it is **not** sliced by its signup date. Doing so deletes the oldest customers, who are exactly the ones holding the number up.

## 4. Show the gaps, and paint the two kinds differently

Two classes, and confusing them is expensive:

- **Outside the total** (unclassified records) → these read as **absence**. Something is missing and it shows.
- **Inside the total but misattributed** (the same concept filed under different categories depending on the month) → these **look fine**. They are the worse kind precisely because they don't look like a gap. Their warning belongs **on the total itself**, not beside it.

A clean total that hides its gaps lies better than one that declares them.

## 5. Two axes that look like one: grouper vs entity

A single `company` field carrying a mix of **organizations** (`acme`, `acme-labs`) and **customers** (`customer-a`, `customer-b`) makes the central question — "how much does Acme spend?" — unanswerable, because customers render as sibling organizations. That is two columns, not one.

**And the asymmetry of `null` is information, not a defect to smooth over.** In one source the field may exist *by construction* (the record lives inside that organization's instance, so its presence is a **measurement about the source**), while in another it is genuinely missing. Forcing a `null` where there isn't one, or backfilling one where there is, erases that distinction.

Unassigned rows count toward the **grand total** and **never inside a category line** — a category line that swallowed them would be reporting spend with an unknown owner as if it belonged to a known one.

## 6. The on-screen copy *is* the interface — a caption that lies is a bug with no test

The recurring case: the grouping code is correct — unassigned rows never enter any organization's line — and the note on screen reads *"these movements are included in the total"*, flatly. Ambiguous, and under the natural reading, false.

**No typecheck, linter, or unit test catches that.** On a screen feeding a money decision the user acts on the text, not on the code.

When reviewing a data panel, read every caption **as an assertion** and ask of each: *is this still true of the code in front of me?*

The same failure has a layer above the code: a caption can also assert something about the
**client's real-world business** (who their vendors are, what they outsource) rather than about
the data on screen. That claim needs the client's confirmation, not a plausible inference — see
[client-business-claims-in-copy.md](client-business-claims-in-copy.md).

## 7. A shared-visibility dashboard: isolation applies to the detector's OUTPUT, not just its input

A backend that flags "inconsistent" rows — the same recurring charge attributed to
different owners in different periods — has to *read* the ownership label to detect the
inconsistency. That part is normal and necessary. The trap is shipping the finding with
the evidence still attached:

    {"item": "<vendor>", "owners": ["business", "personal"], "periods": ["2026-03", "2026-07", "2026-08"]}

Every field is accurate, and together they publish — on the screen the *other* stakeholder
opens — exactly which periods one side paid for something privately. Nobody wrote a line of
code intending that; it fell out of shipping the whole classification instead of the verdict.

**Isolation applies to the OUTPUT, not just the input.** It is fine and necessary for a
classifier to see private data internally — that's how it classifies. What crosses the
boundary must be the *signal*, not the *evidence*:

    ship:  "this total is missing 2 periods"      (an integer, no owner names)
    don't: "it was privately paid in March and July"

Verify by grepping the raw payload for the forbidden field/value, not by reading the code
that builds it — the string should not appear at all.

**Narrowing a field changes its shape, and downstream copy silently stops making sense.**
If a consumer's sentence assumed the list could have two-or-more entries ("counted as A or
as B"), and the redesigned payload now emits at most one, that sentence renders as a
neutral statement of fact instead of a warning — the alert stops alerting while still
displaying, which is worse than disappearing. Whenever a list-shaped field gets narrowed or
values get stripped out of it, tell every downstream consumer that its length can now be
smaller (including 0 or 1), and re-read every string built from it.

## 8. When the exception covers the heaviest case, suspect the RULE, not the case

A shared-visibility dashboard flagged "vendors that don't count the same way every
period" — the same vendor attributed to different owners across periods. Shipping it took
four rounds of fixes, all individually correct (cut the leak in §7, scope it to the
panel's window, replace a count with a boolean so the exact period couldn't be derived by
subtraction, add a drift detector). Four correct fixes, all in service of a false premise.

The refutation was visible from round one: one vendor — a shared SaaS subscription anyone
on either side might expense — was filed as **an exception**, a single special case to
carve out. It was actually the single most frequent vendor in the data. A vendor used by
both sides in different periods isn't an anomaly; it's the ordinary shape of any shared
vendor, which is most of them.

**Count the weight of the exception before shipping it.** An exception covering a rare
corner is a corner. An exception covering the *most frequent* case in the data is the data
telling you the rule itself is wrong — ask, before another round of polish, "should this
exist at all?" The trap that hides it: **every individual fix was correct**, so each round
produced real evidence of progress. A mechanism getting steadily better is not evidence
that it should exist.

**The deeper diagnosis, worth checking for on any cross-boundary feature:** if answering
the question requires revealing private data to be useful, and requires withholding it to
be safe, no amount of redaction makes it both — the isolation leak was a *symptom* of
asking a private-data question from a shared surface, not the root cause. If a field must
cross the boundary to answer the question, the question belongs on the other side of the
boundary, not in a redacted form on this one.

**Retire it by deleting the computation, not just its rendering.** Dead code nobody reads
is exactly what someone "restores" months later believing it regressed. Leave a tombstone
comment stating the false premise and how it was refuted — otherwise the absence itself
reads as a regression and gets undone.

## Bonus: where a period floor lives

If a cutoff ("only from July onward") exists solely in the render, a second consumer of the same API sees the full accumulated history without ever learning that a cutoff exists.

While it is a **presentation** decision, the render is the right place for it. The moment it becomes "this is what counts", it is a **business rule** and belongs on one side only. Worth deciding deliberately rather than leaving it wherever it first landed.
