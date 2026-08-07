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

## Bonus: where a period floor lives

If a cutoff ("only from July onward") exists solely in the render, a second consumer of the same API sees the full accumulated history without ever learning that a cutoff exists.

While it is a **presentation** decision, the render is the right place for it. The moment it becomes "this is what counts", it is a **business rule** and belongs on one side only. Worth deciding deliberately rather than leaving it wherever it first landed.
