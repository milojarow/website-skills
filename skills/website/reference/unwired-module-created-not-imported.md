# A module that is created but never imported looks exactly like finished work

Three cases of the same defect measured in one repo, and the most expensive one is worth
naming: a pricing rule module — the logic for what price to charge today, with the business's
timezone resolved correctly, a validity window, and a manual-extension path. **Complete and
correct.** And for a full week, **zero imports anywhere in the tree**. The site kept rendering
the list price every day, and on the day the promotion was scheduled to start, nothing would
have changed on screen.

The other two in the same repo: a catalogue constant nobody imports (the catalogue actually
served comes from a different file), and a pricing file that is really just a fallback, because
the backend wins whenever it returns data.

## Why code review misses all three

The file exists, is well written, compiles, and even has good comments. A `grep` for the
concept finds it immediately. **Only the importer distinguishes "done" from "written."** The
failure state (written, not wired) and the success state (written, wired, live) produce an
identical artifact on disk — the difference only shows up one hop away, in who calls it.

## The rule

**The commit that creates a module is the one that wires it.** Creating in one commit and
connecting in a later one opens a window where the repo looks complete and the screen hasn't
changed — and that window is measured in days, because nothing is watching for it: it doesn't
fail a build, a type check, or a test.

If the wiring doesn't fit in the same commit, the module isn't finished yet — say so instead of
marking it done.

## The detector — one line, run before closing any new module

```bash
grep -rl "from.*<module-name>" app/ components/ | grep -v node_modules
```

Zero results = not in the product, however well written. Always pair it with a **positive
control**: run the identical grep against a module you already know is imported, to confirm the
pattern itself works before trusting a zero from it.

## The family this belongs to

Same shape as a `0` that can't be told apart from a `null`, an endpoint that answers `200` with
the login page, or a test that passes because its assertion never ran: **the failure state and
the success state produce the SAME observation.** When that happens, the observation is not
evidence — the fix is to find a signal where the two states actually differ (here: the
importer), not to stare harder at the artifact that looks fine either way.

## Adjacent trap, same family of "the instrument was gagged"

Counting API rows from output already truncated by a pager: requesting `perPage=50`, piping
through `head -12`, and concluding "the collection has 12 rows" — on a collection that actually
had 43. The instrument didn't lie; it was gagged and believed. Same pattern as piping a syntax
checker through `head` and reading the exit code of `head` instead of the checker's.

**When counting, use the count the API itself reports (a `totalItems`-style field), never the
length of whatever got printed to the terminal.**
