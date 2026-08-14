# CLAUDE.md

Guidance for Claude Code working in this repository.

## What this is

One Bash script, `xtream`, plus its README. No build step, no test suite.

Edit `xtream` here. A copy may exist elsewhere on the author's machine
(`~/dotfiles/bin/xtream`); this repository is the source of truth, and work
done anywhere else gets lost.

## The job it does

Someone posts a dump of a few thousand IPTV lines. The user wants the handful
worth typing into a TV, knowing before they type that the line works and
carries the channels they care about. Typing on a TV remote is slow, so being
sure beforehand is the entire point.

So filters run cheapest-first, and a sweep over thousands of lines stays
quick. Results print as they land, so a long run can be interrupted without
losing them. A hit shows what the line carries, when it expires, and the three
fields a TV asks for.

## Constraints that are easy to break

- **Bash 3.2.** macOS ships it, and the script must run there. No associative
  arrays, no `mapfile`, no `wait -n`, no `${var@Q}`, no `EPOCHREALTIME`. Use a
  delimited string where you want a set.
- **`set -euo pipefail` is on.** A bare `return` after a failed test returns
  that failure and, inside a loop body, kills the script. Write `return 0` when
  you mean success. Guard greps and arithmetic that may legitimately fail with
  `|| true`.
- **`die` runs inside command substitutions**, where a bare `exit` would only
  kill the subshell. It signals the top shell instead, which is why callers
  need `|| exit 1` on captures.
- **`jobs` does not work for counting workers.** A command substitution runs in
  a subshell with an empty job table, so `$(jobs -pr | wc -l)` always returns
  0. Batch tracks its workers by hand: a `.done` file marks a finished one and
  `kill -0` catches one that died. Both are builtins, so polling costs nothing.
  This bug once launched every line of a dump at once — 3238 processes against
  a 4000 limit.
- **The per-line history helpers must stay fork-free.** `history_key`,
  `history_read`, `history_note` and `fmt_age` run once per line of a dump, in
  the parent, between rendering results. They set a global instead of printing
  precisely so no caller needs `$(...)`, and reaching for `tr`, `sed` or `date`
  inside them puts thousands of subshells back into the hot loop. `HIST_NOW` is
  read once per batch for the same reason.
- **Portability of the small tools.** BSD and GNU differ, so `sed -i`,
  `date +%N` and `grep -P` are out. `column` may be missing entirely, which is
  why `tabulate` has a fallback.

## Working with panels

Real IPTV panels misbehave in specific ways the script already handles. Don't
"simplify" these away:

- They invent HTTP status codes (401, 451, 456, 458, 509, 513, 884). Each is
  explained in `http_probe_stream`; a bare number tells the user nothing.
- They answer a dead channel with a placeholder clip, so a player shows it
  happily. The redirect target is what gives it away.
- They redirect to a tokenised URL on another host, so the probe follows
  redirects and judges the final hop.
- They count a connection for a few seconds after it closes, so `check` retries
  once after a pause rather than calling a live stream dead.
- They file the same content differently: the country in the channel name, or
  in the category. Queries match both.

## Testing

There is no test suite. Verify by running against a real dump:

```bash
bash -n xtream                                    # syntax
./xtream lines.txt lines | head                   # parsing, no network
./xtream lines.txt -n 53 quick                    # one line end to end
./xtream lines.txt -n 1-40 -z london playable uk  # the sweep, small slice
```

Timing matters. A change that makes a 3593-line sweep slower is a regression
even if the output is identical.

## Publishing

The repository is public. Never commit real credentials, including in the help
text and commit messages. Examples use `panel.example.com`, `someuser`,
`somepassword`. Sweep output (`-o`) holds the credentials of every line that
answered, so it stays out of the repository (`.gitignore` covers the usual
names).

Commit messages: conventional-style subject, and a body explaining what was
wrong rather than what was typed. No `Co-Authored-By` trailers.
