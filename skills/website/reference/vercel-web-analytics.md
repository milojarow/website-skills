# Vercel Web Analytics — mounting `<Analytics />` is half the job

Adding `@vercel/analytics` and mounting `<Analytics />` in the root layout does **not** turn Web Analytics on. There is a **per-project switch on the platform side, off by default**. With the component shipped and the switch off, the site records **0 visitors, 0 pageviews** — and that history is not recoverable later. Nothing errors, nothing warns.

"I have analytics" is really **three independent conditions**, and they fail independently:

1. **The component is mounted** — repo.
2. **The project switch is enabled** — platform. ← *the one that gets forgotten*
3. **The plan covers what you want to measure** — billing.

---

## The four signals that prove nothing

These are the checks one naturally reaches for. All four can be true while the site measures nothing:

1. `@vercel/analytics` is in `package.json`.
2. `<Analytics />` is mounted in `app/layout.jsx`.
3. The layout chunk contains the string `/_vercel/insights/script.js`.
4. `curl https://<site>/_vercel/insights/script.js` returns **200 with the real tracker** (a few KB of code, not a stub).

The script is served from the edge regardless of the switch. What the switch controls is whether the **ingest is retained**. Seeing the script arrive is seeing half the circuit — the 200 is the shop window; the project's `webAnalytics` field is the source.

## The diagnosis that actually works

Read the project's `webAnalytics` field from the API:

```bash
TOKEN=$(jq -r '.token' ~/.local/share/com.vercel.cli/auth.json)   # linux
PRJ=$(jq -r '.projectId' .vercel/project.json)
TEAM=$(jq -r '.orgId' .vercel/project.json)

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.vercel.com/v9/projects/$PRJ?teamId=$TEAM" | jq '.webAnalytics'
```

| State | What comes back |
|---|---|
| **Off** | `{"id": "..."}` — the id and nothing else |
| **On** | `{"id": "...", "enabledAt": <epoch-ms>, "hasData": true}` |

**The absence of `enabledAt` *is* the diagnosis.** Nothing to interpret.

Via MCP (`mcp__vercel__get_web_analytics`), two responses that mean different things and are easy to confuse:

- `400 web_analytics_not_enabled` → **the switch is off.** Free to fix.
- `402 payment_required` (*"Accessing Analytics custom events requires an Enterprise or Pro plan"*) → the switch is fine; the **plan** is what falls short.

**Always query another project in the same team as a control** before blaming the API. If every project fails the same way, it is still the switch — not a broken API, not missing permissions. A whole portfolio can sit with it off.

## Turning it on

```bash
vercel project web-analytics                 # on the linked project
vercel project web-analytics --format json   # non-interactive, returns {"enabled": true, ...}
vercel project speed-insights                # the sibling, same switch pattern
```

Requires CLI ≥ 57. **No redeploy needed** — the script was already loading from the deployed bundle; the only missing piece was the other side storing. Verify immediately that `webAnalytics` now carries `enabledAt`.

## Free-tier ceiling

With the switch on, the free tier gives **pageviews and visitors, but not CTA clicks**: custom events (`track('cta', {...})`) are Pro-and-up and are **discarded silently** — no error, no charge. To measure outbound clicks (e.g. WhatsApp) at no cost, use the Meta pixel, which an ads campaign needs anyway. See [third-party-scripts-csp.md](third-party-scripts-csp.md).

## Where this belongs in the go-live checklist

Enabling Web Analytics is part of **attaching the domain / going to production**, right next to `sitemap.js` and `robots.js`. It is the same class of pending work: platform work that does not live in the repo, that nobody sees fail, and that only surfaces when someone asks for numbers a month later.

**Short rule: the same commit that removes the `noindex` is accompanied by `vercel project web-analytics`.**
