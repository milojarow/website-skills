# A fix that lists and asks but does not SAVE is worse than not touching anything

Fixing a capture screen: the READ side and the WRITE side usually duplicate the same
validation, and fixing only the first produces the worst possible state — the screen shows
the object, asks for its fields, lights up the Save button, and on Save answers **"that
object doesn't exist."** The person types, sees the button light up, and walks away believing
it saved.

**This never shows up reading the diff.** The diff looks perfect. It only appears when someone
actually captures data.

## The positive control is doing the action, not reading the code

After touching any capture path, the minimum verification is:

1. Open the screen and **type a real value**.
2. Press Save.
3. Confirm it landed **in the store**, not just on screen.
4. **Refresh** and confirm it is still there.
5. Confirm the **counter moved by exactly one**.
6. **Delete the test data** if it isn't a legitimate business record.

Step 5 is what separates "it saved" from "it painted." Steps 3 and 5 together also catch the
second defect in this family:

## The screen that saves correctly and never finds out

Same day, same module: capture saved correctly — author and reason both correct, verified in
the store — **and the screen kept saying "we don't know yet"** with the field blank. The
subscription to the store was missing.

Same defect, different face: **the gap between "it saved" and "it shows."** If the module has
a `useVersionOfX()` / `useSyncExternalStore` pattern, every screen that writes to that store
must subscribe — including the ones that only display it "in passing."

⚠️ And if the derived values live in a `useMemo`, the store's version has to be in the
dependency array. The linter will flag that dependency as unnecessary — it can't see that the
function reads module-level state. Between silencing the linter and removing the memo:
**remove the memo**, when the element count is small. A forgotten dependency there doesn't
throw — it quietly teaches a stale value.
