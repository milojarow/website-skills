# Calendar days vs instants — the `date` column that shifts every row by one day

A column declared `date` in Postgres does **not** reach the front as `"2026-08-07"`. The driver serializes it as a full timestamp:

```json
"date": "2026-08-07T00:00:00.000Z"
```

Format that with `Intl.DateTimeFormat(..., { timeZone })` over `new Date(value)` for any zone west of Greenwich and you get **the day before**. Midnight UTC read in a negative-offset zone is the previous evening. Not one row — *every* row on the screen, all off by one.

**Why this is dangerous and not merely annoying:** there is no error, no console warning, no 400. The dates look plausible; they are simply all wrong. A typecheck and a linter both pass with the defect in place. It's the same family as [an endpoint whose real shape differs from its documented one](api-boundary-contracts.md) — the data is present, the shape isn't what was assumed, and the screen lies with a healthy face.

## The fix, and why this form and no other

```js
// Calendar day: slice, then anchor to NOON UTC.
const day = (v) => new Date(`${v.slice(0, 10)}T12:00:00Z`);
```

Slicing to 10 characters instead of branching on the string's length (`v.length <= 10 ? … : …`) makes the function **indifferent to both shapes**: if the backend ever starts sending the bare date, the same code returns the same answer. The branching version is the one that carries the bug — its "short" arm was correct and its long arm fell through to a raw `new Date(iso)`.

Noon UTC survives every zone from UTC−11 to UTC+11 without changing day.

## The real correction is not the slice

It's that **calendar days and instants are different types and need different functions**:

- A **calendar day** (an event's date, a signup date, a due date) is the same day everywhere on earth. It must never pass through a `Date` that assigns it a zone.
- An **instant** (`created_at`, `updated_at`, an audit timestamp) genuinely has a time of day, and there `new Date(iso)` formatted in the local zone is correct — slicing *that* to 10 characters would show the wrong day for everything captured after evening.

Having one function for both guarantees being wrong about one of them. Naming them distinctly (`formatDay` vs `formatMoment`) is what stops the next edit from re-merging them.

## Verifying it without standing up a test runner

Run **both** versions of the formatter over the same cases and print them side by side:

```
input                       expected   OLD          NEW
2026-08-07T00:00:00.000Z    Aug 7      Aug 6 WRONG  Aug 7 ok
2026-01-15T00:00:00.000Z    Jan 15     Jan 14 WRONG Jan 15 ok
2026-08-06 (bare)           Aug 6      Aug 6 ok     Aug 6 ok
```

Two things make that check worth running:

1. **Include a winter date.** Many zones change offset with daylight saving — and neighbouring zones don't always agree, some keeping DST after others drop it. A fix validated only against summer dates may be leaning on that season's offset and break months later.
2. **Confirm the OLD code FAILS.** If the check passes both versions, it measured nothing. Validate the instrument before trusting it on the object being measured.
