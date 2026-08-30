# An average of ratios is not a rate — and it prints identically to one

Citing `sum(fees) / sum(prices)` across a handful of transactions as "the rate" is calculated,
printed, and quoted exactly like a real measured rate. It can still be a label error, not a
number error — the number is correct, the claim about what it measures is not.

## The tell

Measured case: a weighted average of 4.31% across seven sales was reported as "the business
rule." None of the seven individual sales carried anything close to that number — the real
range was 1.8% to 8.4%, more than a 4x spread. Two line items with the **identical** price
also carried the identical fee amount in currency, not the identical percentage — the signal
that the underlying rule is a flat amount, not a rate at all.

**The check costs one line:** does any individual row actually show this number? If the
average appears in zero observations, it's a summary, not a rule. And if the spread across
rows crosses an order of magnitude, the average describes nothing — it's not hovering around
a shared value, it's mixing unrelated ones.

## The same defect one layer up: the metric measures the wrong set

A separate count in the same session — "40 of 111 names match the lookup table" — was reported
as "the vocabulary doesn't exist." The vocabulary existed, fully, in a *different* table that
covered 108 of the 111. **The count measured one table; the conclusion spoke about the whole
system.** The number was right and the sentence built on top of it was about a bigger set than
the one actually sampled.

Before sending any measurement, write the boundary of what was measured in one sentence
("N out of the rows in table X"), and check that the conclusion doesn't silently widen that
boundary to "the data" or "the system" or "the vocabulary" in general.

## Rule

1. A ratio of two sums is a *summary statistic*, not a business rule, until at least one real
   observation actually equals it.
2. High dispersion across the underlying rows (order-of-magnitude spread, or identical inputs
   producing identical absolute — not proportional — outputs) is itself evidence that no single
   rate is being measured.
3. State the exact set a count was taken over before quoting the count elsewhere — a number is
   only as wide as the set it came from.
