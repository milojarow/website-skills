# Phone numbers as identity keys: repairing prefixes, and why two counts that agree can both be wrong

Any feature that answers *"is this the same person?"* — deduplication, "returning customer", campaign lists, counting uniques — resolves to a comparison between two stored strings. When the stored string is a phone number, that comparison is far less reliable than it looks, and it fails **silently**.

## A corrupt country prefix is repairable, because the body carries the evidence

Measured on a booking system's export of ~60 appointments whose `Teléfono` column was stored as bare digits with no `+`: a minority of rows carried a country prefix that was simply wrong — a prefix belonging to an entirely different country — while the rest were well formed.

The key observation is that **only the prefix was broken**. In every damaged row the 10-digit national body was intact and valid, and the body is what identifies the country:

```python
MX2 = {'55', '33', '81'}                    # 2-digit MX metro area codes
area = c10[:2] if c10[:2] in MX2 else c10[:3]
is_us = area in KNOWN_US_AREA_CODES
fixed = ('1' if is_us else '521') + c10
```

So the repair is not a guess. **The stored prefix is the classification the source system made; the area code is the evidence.** When they contradict each other, the area code wins — it is data about the subscriber, while the prefix is data about whatever wrote the row.

### Validate the rule against an answer already in the data

The part that turns a hypothesis into a fact: one of the repaired numbers came out **byte-identical to another row in the same file that had been stored correctly** — the same person, booked twice under two different names. The repair rule was therefore verified against an answer that already existed in the dataset, not against the author's judgment.

Look for this before writing your own test fixtures. A dataset with duplicates usually contains its own answer key: if some records took the broken path and others took the correct one, the correct ones are ground truth for the rule that repairs the broken ones. **If your normalization rule can be proved this way, prove it this way.**

## The corollary that actually bites: two wrong counts that agree

Counting unique customers gave **47 by name and 47 by phone**. Two independent methods, identical answer — and *both were wrong*, because two errors of opposite sign cancelled:

- one person recorded under **two different names** (one of them an emoji) inflated the name count;
- two **different people sharing one name** deflated it.

**Agreement between two methods is not verification when their errors can point in opposite directions.** It is one of the most convincing wrong answers available, because independence is exactly the property that normally makes agreement meaningful. The only trustworthy count came from normalizing first and grouping by the repaired number.

Before believing two matching figures, ask what each method's failure mode *is*, and whether they push the same way. If one over-counts and the other under-counts, their agreement carries no information at all.

## Practical rules

- **Compare by the last 10 digits, never by the whole string.** The same subscriber arrives as `52…`, `521…`, or with no prefix at all depending on which entry point wrote the row.
- **Normalize to E.164 at ingest, not at display.** Everything that groups — dedup, "is a returning customer", segments, campaign targeting — depends on the stored form. Normalizing only in the render layer leaves every aggregate quietly wrong while the screen looks right.
- **Treat the stored prefix as a claim and the area code as evidence.** Re-deriving is cheap; trusting a prefix that another system wrote is not.
- **A number that does not match the local pattern is not an error.** It is usually a customer from another city or country — business data, not garbage to clean. Repair what is provably corrupt; do not "fix" what is merely unfamiliar.
- **Never compare identity by a hand-typed constant.** Same lesson as the NFC/NFD trap in [api-boundary-contracts.md](api-boundary-contracts.md): values that look identical on screen can differ byte-wise.
