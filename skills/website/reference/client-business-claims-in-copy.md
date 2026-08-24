# UI copy that asserts a fact about the client's business needs the client's confirmation, not just a plausible inference

A screen the client opens is the last place an inference can slip through unverified. Internally
it's a note to self; on that screen it's the business talking about itself.

## The case

An admin panel showed a total ad spend that only covered one account. To keep the number from
reading as "all spend, everywhere", a caption was added explaining what it did *not* cover — and
that caption asserted a fact about the client's business (that a specific third party handles a
service the client does not run in-house). The client read it and said it was false. It had been
live for minutes, on the screen he opens himself.

The measurement behind the caption was real and narrow: some identifiers our credential could not
resolve. That supports *"this total is incomplete"*. It does not support naming who or what fills
the gap — that's a guess wearing the caption's certainty.

## Two different things, only one of them publishable

```
what the measurement SAYS      → "this total covers only account X"        publishable
what the measurement SUGGESTS  → "therefore a third party handles Y"        NOT publishable
```

The gap is real; the story explaining the gap is a hypothesis. Ship the gap, not the story.

## How to word a scope warning without asserting more than you measured

```
bad    "The client uses another vendor for X, so that spend doesn't show here."
good   "Only what we launched ourselves. If someone else runs their own account,
        that spend won't show here."
```

The second version is true under every possible explanation for the gap and names no one. It
warns about the *shape* of the blind spot instead of guessing what fills it.

## Placement matters as much as wording

**An incomplete total that looks complete is worse than a visible gap** — a gap invites a
question, a number gets believed on sight. Put the scope warning **next to the number**, not in a
page footnote: whoever is reading a large number rarely scrolls down to find the fine print that
would have changed how they read it.

## Where the false claim tends to enter: relayed inferences

When the underlying data point arrives through another agent or collaborator, it can arrive as a
flat assertion instead of the hypothesis it actually is. Whoever publishes the copy is the last
filter, and the test is not *"was I told this with confidence?"* but *"is this a fact ABOUT THE
CLIENT, and did the client confirm it?"* — confidence in the telling is not the same evidence as
confirmation from the source.

Related: a caption is [read as an assertion about the code](money-dashboard.md#6-the-on-screen-copy-is-the-interface--a-caption-that-lies-is-a-bug-with-no-test)
behind it; this is the same failure one layer up — a caption read as an assertion about the
**client's real-world business**, which no test in the repo can check.
