# xtream

A small terminal tool for working through IPTV lines: which ones are alive, which
carry the channels you want, and which actually stream them.

The job it was built for: you have a text file with a few thousand lines shared
by someone, you want the handful worth typing into a TV, and typing anything
into a TV with a remote is slow enough that you only want to do it once.

```bash
xtream lines.txt -z santiago -m tnt playable chile -o good.txt
```

Read as: of every line in the file, keep the ones on a Chilean server, carrying
channels matching "chile", with TNT among them, that will actually stream one.
What comes back is ranked, best first:

```
  1. line 53 · someuser  1270 ch  must 1/1  exp 2026-10-11  conn 1/3  America/Santiago
     GEO CHILE (REGIONAL) 1062 · ZAPPING CHILE 98 · FUTBÓL CHILE 34
     http://panel.example.com:8080/get.php?username=someuser&password=…
     tv: panel.example.com:8080  ·  someuser  ·  somepassword
```

The middle line is what it carries, by category and size, because 1270 channels
are worth nothing if none of them is the football. The last line is the three
fields a TV asks for, URL-decoded and without the `http://` that every app
assumes and nobody enjoys typing on a remote.

It speaks two protocols:

- **Xtream Codes** — the usual `get.php` / `player_api.php` API (username + password)
- **Stalker / Ministra portals** — MAC-based, no username/password

…and it reads a whole **batch** of lines from a local file or a base64 paste
code, running any command against every line in parallel.

> **Where the codes come from.** The base64 paste codes this tool decodes are the ones shared on the subreddit **[r/IPTV_ZONENEW](https://www.reddit.com/r/IPTV_ZONENEW/)** — each post's "Xtream" text is a base64-encoded [paste.sh](https://paste.sh) link holding a stack of lines. Copy that text and hand it straight to `xtream`.

---

## Platform

It's a single Bash script (Bash 3.2+, so stock macOS is fine) using standard Unix tools.

- **macOS** — works out of the box.
- **Linux** — works out of the box.
- **Windows** — not native (it's a Bash script), but two supported paths:
  - **WSL** (recommended) — [Windows Subsystem for Linux](https://learn.microsoft.com/windows/wsl/install) *is* Linux, so it runs exactly as above. `wsl --install`, then treat it like any Linux box.
  - **Git Bash / MSYS2** — the script itself is portable, but the extra tools (`jq`, `python3`, `ffmpeg`, `mpv`, `fzf`) aren't bundled; install them via [MSYS2](https://www.msys2.org/)'s `pacman`. Minimal Git Bash may also lack `column`, in which case tables print unaligned but still work.

## Requirements

| Tool | Needed for |
|------|-----------|
| `bash`, `curl`, `jq` | everything (core) |
| `python3`, `openssl` | decoding base64 / paste.sh dumps and Stalker portals |
| `ffmpeg` (`ffprobe`) | `check` — verifying a stream really plays |
| `mpv` | `play` — opening a channel |
| `fzf` | the picker in `play`, only when a query matches several channels |
| `op` (1Password CLI) | optional, only if you store credentials as `op://` refs |

Install the core tools with your package manager, e.g.:

```bash
# macOS (Homebrew)
brew install jq ffmpeg mpv fzf

# Debian/Ubuntu
sudo apt install jq ffmpeg mpv fzf python3 openssl
```

## Install

Drop the single `xtream` script somewhere on your `PATH`:

```bash
curl -fsSL https://raw.githubusercontent.com/Klaudioz/xtream-cli/main/xtream \
  -o ~/.local/bin/xtream
chmod +x ~/.local/bin/xtream
```

Make sure `~/.local/bin` is on your `PATH`. That's it — no build step.

---

## Quick start

### Work through a file of many lines

Point it at a text file of `get.php` URLs (or a base64 code from a post). Every
line gets a number, and that number addresses it for the rest of the session:

```bash
xtream lines.txt lines                          # the index: which URL is line 53
xtream lines.txt -z santiago playable chile     # the sweep
xtream lines.txt -n 53 search chile             # what line 53 carries
xtream lines.txt -n 53 check 98                 # does that channel really play
xtream lines.txt -n 53 play 98                  # watch it before committing
```

The sweep has three filters, cheapest first:

| Flag | What it does |
|------|--------------|
| `-z RX` | keep lines whose server reports a matching timezone (`santiago`). It comes from a tiny request, so lines that fail it never download a catalogue — this is what makes thousands of lines quick. Expired lines drop out here too. |
| `QUERY` | matches a channel's name **or** its category, because panels split evenly between "TNT CHILE PREMIUM HD" filed under DEPORTES and "TNT SPORTS 1" filed under FUTBÓL CHILE. `-C RX` restricts to the category alone. |
| `-m LIST` | channels the line has to carry, comma-separated regexes, matched inside what QUERY already selected. That scoping is what disambiguates the half-dozen `TNT SPORTS <country>` feeds. |

Everything else is counted and left out: wrong timezone, expired, no matching
channels, missing a must-have, lists them but refuses to stream, host long dead.
`-a` keeps every line and says which of those it was. Results print as they land
and again ranked at the end, so a long run can be interrupted without losing
what it found. `-o FILE` mirrors the run into a file, and `-n @FILE` reads the
line numbers back out of one.

### Test a single line someone sent you

Paste a provider's URL directly — no setup. With no command it prints a quick
verdict (status + categories + channel count):

```bash
xtream 'http://host:8080/get.php?username=USER&password=PASS&type=m3u_plus'
xtream 'http://host:8080/get.php?username=USER&password=PASS' search 'espn'
xtream 'http://host:8080/get.php?username=USER&password=PASS' check 'espn hd'
```

> Quote the URL — an unquoted `&` backgrounds the command.

### Rank a batch without caring what it carries

```bash
xtream lines.txt                       # one-line status of every line
xtream lines.txt --stats status        # …plus live/movie counts per line
xtream lines.txt counts                # channel counts per line
```

Lines on a shared server hand out the same channel list, so identical results
are folded together — which conveniently also flags any line whose package
differs.

### Paste formats it understands

The decoded paste can be any of these — the tool auto-detects the shape:

- a list of `get.php` / `player_api.php` URLs (or `M3U:` lines containing them)
- a host line + a `user  pass` table
- a host URL + `Username:` / `Password:` (or `User:` / `Pass:`) pairs
- Stalker `Panel ➤ … / Mac ➤ …` blocks (below)

### Stalker / Ministra portals

Pastes with `Panel ➤ … / Mac ➤ …` blocks are auto-detected — the **MAC is the credential**. Everything works the same:

```bash
xtream '<base64-code>'                                    # status per MAC
xtream '<base64-code>' search 'bein sports'
xtream 'stalker:http://portal.example:80/c/#00:1A:79:XX:XX:XX' check 'bein sports 1'
```

## Commands

| Command | What it does |
|---------|--------------|
| `playable [QUERY]` | does this line carry these channels **and** stream them? filters on `-z` / QUERY / `-m`, ranks what survives, prints the URL and the TV login fields |
| `lines` | the numbered index of a dump: number, user, URL |
| `status` | active/expired, expiry date, connections in use |
| `search [QUERY]` | channels whose name or category matches a case-insensitive regex |
| `check <ID\|QUERY>` | probe the stream with ffprobe — does it really play? |
| `url <ID\|QUERY>` | print the direct stream URL |
| `play <ID\|QUERY>` | open it in mpv; an id or a single match starts straight away |
| `counts [WHAT…]` | how many live channels / movies / series |
| `categories` | list categories |
| `save [NAME]` | save the current line as a profile |
| `quick` | status + categories + live count |

With a single line selected (`-n N`), every command behaves exactly as if you
had handed xtream that line's URL — picker, player and all.

Useful options: `-n` line numbers (`53`, `10-20`, `1,5,900`, `@file`), `-C`
category, `-m` must-have channels, `-z` timezone, `-a` show the failures too,
`-o` mirror the run to a file, `-j` how many lines at once (chosen from the size
of the dump unless you set it), `-T` per-line timeout, `-r` bypass the cache.

Every run prints how long it took.

Run `xtream --help` for the full list and options.

## Saved profiles

Lines you want to keep live in `~/.config/xtream/<name>.conf` (created with mode `600`):

```ini
XTREAM_HOST=http://line.example.com:8080
XTREAM_USER=myusername
XTREAM_PASS=mypassword
```

Save the current line with `xtream … save NAME`, then reach it with `xtream -p NAME`. Any value may be a **1Password** secret reference (`op://Vault/Item/field`), resolved at runtime with `op read`, so the password never sits in the file.

Channel lists are cached under `~/.cache/xtream/` for 6 hours (`-r` bypasses the cache).

---

## Credentials & privacy

- **No credentials live in this repository.** The script reads them at runtime from the URL you pass or from your local `~/.config/xtream/` files, which are never committed.
- Profile files are written with `600` permissions, and secrets can be kept in 1Password instead of on disk.
- The base64 codes shared on r/IPTV_ZONENEW contain **other people's line credentials**, posted publicly by third parties. This tool only reads them; what you do with them is your responsibility. Use it to check lines you're authorized to use.

## License

[MIT](LICENSE)
