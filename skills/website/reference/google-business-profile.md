# Google Business Profile — the lever above the organic results

For a **local business**, the listing outranks the site. The map block sits above
the ordinary results, and Google ranks it by *relevance, distance and prominence*.
A perfect sitemap does not compete with it. Sitemap + robots are the **necessary**
condition; the listing is the one that moves the needle. See
[indexing-and-search-console.md](indexing-and-search-console.md) for the sitemap half.

Audit the listing by **measuring it with the Places API**, not by reading screenshots.

## Declared hours behave like the entry requirement — reviews do not

Measuring a generic local query (`<trade> <city>`) with `textsearch` and checking
which results carry `opening_hours` shows a clean split: the listings occupying the
top positions of the local block **all declare business hours** — including ones with
**zero reviews** — while the listings below that line mostly declare none.

That inverts the usual assumed order of work: **reviews are not the price of
admission to the top block; declared hours behave like it is.** Hours are also the
cheapest item on the list — filled in a minute, and they don't depend on anyone
writing anything.

The second effect is not a correlation at all: the local results screen carries an
**"Open now"** chip at the top. **With no declared hours, that filter removes the
listing entirely.**

## How to measure it instead of guessing

Requires the **Places API** enabled on the key (it is not on by default alongside
Geocoding/Routes).

```bash
# What does the listing actually hold?
curl -s "https://maps.googleapis.com/maps/api/place/textsearch/json?query=<business+city>&region=mx&key=$K"
# → place_id

curl -s "https://maps.googleapis.com/maps/api/place/details/json?place_id=<ID>&language=es&key=$K&fields=name,formatted_address,formatted_phone_number,website,opening_hours,business_status,rating,user_ratings_total,geometry,photos"
```

**Missing fields come back ABSENT, not `null`** — `opening_hours`, `rating`,
`user_ratings_total`. That absence IS the finding; a `jq` expression that expects a
null will quietly report nothing.

To place a business against its competition, run `textsearch` on the **generic term**
and test `'opening_hours' in result` for each hit. That is the table above.

## Listing defects to look for, ALWAYS, in this order

1. **The pin.** Reverse-geocode the coordinates from the listing URL and compare
   them against the real street address. Being off by ~1 km — different street,
   different neighbourhood, different postal code — is a real and common failure.
   It matters more than it looks: distance is measured from the searcher **to the
   pin**, and if verification is requested by postcard, the letter arrives at a
   stranger's house.
2. **The website field.** It often points at a generic auto-generated
   `business.site` page someone created years earlier. The listing has been sending
   its traffic somewhere else the whole time.
3. **The phone.** An empty field on a business whose conversion IS the phone call.
4. **The hours.** See above.
5. **The photos.** A listing with 2 photos while the site's CDN holds dozens.

## Search Console and Business Profile verify different things

This is the confusion worth clearing up early — you can hold one and not the other
with no contradiction:

| | What it asks | How it's proved |
|---|---|---|
| Search Console | do you control this **domain**? | TXT record in DNS |
| Business Profile | do you operate this **premises**? | postcard, phone call or video of the site |

Whoever builds and hosts the site proves the first legitimately. The second belongs
to the business owner, and no amount of domain control substitutes for it.

**The provider must NOT claim the listing as its own.** The owner goes in as
*primary owner*, the provider as *manager* — Google documents that a manager can
edit every URL, reply to reviews, post, and pull statistics, and that *"an agency can
serve as a manager without owning the profile"*. The only things a manager cannot do
are add/remove users and delete the profile, which is exactly what it should not be
able to do. Reviews are unrecoverable if the relationship ends badly: they belong to
the business, not to whoever built the page.

Access should also come from a company account, not from the personal Gmail of
whoever happens to be doing the work.

## Sign you are editing as the public, not as the owner

If Google's confirmation emails say **"Claim this business"** or use *Local Guide*
language, the edits are going in as anonymous suggestions: they enter moderation and
can be delayed or rejected. The owner's interface says **"Edit profile"**; the
public one says **"Suggest an edit"**.

Suggestions **do land** — address, pin, phone and website have all been observed
applied — but they grant no control, no review replies, and no statistics.
