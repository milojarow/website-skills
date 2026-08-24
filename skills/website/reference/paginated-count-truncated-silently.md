# A count from a paginated API lies without warning — and the truncated number still looks plausible

Counting rows/comments/records by asking for one page and totalling what came back.
The endpoint returns exactly `limit` items, **never says there are more**, and the
resulting number is plausible enough to report and to build on.

## Two measured cases, same day, two different call sites

```
head -12 on a list request          →  "12 services"      real: 43
limit=100, no pagination (Graph API) →  "117 comments"     real: 269 (across 369 posts)
```

Neither call *failed*. Both answered **exactly** what was asked: "give me 12", "give
me 100". The defect is in reading the response as the whole universe.

The second case was expensive because the truncated number carried a business
conclusion ("not urgent, there's only a few") that direct human observation
contradicted. The measurement was the thing that was wrong.

## The check, before reporting any count

**The cheap question:** *is the count I got equal to the limit I asked for?* If you
asked for 100 and got 100, assume there is more. If you asked for 100 and got 83,
it's actually done.

**The positive control:** re-run the same count with a different limit. If the total
changes with the limit, the total isn't a total — it's a page.

```
limit=100  →  117     ← the limit is deciding the answer, not the data
paginated  →  269
```

## Paginating a Graph API correctly

`paging.next` arrives as a **complete URL** with the token already embedded — follow
it verbatim until it's absent, never reconstruct it:

```python
d = get(endpoint, limit=100)
while d and d['data']:
    accumulate(d['data'])
    nxt = d.get('paging', {}).get('next')
    if not nxt:
        break
    d = get(url=nxt)   # the full URL, not rebuilt by hand
```

Guard rails:

- Put a safety cap on the accumulator (e.g. 600) so a `next` that never ends can't
  hang the process — but **say so if the cap is hit**, or you've reintroduced the
  same defect wearing different clothes.
- Watch for **nested** pagination: counting comments across N posts requires
  paginating **two** levels — the posts AND the comments of each post. The inner
  level truncates more easily because the outer one looks complete.

## The rule

Sibling of "a zero needs a positive control" — except here the wrong number isn't a
zero, which is what makes it worse: a zero draws attention on its own, a plausible
number doesn't. Before reporting any count sourced from a paginated API, ask: *did
the data decide this number, or did my limit?*
