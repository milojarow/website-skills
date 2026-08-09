# Domain email auth (SPF/DKIM) — verify from outside, never from your own log

Once a domain is attached, it usually starts **sending mail** too — transactional notifications, contact-form relays, the operator's own mailbox. Getting DKIM signing to work is the easy half. Knowing whether it works is where the real trap is.

---

## "Success = silence": grepping your own log for a signal the system never emits

Measured on a Haraka + mailauth submission server while setting up DKIM for a new domain.

The internal runbook said:

> *to confirm DKIM is signing, look for `DKIM signed!` in the submission log.*

That string appears **0 times in the entire log** — not just for the new domain, but for **every** domain, including ones that had been signing correctly for months and were independently verified healthy.

So the documented procedure for verifying DKIM produces a **guaranteed false negative**. And the natural next step for whoever follows it is to *regenerate the DKIM key* — which does break mail that was healthy, on an invented cause.

### The general shape

**A log that only records failure cannot confirm success.** In systems like this the happy path is silent by design; only the error branch writes. Searching there for a positive signal returns empty *always*, and empty reads exactly like "it's broken".

The useful half does exist, and it is the error half:

```
mailauth/dkim_sign error: ENOENT      ← real, and real when the key symlink is broken
```

What does not exist is the positive confirmation.

This is worse than a resolver with a negative cache — that one at least belongs to a third party. This came signed by the house, in your own documentation, with all the authority that carries.

---

## What to do instead: an external verifier

The only instrument that measures what actually matters — what the **receiver** sees, not what the sender believes:

```
# Send an AUTHENTICATED message over 587, from an account on the domain, to:
check-auth@verifier.port25.com

# It replies with a report:
#   SPF check:   pass
#   DKIM check:  pass    ← ID(s) verified: header.d=<your-domain>
```

Authenticated submission matters: a message that skips the submission path may not take the signing path either, and then you have verified nothing.

---

## The rule this generalizes to

**Before using a grep as proof that something works, run it against a case you KNOW is healthy.** If the known-good control also comes back empty, the *instrument* is broken, not the thing you were measuring.

It is the "log" variant of the independent-reference rule: if your known-good control fails the test, the test is the suspect.

---

## Noise that is NOT the defect (so nobody chases it)

The same log carried several `dkim_sign ENOENT` errors looking for a key named after the server's **hostname** rather than after a domain. Those do not prevent signing — a message went through that same path and came out DKIM `pass`. It is the MTA attempting to sign with the HELO identity.

Verified before touching anything, and correctly left alone. A loud error next to the thing you're debugging is not automatically the cause of it.
