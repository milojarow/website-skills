# Lint on a dirty repo: the verdict is the DELTA, measured at the same relative path

Most real repos carry pre-existing lint findings. On one Next.js + flat-config
(`eslint.config.mjs`) repo the count was **50** (`react-hooks/set-state-in-effect`,
`react-hooks/refs`, …). With that much standing noise, "run the lint" gates nothing:
neither green nor red means anything. The only question with an answer is **did my
change add findings?** — the delta against `HEAD`.

## The trap: a baseline linted from `/tmp` runs FEWER rules

The natural way to get the "before" state:

```bash
git show HEAD:src/components/Foo.tsx > /tmp/base/src/components/Foo.tsx
bunx eslint --config ./eslint.config.mjs /tmp/base/src/components/Foo.tsx
```

Measured: **0E 1W** for the baseline vs **5E 1W** for the working tree. Read
head-on, that says "you introduced 5 errors". It is **false**.

A flat config scopes rules with globs (`src/**`, `**/*.tsx`) resolved **relative to
the config file's directory**. A copy sitting under `/tmp` matches none of them, so
the baseline was linted with a smaller rule set. The comparison used two different
rulers and fabricated a regression that never existed.

This is worse than a false negative. An agent that believes the number goes and
"fixes" healthy code, or reports to the user that it broke something it did not touch.

## The correct measurement: a worktree at the same relative path

Same relative path, same `node_modules`, same command, same formatter:

```bash
WT=<scratchpad>/wt-base
git worktree add --detach "$WT" HEAD
ln -sfn "$PWD/node_modules" "$WT/node_modules"     # eslint must resolve the plugins

# baseline
(cd "$WT" && bunx eslint src/components/Foo.tsx -f json \
  | jq -r '.[0] | "\(.errorCount)E \(.warningCount)W lines=\([.messages[].line]|join(","))"')

# working tree — byte-identical command
bunx eslint src/components/Foo.tsx -f json \
  | jq -r '.[0] | "\(.errorCount)E \(.warningCount)W lines=\([.messages[].line]|join(","))"'

git worktree remove --force "$WT"; git worktree prune
```

Real result on the same change: **5E 1W** before and **5E 1W** after — with the line
numbers shifted (`460→494`, `848→888`, `1226→1308`) by exactly the amount inserted
above them. Delta zero.

## Details that make it hold up

- **Ask for the LINES, not just the count.** Pairing six old findings with six new
  ones and confirming the shift matches what was inserted is what upgrades "same
  number" into "same findings". Two counts that agree can still hide one finding
  fixed and another introduced.
- **For a finding that lands inside the diff**, close it on the `-` side of
  `git diff`: if the flagged construct was already in the replaced line, the finding
  is pre-existing even though its line number is new.
- **Lint only the touched files** (`git diff --name-only`) so the report stays readable.
- Put the worktree in the scratchpad and remove it with `--force` + `git worktree
  prune`, so no stale entries are left inside `.git`.

## The general rule

A lint or typecheck count **without a baseline is not evidence** — the verdict is
the delta. And when the delta comes back suspiciously large or suspiciously clean,
**the suspect is the instrument, not the code**: verify the measuring device before
you believe anything about the thing measured.
