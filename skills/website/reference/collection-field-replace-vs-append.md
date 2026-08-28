# A "set of labels" field is usually REPLACE, not append — canary one row first

`PUT /resource/<id>` with `{"tags": ["new-tag"]}` commonly **replaces the whole array**,
not adds one entry. If the row already carried other tags (payment method, currency,
category), sending only the new one silently strips the rest — with a `200` on every call.

The API docs describing the field as `"tags"` with no verb give no warning. The fix is not
clever: **read the row, merge client-side, send the complete list** — but only if you
checked first.

## The procedure that catches it

1. **Canary one row.** Update a single record, then read it back *through the API* and
   confirm every field you did NOT intend to touch is still there. Only then run a batch.
2. **Check for side effects before a burst of writes.** Automations/webhooks that a single
   manual edit never triggers can fire on 20 consecutive updates.
3. **Verify afterward by re-deriving from the source**, not from the payload cached before
   writing. Two counts that must land at zero:

       rows that gained the new value but lost an old one : 0
       rows that should carry the new value and don't      : 0

## Where this shows up

Any "set of labels" field: Notion multi-selects, GitHub issue labels via `PUT /labels`
(vs `POST`), Jira labels, S3 object tagging (`PutObjectTagging` is full-replace by design).
When a field is a collection, assume replace until a canary proves otherwise — the failure
is silent and destroys data the request never mentioned.
