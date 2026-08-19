# Counting strings in the HTML of an SSR page: the RSC payload doubles everything

Verifying a live page with `curl` + `grep` is cheap and correct. But the HTML an App
Router page serves has two properties that break the naive count, and both produce a
number **higher or lower** than the truth with no error anywhere.

## Trap 1 — the RSC payload repeats every visible string

Next embeds the flight payload in the same document, inside
`<script>self.__next_f.push(...)</script>` blocks. Every text you rendered appears
**twice**: once in the markup, once inside the payload.

Measured on a page with 4 cards: `grep -o 'sample rate' | wc -l` → **8**. Anchored to
the rendered markup (`grep -o '>sample rate<'`) → **4**, the real number.

The practical damage: "I expected 4 and there are 8" reads as *I am rendering too
many* and sends you debugging a bug that does not exist.

## Trap 2 — `grep -c` counts LINES, not occurrences

Production HTML arrives on essentially one line. `grep -c` returns **1** even when the
string appears 40 times, or **2** if there happen to be two lines. To count
occurrences it is always `grep -o … | wc -l`.

## Trap 3 — comparing two counts with different anchors

The small version of "use an independent reference or you are comparing the bug with
itself". Checking that 3 cards say `Available` and 1 says `Not available`:

```bash
A=$(grep -o ">Available<"   x.html | wc -l)   # anchored     -> 3
B=$(grep -o "Not available" x.html | wc -l)   # not anchored -> 2  (doubled by the RSC)
```

`A` counts rendered markup, `B` counts markup **plus** payload. The verdict came out
`FAIL` with the code perfectly healthy.

### Anchoring is not "use the same form on both sides" — it is what SEPARATES the copies

Verified independently on a second machine against the same document (111 KB, 142
lines, 5 `self.__next_f` blocks):

| string | `grep -c` | `grep -o` | markup | RSC payload |
|---|---|---|---|---|
| `Available` | 2 | 6 | 3 | 3 |
| `>Available<` | 1 | 3 | 3 | **0** |
| `Not available` | – | 2 | 1 | 1 |
| `sample rate` | – | 8 | 4 | 4 |

The anchored form scores **zero** inside the payload, because the RSC encodes the same
text with another structure and never carries the `>` `<` glued to it. So **anchoring
to the markup is the mechanism that discards the payload copy**, not a matter of
symmetry.

Consequence: equalizing anchors is not enough. Use the bare string on **both** sides
and both counts double, the comparison looks perfectly consistent, and it is still
wrong. The rule is **anchor to the rendered markup**, and only then keep both sides
identical.

Extra: `Available` is a substring of `Not available`, so without the `>…<` anchor the
positive count is contaminated by the negative one.

## The recipe

```bash
curl -s https://<site>/ > /tmp/live.html
grep -o ">Available<"     /tmp/live.html | wc -l
grep -o ">Not available<" /tmp/live.html | wc -l
```

And **before believing any zero**, run a positive control with the same method: count
something you know must be there (a button label, a field caption). If the control
returns 0, the instrument cannot find anything and no zero from that run is worth
anything.

## Corollary: proving a secret did not reach the client

The same positive control applies to sweeping the bundle. `grep -rl "$TOKEN"
.next/static` → 0 proves nothing on its own. Run the same grep for a string that
**must** be in that bundle (a button label, a UI error code); if that one appears and
the token does not, the zero is evidence.
