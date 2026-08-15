# The API boundary — identity fields, unicode matching, the real response shape, and fields that change meaning

Four failures that all live at the same seam: the front sends or reads something that *looks* right and the backend disagrees. None of them is caught by a typecheck or a linter.

## 1. Send identity fields explicitly, even when you think the token carries them

A backend that validates an identity field (`author`, `owner`, `createdBy`) against a table — `WHERE name = $1` — will reject every request that omits it. The client that composes a POST out of the "business" fields only (`project_id`, `title`, `detail`) and assumes the server derives *who wrote it* from the session token fails **100% of the time**, from the user's first click.

The decision is not symmetric, which is what makes the rule easy:

- **The backend derives it and you send it too** → costs nothing, and it matches by construction (both come from the same authenticated user).
- **The backend requires it and you don't send it** → every request 400s.

Omitting it works in one of the two worlds; sending it works in both. **Send it.**

## 2. NFC vs NFD — the same name, different bytes, `=` says no

Any exact comparison against a human name is a trap when the value carries an accented character. `í` has two valid encodings that render **identically on screen**:

```
"café"  NFC → 4 chars → 636166c3a9      (é = c3 a9, one character)
"café"  NFD → 5 chars → 63616665cc81    (e + combining acute, two characters)
NFC == NFD ?  False
```

Databases typically store what was originally inserted (commonly NFC). If the value picks up a `.normalize()` anywhere in the chain, gets copy-pasted from another app, or is typed by hand, `WHERE name = $1` misses and the error you get is *"unknown author"* while you stare at two visually identical strings.

**The armor is to never type it.** Take the value from the session JSON (`/auth/me` or equivalent) and pass it **verbatim** — no `trim`, no `normalize`, no capitalization. It travels NFC end to end and matches by construction.

**And don't hardcode it**, for two reasons: it's brittle, and it puts real people's names into the frontend repo.

For bulk loads (a migration, a seed), **let the side that already holds the value correct supply it** — read it from its own table inside the `INSERT` instead of writing it into the JSON that travels. Every hand-written copy is one more chance to introduce NFD.

**Diagnosing it once it's happening:**

```sql
SELECT name, encode(name::bytea,'hex'), length(name) FROM users;
```

and compare against the hex of what arrives. **Comparing bytes to bytes takes a minute; comparing two strings that look the same takes an afternoon.**

## 3. Verify the response's real shape, not the document that describes it

A written contract saying `{id, name, email}` is not evidence. Measured with a real token, the same endpoint can return `{email, name}` — no `id`.

Declaring `id` required in the TypeScript type makes the compiler **endorse** a `user.id` that is `undefined` at runtime: a type error converted into a plausible, false value, which is worse than a crash. Nothing turns red; something downstream silently keys on `undefined`.

**Call the endpoint with a real token and type what it actually returns.** If the contract and the measurement disagree, the measurement wins and the contract is the bug to file.

This is the same family as [a `date` column that arrives as an instant](calendar-days-vs-instants.md): the data is there, the shape isn't what was assumed, and the screen lies with a healthy face.

## 4. A field that changes MEANING — not name — inverts every verdict built on it, in silence

Measured in production: a panel spent weeks painting **"delivered"** over what was really
"accepted", and the day the backend was corrected the *same untouched UI* started painting
**"never arrived"** over every healthy message.

**Why this is worse than a rename.** A rename degrades to an absent field: a tolerant
normalizer returns `undefined`, the default kicks in, and at worst the UI says nothing — it is
either visible or harmless. A **semantic change keeps the name, the type and the range of
values**. Nothing fails. The same `false` that used to mean "it failed" now means "not
confirmed yet", and the UI keeps drawing the old verdict with total confidence. The code has no
way to notice, and neither does the type: `boolean` is still `boolean`.

```
before:  delivered:true = "the provider accepted it"        -> UI ✅
after:   delivered:true = "the provider CONFIRMED delivery" -> UI ✅   (same line, different truth)
```

The case that makes it visible: under the new semantics a healthy message is **born `false`**
and rises to `true` a second later, when the receipt arrives. If the UI refetches the record
right after sending — the most ordinary pattern there is, "let the stored record have the last
word" — **every successful send flashes the failure state** for that second. The false alarm
lands on the happy path, on every operation, not in some rare corner.

### The rule

**A boolean cannot carry a verdict that has more than two states.** As soon as the real process
has phases (`accepted → sent → delivered → read | failed`), the verdict is read from a **status**
field and the boolean becomes a past datum, not a source.

And the corollary almost everyone forgets: **the empty state `""` is not a state — it is the
absence of an instrument.** Rows written before tracking existed were never measured. Drawing ✅
on them is inventing; drawing ⚠️ repaints months of healthy history as failures. **Draw nothing.**
Silence is the honest mark for "I don't know".

```js
const failed = m.status === 'failed';   // only an explicit failure is a failure
if (!m.status) return null;             // no instrument, no verdict
```

### Two implementation traps that come with it

1. **Re-key the cache/diff signature too.** If the list keeps a signature to decide "this is
   already on screen, don't replace its state" and that signature is keyed on the OLD field, the
   mark freezes at whatever it was on the first fetch and the transitions never repaint. The
   signature must be keyed on the field that now decides.

2. **One HTTP status code can gain a second meaning in the same deploy.** A `409` that meant
   exactly one thing ("another actor took the resource") can start also meaning "the time window
   closed". The old UI not only showed the wrong reason — it **mutated state** (marking the other
   actor as active) based on what the 409 supposedly proved. **A status code is not a
   discriminator:** require a `code` field in the body and route on it — never on which fields
   are present or missing (absence becomes false the day someone adds the field in passing), and
   never on the message text.

### How to catch it before the user does

For any field that feeds an on-screen verdict, ask:

- **What is its value at instant 0 of a SUCCESSFUL operation?** If it is the same value that
  draws the error, flicker is guaranteed.
- **What do the rows written before the change carry?** Ask for the real count per value instead
  of assuming it. Measured once: every existing row carried the flag explicitly `true`, so the
  "absent ⇒ ok" default never fired — relying on it would have been imaginary coverage.
- **The negative assertion is worth more than the positive ones:** grep the **served bytes** for
  the old comparison and require **zero** occurrences. Proving the new logic exists does not
  prove the old one died.
