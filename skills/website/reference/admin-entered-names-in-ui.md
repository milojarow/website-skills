# Admin-entered names break a UI three ways, all silent

Applies to any UI where the CONTENT is named by an admin from a panel, and the app then
renders that name somewhere else: a product/variant name, a category label, a customer
name — anything typed once and displayed in a different control later.

## 1. Truncation keeps the FIRST words

A name inside a `truncate` / text-overflow control keeps whatever comes first. So the word
that actually distinguishes two options must go first, and any variant/qualifier goes
after a separator:

    "Small order of fries" (22 chars) -> "Small order of fr…"   ← hides the product
    "Fries · small order"  (20 chars) -> "Fries · small ord…"   ← product stays visible

Both say the same thing; only one survives truncation. Measure against the **longest
label already living in that control**, never against an imagined width. This bites
hardest when the control exists specifically to disambiguate two neighboring options — a
name that hides the distinguishing word defeats the reason the control was built.

## 2. Grammatical agreement breaks with some names, not others

A template like `"{name} added to the order"` bakes in a fixed number/gender that agrees
with some inputs and not others (in any language with grammatical agreement — es, fr, pt,
it, de…), and whoever typed the name in the admin panel has no reason to know they're
writing the subject of a sentence.

The fix is not guessing gender/number, or demanding grammatical metadata on entry — it's
rewriting the sentence so it doesn't depend on the name:

- if the name already appears nearby as a heading, the sentence can omit it entirely
- use an impersonal / third-person-singular verb: "Added to order", "Now in cart…"
- counts belong in a stepper/badge, never inside the sentence ("2 ×" forces a plural)
- in an `aria-label`, the article agrees too: `Remove ${name}` needs the same rewrite

Bonus: present tense is usually also the more *correct* one for these cases, since the
copy describes editable STATE, not a past event — it stays true even after the user
reduces the quantity to zero.

## 3. The same name is read by SUBSTRING in one place and by EXACT MATCH in another

A classifier that matches by substring files the item under the wrong category when a
name changes. A lookup map keyed by exact string stops matching on the very next rename
and falls back silently, with no error and no log — so the defect surfaces far from its
cause. Renaming from an admin panel has a wider blast radius than it looks.

## Verification

Run the real classifier/lookup functions against candidate names, with a **positive
control** alongside — live names that must still classify correctly. If the control
doesn't distinguish a working case from a broken one, the instrument proves nothing and a
green result means nothing.
