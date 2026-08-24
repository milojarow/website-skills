# Verifying that YOUR commit deployed: the witness must be unique to the diff, and pure ASCII

Two false verdicts, measured the same day on the same Next.js/Vercel project, one in each
direction. Both read exactly like the truth, which is what makes them expensive.

## 1. Polling a URL until it answers only proves something is alive

The usual "wait for the deploy" pattern — poll the URL until it responds — only proves a server
is up. The previous version answers too. If the build hasn't finished yet, the loop exits
**green** against the old bundle.

That loop answers "is anything there?", not "is my change there?". Treat it as a liveness check,
never as a deploy verification.

## 2. The witness has to be unique to the DIFF, not just present

A real false positive: a deploy was reported verified by grepping for a CSS utility class that
**six other components already used**. The grep matched the old bundle just fine — the class
had never left.

Pick the witness from the diff itself: a string that exists **only** after this change — a new
identifier, new copy, a new field name. Never a utility class, a framework name, or anything the
project already had before the change.

## 3. 🔴 Never pick an accented witness — guaranteed false negative

Minified bundles escape non-ASCII characters. Searching for the literal accented string finds
nothing, even though the text is fully deployed:

```
searching for:  "Nada se canceló"      → 0 matches
what's there:   "Nada se cancel\xf3"
```

Zero matches against text that **is** live, and it reads identically to "not deployed." This is
exactly the kind of false negative that leads to `rm -rf .next` and server restarts that fix
nothing, because nothing was broken.

**Rule: the witness goes in pure ASCII.** If the change is text-only and carries accents, search
for the unaccented fragment (`"Nada se cancel"`) or pick a different anchor from the same diff —
a class name, a data attribute, an id — instead of the accented prose.

## 4. Non-ASCII isn't the only survival trap — object property names survive; syntax, comments, and local names don't

Measured against the same served chunk, comparing candidate witnesses one by one:

| witness | survives minification | why |
|---|---|---|
| an object property name (`moneda`, `precio_vigente`) | ✅ | renaming it would break every access to that key |
| a literal string that renders on screen | ✅ | it's data, not an identifier |
| a local variable name | ❌ | gets renamed to a single letter |
| operators / syntax (`?.`, `??`, template literals) | ❌ | compiled to an equivalent form |
| **a comment added on purpose, to serve as a marker** | ❌ | comments are stripped — the most treacherous false negative, because it was placed exactly to be found |
| a non-exported function name | ❌ | gets renamed |

Optional chaining is the sharpest version of the syntax trap: `datos?.moneda` compiles to a
conditional (`null == a ? void 0 : a.moneda` or similar) — the `?.` stops existing as text, even
though the property it guards is still there. Grepping the literal source expression against the
bundle is a guaranteed miss:

```
searching for:  datos?.moneda     → 0 matches   ❌ rewritten by the compiler
searching for:  moneda?           → 0 matches   ❌ same
searching for:  moneda            → N matches   ✅ the bare property survives
```

**Rule: pick the witness for what it IS, not for how it looks in the source.** Source and bundle
are two different texts, and grep only ever sees the second one. An object/field name from the
diff is the most durable witness available; a piece of syntax or a comment you added specifically
to be greppable is the least durable, even though it feels the most deliberate.

## 5. The positive control that closes the method

Before trusting a zero, confirm the search tool can find *anything*: grep the bundle for a string
you are certain is present. If that control also returns zero, the broken thing is the
instrument — encoding, wrong path, a compressed response — not the deploy.

## Corollary: the same defect on the server side, not just the bundle

`curl` returning `200` on a route doesn't prove it's **your** version either. If the platform
exposes a deployment identifier (a response header, a status endpoint), diff that against the SHA
you just pushed — it's cheaper and more honest than grepping for anything.

Counting occurrences inside a served page has its own separate traps (the RSC payload doubling
every string, `grep -c` counting lines instead of matches); see
[counting-strings-in-rendered-html.md](counting-strings-in-rendered-html.md). Proving the build
itself with a positive marker (log line, embedded i18n string) is covered in
[client-mirrored-server-limit.md](client-mirrored-server-limit.md), under "Proving that what is
deployed is the new build."
