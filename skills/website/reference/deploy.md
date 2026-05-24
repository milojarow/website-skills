# Deploy, errors & CDN

## Builds — never local

Don't run local production builds. **Let Vercel build.** Read build logs with `vercel inspect`. Local builds drift from the Vercel environment and waste time chasing differences that won't exist in production.

## Error handling

For async operations, use `try / catch / finally`:

- **catch:** show a user-friendly message; hide technical details from the UI.
- **finally:** reset loading states here — so a thrown error never leaves a spinner stuck spinning.

## External CDN — no double hosting

If the project serves media from an external CDN (e.g. `cdn.example.com`), put **`public/` in `.gitignore` from the initial scaffold.**

Committing local copies of CDN-served assets creates **double hosting**: Vercel's CDN ships duplicates while the real CDN serves the actual assets — wasted bandwidth, drift between the two copies, and confusion about the source of truth. The external CDN is the source of truth; the repo must not carry copies.

If a project already double-hosts, the cleanup is: add `public/` to `.gitignore`, purge the tracked copies from history, force-push, and GC. (Detection + full cleanup procedure live in the operator's notes — this is a heavy, history-rewriting operation, so confirm before running it.)
