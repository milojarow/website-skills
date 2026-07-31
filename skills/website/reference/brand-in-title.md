# Brand searches — if the name isn't in a `<title>` or an `<h1>`, you don't win it

A diagnostic symptom, and it is enough on its own:

> Searching **"ACME ingeniería"** (the full brand) returns the site.
> Searching **"ACME"** or a shorter form of the brand does not.

It sounds like an authority or age problem. Usually it isn't.

## The cause

Check whether the exact string people type appears in a `<title>` or an `<h1>`
**anywhere on the site**. The common failure: every `<title>` ends in the
abbreviation (`| ACME`) and the homepage `<h1>` is purely descriptive
("Crane and lifting-equipment rental"). Result: the string "ACME Ingeniería" appears
in **not one `<title>` and not one `<h1>`** on the whole site.

It may well appear a dozen times — but all in **secondary signals**:

- the logo's `alt`
- the navbar `aria-label`
- `og:site_name`
- the JSON-LD `name`
- the footer copyright

That explains the asymmetry precisely: **the full long phrase is literally present on
the page and nobody else in the world uses it**, so weak signals are enough to win
it. The short phrase has to compete, and it has nothing to compete with.

## The 30-second check

Against the **served** HTML, not the source:

```bash
curl -s https://<domain>/ -o home.html
grep -oP '(?<=<title>).*?(?=</title>)' home.html
grep -oP '(?<=<h1[^>]*>).*?(?=</h1>)'  home.html
```

And the question: **does the string people type appear there?** Not "the name is on
the page" — it has to be *in the title or the heading*. If it only lives in `alt`
and JSON-LD, it does not count for this fight.

Do it on EVERY page, not just the homepage:

```bash
for p in "" path/one path/two; do
  t=$(curl -s "https://<domain>/$p" | grep -oP '(?<=<title>).*?(?=</title>)' | head -1)
  printf '%3d  /%-28s %s\n' "${#t}" "$p" "$t"
done
```

## Fixing it without sacrificing the commercial searches

**Do not invert the title.** The temptation is to put the brand first; that steals
room from the term that actually brings customers. What matters is that the string
IS there, not that it comes first.

Pattern that works, measured against the character budget (~60 before Google starts
truncating the display):

```
<Service> en <City> | <Brand>      ← 59 chars, fits
```

And in code, **a dedicated constant for the title suffix**, distinct from the
abbreviation used in `alt` and in prose:

```js
shortName:  'ACME',              // for alt and for "I saw the ACME site"
titleBrand: 'ACME Ingeniería',   // ONLY for the <title> suffix
name:       'ACME Ingeniería & Mantenimiento',  // og:site_name, JSON-LD, footer
```

Reusing `shortName` for the title "because it's the short name" drops you straight
back into the same hole. These are three different jobs.

**The FULL name stays intact** in `og:site_name`, the JSON-LD `name`, the footer and
the logo `alt` — that is where it has to match the Google Business Profile listing
letter for letter. See [google-business-profile.md](google-business-profile.md).

**And put the brand in VISIBLE TEXT**, not only in attributes. If the navbar logo is
an image, the company name may not be written anywhere on the homepage except the
copyright line. A micro-label above the `<h1>` ("Brand · City") solves it without
touching the headline, which is doing the commercial work.

## Before promising the client #1 on their own name

Search the string and look at **who holds those results today**. An abbreviation is
frequently also an established term in the same language — a teaching method, a
standard, a protocol — published by universities or institutions with years of domain
authority. That is not fixable with code, and it has to be said up front.

What decides that fight is the **entity**: a verified Google Business Profile listing
is what teaches the search engine that this string is this business and not the other
thing.

Also check the exact-match domain: `<brand>.com` without hyphens may exist and be
occupied by an abandoned shell. Open it before forming an opinion — a dead
placeholder holds the name but does not defend it.
