# The client clock decides WHEN TO ASK — never whether a deadline expired

The backend serves an already-computed window: `{ open: true, expires_at: "…Z" }`. The UI has to
disable a control when the window closes. **Nothing happens when it closes** — no event, no
record is written, time simply runs out — so without a local timer the control stays enabled
until the next refresh.

The obvious version, and the wrong one:

```js
const open = data.open && Date.now() < expiresAt;   // ❌
```

## Why it is wrong

`expires_at` was computed by the **server's** clock. Comparing it against `Date.now()` compares
two different clocks and hands the verdict to the one you control least: a tablet, a borrowed
phone, a laptop that just came back from suspend.

A device running an hour fast **locks the control on every record at once**, and the report that
reaches you is "I can't do anything, the system is down" — no error, no log, and perfect on the
machine of whoever wrote it.

The asymmetry that settles the design: **a stale flag costs one extra request; a control locked
by mistake costs a day of work.** And the server rejects the late write anyway, so the "benefit"
of deciding early was zero.

## The correct shape

The clock produces **an intention to ask**, not a verdict:

```js
open: data.open !== false,                                   // the server decides
due:  data.open !== false && Number.isFinite(ms) && ms <= 0  // the clock only asks
```

And when `due` fires, **one** refetch, guarded by `id:deadline`:

```js
const askedRef = useRef('');
useEffect(() => {
  if (!due) return;
  const key = `${id}:${deadline}`;
  if (askedRef.current === key) return;   // without this: infinite loop
  askedRef.current = key;
  refetch();
}, [due, id, deadline]);
```

**The guard is not optional.** The refetch returns the SAME `expires_at`, so the clock says `due`
again on the next tick and without a key it asks forever. With clock skew the panel asks early or
late instead of deciding wrong.

Three details that complete the pattern:

- **Never let the clock OPEN what the server closed.** It may only request a re-check of
  something the server still considers open. Reopening belongs to the server.
- **Ask the endpoint that does NOT write.** If the detail read marks as seen or bumps a counter,
  that write emits a realtime event and the consumer feeds itself. Ask the list, not the detail.
- **The timer must not exist when there is no deadline to watch** (`if (!watching) return`), and
  it should move nothing but a `now` value. If any scroll effect depends on the state the tick
  touches, the user loses their reading position on every tick.

## The assertion nobody writes

Testing "expired ⇒ closed" is not enough. Test the **skew**:

```
server open + device believes it expired  ->  { open: true,  due: true  }
clock 6 h fast                            ->  { open: true,  due: true  }
server closed                             ->  { open: false, due: false }
the clock never opens what the server closed -> { open: false, due: false }
```

## Sibling: a test bench running in UTC cannot see a time-zone bug

From the backend of the same system: hundreds of date assertions, **all green with the bug
live**, because the harness ran in UTC — at offset zero `"…35.793"` and `"…35.793Z"` parse
identically. The shift only appeared on the user's device.

Fix: force `process.env.TZ` to a zone with a real offset on the **first line** of the bench, with
the reason written above it so nobody "cleans it up" — and verify that `TZ` at runtime really
moves `Date` in that engine instead of assuming it. See
[calendar-days-vs-instants.md](calendar-days-vs-instants.md) for the parsing rule behind it.
