# xtream

A small terminal tool for checking IPTV lines and searching their channels. It answers two questions fast:

- **Is this line active?** — status, expiry, connection limits
- **Does it carry the channel I want, and does it actually play?** — search by name, then probe the stream with `ffprobe`

It speaks two protocols:

- **Xtream Codes** — the usual `get.php` / `player_api.php` API (username + password)
- **Stalker / Ministra portals** — MAC-based, no username/password

…and it can expand a whole **batch** of lines shared as a single base64 code, running any command against every line in parallel.

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
| `fzf` | the interactive picker in `play` |
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

### Test a line someone sent you

Paste a provider's URL directly — no setup. With no command it prints a quick verdict (status + categories + channel count):

```bash
xtream 'http://host:8080/get.php?username=USER&password=PASS&type=m3u_plus'
```

Then search and probe:

```bash
xtream 'http://host:8080/get.php?username=USER&password=PASS' search 'espn'
xtream 'http://host:8080/get.php?username=USER&password=PASS' check 'espn hd'
```

> Quote the URL — an unquoted `&` backgrounds the command.

### Test a whole batch (a base64 code from r/IPTV_ZONENEW)

Copy the "Xtream" text from a post and hand it in. It decodes, decrypts the paste, finds every line, and runs your command on each — in parallel:

```bash
xtream '<base64-code>'                 # one-line status of every line
xtream '<base64-code>' search 'cnn'    # search each active line
xtream '<base64-code>' counts          # channel counts per line
xtream '<base64-code>' --stats         # status + live/movie counts
```

Lines on a shared server hand out the same channel list, so identical results are folded to `(same as <first line>)` — which conveniently also flags any line whose package differs.

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
| `status` | active/expired, expiry date, connections in use |
| `search [QUERY]` | channel names matching a case-insensitive regex |
| `check <ID\|QUERY>` | probe the stream with ffprobe — does it really play? |
| `url <ID\|QUERY>` | print the direct stream URL |
| `play [QUERY]` | pick a channel (fzf) and open it in mpv |
| `counts` | how many live channels / movies / series |
| `categories` | list categories |
| `save [NAME]` | save the current line as a profile |
| `batch <code> [cmd]` | expand a base64/paste.sh dump and run cmd on each line |

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
