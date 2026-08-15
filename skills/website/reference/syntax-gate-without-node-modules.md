# A JSX syntax gate with no `node_modules` and nothing to install

House rule is that nothing builds locally — Vercel builds ([deploy.md](deploy.md)) — so a
checkout often has **no `node_modules` at all**. The practical consequence is that there is
**zero command that can say no** before a push: the only referee is a remote build you have to
remember to look at, and a CLI deploy can report success while the build itself ends in `Error`.

`node --check` does not help: it does not parse JSX. But **`bun`'s transpiler parses JSX/ESM
without resolving imports**, so it needs no dependencies installed:

```js
// syncheck.js — exit 1 if anything fails to parse
const t = new Bun.Transpiler({ loader: 'jsx' });
let bad = 0;
for (const f of Bun.argv.slice(2)) {
  try { t.transformSync(await Bun.file(f).text()); console.log(`ok    ${f}`); }
  catch (e) { bad++; console.log(`FAIL  ${f}\n      ${String(e).split('\n').slice(0,4).join('\n      ')}`); }
}
process.exit(bad ? 1 : 0);
```

This covers exactly the class of damage a blind edit introduces: a comment block closed early
that leaves prose sitting as code, one brace too many, an unclosed JSX element. It does not cover
types or runtime behaviour, and does not pretend to.

## Testing LOGIC (not just syntax) needs the path alias

If a module imports through `@/…`, bun resolves that alias from `jsconfig.json` / `tsconfig.json`
**only for files inside the project**. A test living in a scratch directory fails with "Cannot
find module". Copy the test into the project, run it, delete it:

```bash
cp <scratch>/norm.test.js ./.tmp-norm.test.js && bun ./.tmp-norm.test.js; rm -f ./.tmp-norm.test.js
```

That is enough to import pure normalizers and helpers (even from a file marked `'use client'`)
and assert against the **real** payload shapes — copied from a `curl` of the backend — instead of
the shapes you imagine. Same principle as
[typing the response you measured](api-boundary-contracts.md).

## Three traps that bite the same day you build the gate

1. **A shell without word-splitting.** In `zsh`, `FILES="a.js b.js"; bun check $FILES` passes
   **one** argument containing a space and fails with `ENOENT: … 'a.js b.js'` — which reads as
   "those files don't exist" and sends you looking in the wrong place. Pass the file names
   literally, or split explicitly (`${=FILES}`).

2. **`cmd; echo; echo "exit=$?"` reports the exit code of the `echo`.** The screen shows an error
   and `exit=0` right underneath it. **A gate that lies about its own exit code is worse than no
   gate.** Read `$?` immediately, with nothing in between.

3. **Verify the instrument before believing it.** Feed it one deliberately broken file and one
   healthy file, and run it **twice with nothing changed**. Also check it catches the exact shape
   of the last bug you introduced by hand — if it doesn't, it is decorative.
