# Stack & conventions

## The DeployNow stack

- **Language / runtime:** JavaScript, Node.js
- **Framework:** React + Next.js **App Router**
- **Styling:** Tailwind CSS + **DaisyUI**, mobile-first responsive
- **Auth:** NextAuth
- **Deploy:** Vercel

No extra UI or icon packages unless genuinely necessary — DaisyUI + Tailwind cover most needs. Every dependency is weight; justify it before adding it.

## Design bar

Production-grade and **distinctive** — not cookie-cutter, not generic-AI-looking. Beautiful, responsive, ready for production.

## File order inside a component file

Top to bottom:

1. Exported component
2. Subcomponents
3. Helpers
4. Static content / constants

## Naming

| Thing | Convention | Example |
|---|---|---|
| Component files | PascalCase, **prefixed by type** | `ButtonAccount.jsx`, `CardAnalyticsMain.jsx` |
| Other files | kebab-case | `format-date.js` |
| Directories | kebab-case | `user-settings/` |
| Variables / functions | camelCase, descriptive + auxiliary verbs | `isLoading`, `hasError` |
| Components | PascalCase | `CardAnalyticsMain` |

**Type prefixes** (the component's role, then the specifics): `Button`, `Card`, `Modal`, `List`, `Form`, `Nav`, `Section`, `Hero`, `Table`, `Layout` — e.g. `ButtonAccount`, `CardAnalyticsMain`, `ModalConfirmDelete`, `ListTransactions`.

## Functions

Use the `function` keyword for pure functions (not arrow consts) — hoisting + readability. Keep conditionals concise; skip unnecessary braces for single statements.

## Language

English for all code — variables, functions, comments, error messages, URLs, commit messages. Spanish only for UI-facing text and labels.

## JSX unicode-escape gotcha

Inside JSX **text** (between tags), a backslash-`u` unicode escape renders as its **literal characters**, not the accented letter it encodes. Always type the real character (á, é, í, ó, ú, ñ, ¿, ¡) directly in JSX text.

JS string literals — attribute values, JS expressions, or `.js` non-JSX files — **do** interpret unicode escapes, so both forms work there; type the real characters everywhere for consistency.
