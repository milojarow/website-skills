# Mirroring a server limit in the client: counting the same units, deploying in the right order, proving the build

To warn *before* sending, the client needs the server's maximum — so the number now exists on
both sides, and the counter next to the textarea is a second implementation of a server rule.
Three things go wrong, none of them loudly.

## 1. Count what the server counts: CODE POINTS

If the server measures with `[...body].length` (code points) and the client shows
`draft.length` (UTF-16 units) or `Buffer.byteLength` (UTF-8 bytes), the two disagree silently
and only for certain text.

**Mandatory control on the instrument** — the same 5 000 emoji give three different numbers:

| method | result |
|---|---|
| `[...s].length` (code points) | 5 000 |
| `s.length` (UTF-16 units) | 10 000 |
| `Buffer.byteLength(s)` (UTF-8 bytes) | 20 000 |

A test whose fixture is ASCII cannot tell the three apart, so it passes while the counter is
wrong. Include at least one character outside the BMP.

And replicate the **whole pipeline, in order**, not just the count. A realistic chain is
`draft.trim()` → `normalize("NFC")` → trim the tail → count. The NFC step matters: 20 000
decomposed `é` (`e` + U+0301) are 40 000 raw code points and 20 000 after normalizing. Verify
the counter against the server's literal expression, with cases: real prose, ASCII, emoji, NFD,
trailing whitespace, empty.

## 2. The server deploys first

If you publish the client first, the counter promises a ceiling the server does not accept yet
— it lies to the user, which is worse than having no counter at all. Write the ordering into
the code (a comment on the mirrored constant), don't remember it.

## 3. A test that types the number by hand is not a gate

A server test asserting on `'x'.repeat(20_001)` as a literal stays green with the constant set
to *any* value — it is testing its own copy. Import the constant, and add a test that **pins
the number**:

```js
expect(MAX_BODY_CHARS).toBe(50_000);
```

Moving the limit then breaks the suite and forces the change to be a decision instead of an
accident.

## Proving that what is DEPLOYED is the new build, when the change is an integer

A negative control alone proves nothing: rejecting 50 001 looks identical under a ceiling of
20 000 and one of 50 000 — same output, zero discriminating power. What discriminates is a
**positive marker that exists only in the new build**:

- **Server side:** a log line the previous version never wrote. Probe the live process and see
  it in the journal — that identifies the binary. Comparing the deployed file's `sha256`
  against the repo, and the container's `StartedAt` against the file `mtime`, closes the chain.
- **Published site:** i18n gives the marker for free. The new string travels embedded in the
  page payload, so `curl <url> | grep "<new string>"` proves the build — and using the *other*
  locale's string as a control (it must return 0 in this locale) rules out the false positive.

Counting that string is its own trap; see
[counting-strings-in-rendered-html.md](counting-strings-in-rendered-html.md).
