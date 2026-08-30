# A project CLAUDE.md past the size limit gets SPLIT, not summarized

Measured case: a project `CLAUDE.md` grew to ~190 KB / ~2,200 lines — tens of thousands of
tokens loaded on every session start. Summarizing it by hand destroys the asset: the bulk of it
wasn't filler, it was the record of *how* each decision was reached — the measurement, the
falsified alternative, the defect that caused it. A summary keeps the conclusions and throws
away the proof, and weeks later someone reverts a decision because the reasoning behind it is
gone.

## The correct move is a cut into two layers, and the first step is `cp`, not rewrite

1. `sha256sum CLAUDE.md`, then `cp CLAUDE.md .claude/decision-log.md`, then `sha256sum` again and
   compare. The old file **is** the new file, byte for byte — "nothing was lost" becomes a
   verifiable fact instead of a promise. (There is usually no git history to fall back on: a
   project-level `CLAUDE.md` is commonly gitignored.)
2. Rewrite `CLAUDE.md` down to **one line per decision — the rule, not the story.** The rule is
   what needs to be loaded to avoid breaking something; the story is what needs to stay
   available to consult.
3. Put a navigation instruction at the top of `CLAUDE.md`: grep the log by the decision's
   UPPERCASE title before touching that area (e.g. `grep -n "PAYROLL" .claude/decision-log.md`).

**The maintenance rule is what prevents recurrence**, and it has to live inside the file:
*"when a new decision is made, the full story is APPENDED to the log; only the one-line rule
goes into this file."* Without that sentence written down, the file grows back to its original
size in a couple of months — which is exactly how it got there the first time.

## Non-obvious details

- **Derive the index, don't type it.** `grep -nE "^#{1,3} "` finds section headers; for a
  section with no headers inside it (a single 1,000+ line block can have none), the anchor is
  the UPPERCASE bullet title, found with `awk 'NR>=A && NR<=B && /^- /'`.
- **Never index by line number** — the log gets edited and numbers shift. The anchor is a unique
  string (the bullet's title), never a position.
- **Measure where the bytes are before cutting**, instead of eyeballing it:
  `awk '/^## /{if(h)printf "%7d  %s\n", c, h; h=$0; c=0} {c+=length($0)+1} END{...}' file` — in
  the measured case, two sections held 60% of the total weight.
- **Leave HEADROOM, don't graze the limit.** A first pass that lands just under a suspected
  limit (e.g. 38.8 KB against a suspected 40 KB ceiling) isn't fixed, it's one edit away from
  the same complaint again. Split out a third piece if one part of the content has **its own,
  different moment of use** (read before a meeting, not before touching code) — the trigger for
  a third file is a different audience/timing, not raw size.
- **Before moving anything, grep the codebase for pointers into the file.** A stray code comment
  ("see CLAUDE.md § the billing date") means the phrase has to survive verbatim in whichever new
  file holds that section, or the pointer goes dead and the decision it protects gets reverted.
- **Confirm both pieces land inside the existing gitignore** (`git check-ignore -v <file>`) and
  that `git status --porcelain` stays empty afterward. A project's internal decision log pushed
  into a client repo is a leak.

## How NOT to verify the size limit

Grepping the binary of the coding agent for a size-limit constant surfaces messages from the
**memory store** ("only part of it was loaded"), not from project-level context files — the two
are separate subsystems with separate limits. Don't conflate them, and don't cite a number that
wasn't actually measured: file size and per-session token count are the two numbers that were
measured, and they're what justifies the split — nothing more precise is needed.
