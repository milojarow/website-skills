# Layout: stability + debugging

## Layout stability — siblings never move

Dynamic content (tooltips, descriptions, expanding panels, spawned elements) must **never** displace sibling components. Reserve the space up front:

- Fixed heights or `min-h-*`
- Absolute positioning for overlays
- Reserved / placeholder space

Other components stay immovable regardless of what spawns nearby. A panel that pushes its neighbors when it opens is a bug, not a behavior.

## Layout debugging — trace upward first

When a layout is broken (elements cramped, empty space, wrong alignment), **do NOT patch the leaf component.** The constraint is almost always in an **ancestor**.

Read upward, in order, looking for `maxWidth`, `flex`, `width`, `padding` that caps the child's space budget:

1. The page file
2. Layout files
3. Root layout
4. Global CSS

Only modify the leaf after confirming **every parent in the chain is clean.** Patching symptoms while a parent constraint is still active wastes iterations and looks like flailing.

**Screenshot clue:** when the user marks empty space in red, that geometry means an ancestor refuses to use that space → **fix the ancestor, not the child.**
