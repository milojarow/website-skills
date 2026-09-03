# An `onChange` (or any JSX-inline handler) with logic inside is untestable by construction

A test gate that falsifies several rules one by one — breaking each on purpose, all failing
correctly — can still be **mostly hollow**. Falsifying a rule breaks the pure helper function the
gate measures, not the handler actually wired into the JSX. If the decision itself is written
inline —

```jsx
onChange={(e) => onChange(e.target.value === SENTINEL ? {...} : {...})}
```

— the gate can measure a helper that exists and is correct, while the real path the user's fingers
walk (the inline arrow function inside the `<select>`/`<input>`) never gets exercised at all. The
gate goes green measuring a fossil.

**Rule:** any handler that DECIDES something about state leaves the JSX and becomes a named,
exported function; the gate measures that function.

```jsx
// before — unmeasurable from the gate
onChange={(e) => onChange(e.target.value === SENTINEL ? {...} : {...})}

// after — the gate measures the same path the screen walks
onChange={(e) => onChange(withSelectValue(data, e.target.value))}
```

**The structural control that keeps it honest:** the gate should also assert the JSX still calls
that function **by name** (e.g. `screenSource.includes('withSelectValue(data, e.target.value)')`)
— otherwise someone re-inlines the logic later and every other control stays green measuring a
function nothing calls anymore.

**Corollary, same family:** a gate written as `includes('<some phrase>')` can pass green with the
element **deleted**, because the same phrase lives in unrelated help text nearby. Anchor gates to
the MECHANISM (the condition, the function name) — never to prose that might coincidentally appear
elsewhere.

See also [unwired-module-created-not-imported.md](unwired-module-created-not-imported.md) for the
sibling failure (code exists and is correct, but nothing calls it) and
[capture-half-fix-reads-not-writes.md](capture-half-fix-reads-not-writes.md) for another gate that
measured the wrong half of a flow.
