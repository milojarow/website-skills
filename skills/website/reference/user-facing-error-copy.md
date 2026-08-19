# User-facing copy is read by whoever is in front of the screen, not by whoever wrote it

A failure string written for the team ships to production because nothing rejects it: it typechecks, it lints, it renders. The reader — a customer, a client with an admin account, a stranger looking over a shoulder — is the only thing that can tell it's wrong, and by then it's live.

This is a **sweep**, cheap to run over any repo, with three mechanical triggers.

## The three triggers

A string that a person can see is team-facing (and therefore a leak) if it does any of these:

**1. It names a real person.** Measured: the status bar of a customer-facing form said *"Let <developer's name> know"* when a save failed. Two defects in four words — it assigns the reader work that belongs to the team, and it puts a person's name on a screen that a third party reads.

**2. It uses database vocabulary.** *"Leave it at `visible=false` until…"* — `visible=false` is a column name, not something a person says. Field names, enum values, table names and status codes all read as debris to the person in front.

**3. It assigns work to the reader, or describes a role the reader IS.** Measured on the other side of the same stack: a validation guard answered *"keep it hidden until THE CLIENT confirms it"* — and the person reading that error **was** the client, who had an admin account.

## How to sweep

A grep for error strings is not enough. The work is **separating strings a person can see from strings that only reach a log** — the same sentence is fine in one and a defect in the other.

Watch the "staff only, so it doesn't matter" reasoning: **if the business owner has a staff account, staff-only is still client-facing.** The audience of a screen is the set of accounts that can open it, not the role name in the code.

An unavoidable failure string owes the reader two things:

- **Say what happened without blaming them.**
- **Don't leave them without an exit.** If the exit depends on the team, the *system* takes it, not the user — an automatic retry instead of "tell someone".

## The corollary that makes it true

**If the message promises something, implement the promise.** "We'll keep trying on our own" becomes a fact only when a retry every N seconds exists behind it. Without it, it is the same lie as a false "saved" — just told in the future tense.

## When the server sends the reason and the client reads only the `code`, N failures look like one

Measured on a WebSocket chat: the server rejected with
`{"t":"error","code":"VALIDATION","message":"body too long"}`, and the client did
`switch (frame.code)` — painting one generic toast for the whole `VALIDATION` family. Three
distinct failures of the same endpoint (invalid recipient, empty body, body too long) reached
the screen as the same sentence.

The consequence is not theoretical: the person **could not find out why their message wasn't
sending**, and had to read the server's code to discover something the server had already said
in the frame. The reason travelled down the wire and was thrown away in the `switch`.

**Rule:** when the error contract carries `code` + `message`, the **code picks the family and
the message picks the phrase**. The `message` is contract text, not user text — map it to i18n
strings, never render it raw. Keep the generic fallback only for protocol reasons (malformed
frame, unknown type), where the person can do nothing with the information.

Cheap audit of any client: grep the error handler for `.code` and count how many distinct
server reasons land in each branch. A branch covering more than two *actionable* reasons is the
readability bug.

Related: the ceiling itself is usually mirrored on the client so it can warn before sending —
that mirror has its own traps, in [client-mirrored-server-limit.md](client-mirrored-server-limit.md).

## Initial state is the same lie in the past tense

"All saved" on first paint, when nothing has been saved yet, is that same false claim pointed backwards. Don't invent state on the client: if the backend returns a write counter, branch on it — `0` renders *"Ready to answer"*, `> 0` renders *"All saved"*.

Related: an explanatory caption on a screen is [read as an assertion](money-dashboard.md) about the code behind it, and no test catches a caption that lies.
