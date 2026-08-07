# The API boundary — identity fields, unicode matching, and the real response shape

Three failures that all live at the same seam: the front sends or reads something that *looks* right and the backend disagrees. None of them is caught by a typecheck or a linter.

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
