# `echo "$JSON" | jq` breaks under zsh — and only once the data contains newlines

The most common pattern in the world for inspecting an API response:

```bash
R=$(curl -s "$API/...")
echo "$R" | jq .
```

Under **zsh**, that `echo` interprets escape sequences. A `\n` that inside the JSON is a
**valid** two-character escape becomes a **real** line break, which invalidates the
document. `jq` answers:

```
jq: parse error: Invalid string: control characters from U+0000 through U+001F
    must be escaped at line 2, column 10
```

Measured with the same string on the same machine:

| form | result |
|---|---|
| `echo "$J" \| jq` (zsh) | **parse error** |
| `echo "$J" \| jq` (bash) | ok |
| `printf '%s' "$J" \| jq` | ok |
| `jq . <<<"$J"` | ok |
| `print -r -- "$J" \| jq` | ok |

Control for the experiment itself: `echo 'a\nb' | wc -l` gives **2** lines under zsh while
`printf '%s'` gives 0 — the `echo` really is interpreting.

## Why it is so treacherous

**It only shows up once the data carries newlines.** The command can work for weeks against
single-line records and break the day someone stores a multi-line note. The natural reading
at that moment is "the record is corrupt" or "the API returned garbage", and both are false:
the JSON arrived intact and was corrupted **in the pipe**.

Worse, the failure does not stay put. In a script, an `R=$(... | jq -r .id)` that blows up
leaves the variable **empty**, and the next three lines build truncated URLs that return
`404`. That 404 reads as "it does not exist" or "I lack permission" — two wrong diagnoses
chained to an `echo`.

## The rule

Under zsh, **never pipe JSON through `echo`.** Any of these works:

```bash
printf '%s' "$R" | jq .
jq . <<<"$R"
curl -s "$API/..." -o /tmp/r.json && jq . /tmp/r.json     # most robust in scripts
```

The third one is the one for scripts: the data never passes through shell interpretation at
all.

## The underlying pattern

This belongs to the same family as the counting traps in
[counting-strings-in-rendered-html.md](counting-strings-in-rendered-html.md): a tool
transformed the data on its way through, and the result still looked like a legitimate
answer from the system under test. Other members of the family seen in a single session —
anchoring a grep with `[^<]*` against React's `<!-- -->` separator, comparing two counts
with different anchors, a URL missing a path segment, and `set -- $var` unquoted in a shell
that does not word-split.

Hence the rule worth carrying: **when the verdict is "it fails" or "it is corrupt", the
first suspect is the instrument** — especially when the system just passed an independent
check by another route.
