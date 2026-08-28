# A test fixture with a hardcoded constant expires silently when the surrounding rule changes

A gate check hardcoded a test value chosen for a specific property — e.g. an installment
amount picked because it does **not** divide evenly, to catch a stray `Math.ceil`. The day
a new business rule shipped (a cap on the number of installments), that same constant
became illegal input, and the gate went red **on its own fixture**, not on the code. Real
time lost chasing a defect that didn't exist.

## Two fixes, and the second one is the one people skip

1. **Derive the test value from the rule, don't hardcode it.**

   ```js
   const legalInstallment = line.minInstallment + 7
   ```

2. **Assert the fixture's own premise.** If the point of the value is that it does NOT
   divide evenly, assert that:

   ```js
   if (total % legalInstallment === 0)
     throw new Error('Fixture lost its edge: divides evenly, no longer exercises rounding.')
   ```

   Without this, the day the value *does* happen to divide evenly the check passes without
   testing anything — and a green that proves nothing looks identical to a green that
   proves something.

## The general rule

**Every fixture with a hardcoded constant is an unwritten expiration date.** If the
constant exists because it satisfies a property, verify the property in the same test. If
it exists because it falls inside a range, compute the range instead of typing a number
that happened to be inside it on the day someone wrote the test.
