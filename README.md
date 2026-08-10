# intentd-releases

Public distribution channel for **intentd** — the Intent daemon, a local-first,
headless backend that runs coding agents and serves the Intent desktop and mobile
apps over JSON-RPC 2.0 (Unix socket locally, WSS/TLS over the LAN). Release
artifacts are mirrored here from the `intent-hq/intentd` source repository so
intentd can be installed and auto-updated without access to it.

> **Using the Intent desktop app?** You don't need anything from this repo — desktop
> releases in
> [intent-hq/cloudlands-releases](https://github.com/intent-hq/cloudlands-releases)
> bundle intentd as a sidecar and are self-contained. Install standalone intentd only
> for a **headless** machine (a remote Linux box, a spare Mac) that the desktop and
> mobile apps connect to remotely — see
> [Remote access](#remote-access--pair-a-client) below.

## Install

Every method below installs the intentd **sitter** — a small self-updating supervisor
packaged and named `intentd`. On the first `intentd serve` it downloads the actual
daemon for your release channel (**stable** by default), checks for updates at startup
and then every 12–24 hours, and respawns the daemon if it crashes. You never install
the daemon binary directly.

Supported platforms: macOS (Apple Silicon + Intel), Linux (x64 + ARM64, static musl
builds), Windows (x64).

### macOS / Linux — one-line installer

```sh
curl -fsSL https://github.com/intent-hq/intentd-releases/releases/download/sitter-latest/install.sh | sh
```

Detects OS and architecture, downloads the matching archive from the fixed
[`sitter-latest`](https://github.com/intent-hq/intentd-releases/releases/tag/sitter-latest)
release, verifies its `.sha256` checksum, and installs `intentd` to `/usr/local/bin`
when writable, else `~/.local/bin` (override with `INTENTD_INSTALL_DIR`). Re-running
updates in place.

The script then offers to register intentd as a per-user service that starts at login
(systemd user unit on Linux, launchd LaunchAgent on macOS — running
`intentd serve --resume-all`) and starts it immediately. Set
`INTENTD_INSTALL_SERVICE=1` to set it up without prompting, `INTENTD_INSTALL_SERVICE=0`
to skip.

> **Headless Linux:** systemd user services only run while you are logged in unless
> lingering is enabled. Run `sudo loginctl enable-linger $USER` once so the intentd
> service starts at boot and survives SSH logout.

### macOS / Linux — Homebrew

```sh
brew install intent-hq/tap/intentd
# Run as a login service (launchd/systemd) — executes `intentd serve --resume-all`:
brew services start intentd
```

The formula lives in [intent-hq/homebrew-tap](https://github.com/intent-hq/homebrew-tap)
and is updated automatically by every sitter release.

### Debian / Ubuntu — .deb

```sh
curl -fLO https://github.com/intent-hq/intentd-releases/releases/download/sitter-latest/intentd_amd64.deb   # or intentd_arm64.deb
sudo apt install ./intentd_amd64.deb
# The package does not auto-enable the unit (it is per-user); start it with:
systemctl --user enable --now intentd
```

Installs the sitter at `/usr/bin/intentd` and a systemd **user** unit at
`/usr/lib/systemd/user/intentd.service`. The headless-Linux lingering note above
applies here too.

### Windows — one-line installer

```powershell
powershell -c "irm https://github.com/intent-hq/intentd-releases/releases/download/sitter-latest/install.ps1 | iex"
```

Installs `intentd.exe` to `%LOCALAPPDATA%\intentd\bin` (override with
`INTENTD_INSTALL_DIR`), adds that directory to the user `PATH`, and offers to register
a per-user Scheduled Task that runs `intentd serve --resume-all` at logon. Set
`$env:INTENTD_INSTALL_SERVICE = '1'` to register without prompting, `'0'` to skip.

### Direct download

Download the archive for your platform from the fixed
[`sitter-latest`](https://github.com/intent-hq/intentd-releases/releases/tag/sitter-latest)
release — `intentd-<target>.tar.xz` on macOS/Linux, `intentd-<target>.zip` on
Windows, each with a `.sha256` sidecar — extract it, and put `intentd` on your
`PATH`. Then run `intentd serve` (or wire up your own service around it).

```sh
curl -fLO https://github.com/intent-hq/intentd-releases/releases/download/sitter-latest/intentd-aarch64-apple-darwin.tar.xz
tar -xJf intentd-aarch64-apple-darwin.tar.xz
# → intentd-aarch64-apple-darwin/intentd
```

## Host requirements

- **git** — required. Workspace provisioning and daemon-side fetch/pull/push shell
  out to the `git` CLI.
- **Node.js** (with `npm`/`npx`) — required to run the coding-agent provider CLIs:
  several providers are npm-installed or launched via pinned `npx` packages
  (auggie, claude-code, codex, …).
- **gh** (GitHub CLI) — optional. Enables the GitHub integration without a manual
  token: the daemon resolves its GitHub token as secrets store (the in-app GitHub
  connection) → `GITHUB_TOKEN`/`GH_TOKEN` env → `gh auth token`.

## Remote access — pair a client

The daemon always listens on a local socket (Unix socket; named pipe on Windows). To
let the Intent desktop or mobile app connect from another machine, enable the secure
WSS/TLS listener and pair:

```sh
intentd pair                  # QR code + `intent://pair?…` URI in the terminal
intentd pair --png pair.png   # also export the QR code as an image
```

The payload embeds everything a client needs: the machine's LAN IP(s), the WSS port
(`server.wsApi.port`, default **5181**), the TLS certificate fingerprint (clients pin
it), and the bearer token. Scan the QR code with the Intent iOS app, or use the URI
in the desktop app's remote-connection flow. `intentd token` prints the same
credentials in plaintext.

- On intentd **v0.6.3+**, if the WSS listener is not running, `pair` offers to enable
  it on the spot — it persists `server.wsApi.enabled = true` and starts the listener
  immediately, no restart needed. Unattended runs must pass `--yes`. On older
  versions, enable it in the config file first and restart the daemon.
- The setting lives in `<data-dir>/config.toml` (macOS:
  `~/Library/Application Support/intentd/config.toml`, Linux:
  `~/.local/share/intentd/config.toml`):

  ```toml
  [server.wsApi]
  enabled = true
  port = 5181
  ```

- Make sure the port is reachable from your clients (open TCP 5181 in the machine's
  firewall for your LAN or tailnet). There is no plaintext listener — remote traffic
  is always WSS with TLS and bearer-token auth.

## Channels — stable vs beta

The sitter follows the **stable** channel by default. Switch a machine durably with
the sitter-owned `intentd sitter channel` command:

```sh
intentd sitter channel        # print the effective channel and its origin
intentd sitter channel beta   # pin beta in <data-dir>/sitter/config.toml
intentd sitter channel beta --redownload && intentd restart     # switch and activate now
intentd sitter channel stable --redownload && intentd restart   # explicit downgrade back to stable
```

Automatic update checks are strictly newer-only; `--redownload` is the explicit
downgrade path (e.g. beta → stable). Per-launch overrides take precedence over the
pin: `intentd --sitter-channel beta serve`, or set `INTENTD_CHANNEL=beta`.

## What's in this repository

No source code lives here. This repository hosts release artifacts mirrored from the
`intent-hq/intentd` source repository:

- **`sitter-vX.Y.Z` releases and the fixed `sitter-latest` release** — sitter
  archives (`intentd-<target>.tar.xz` / `.zip`), Debian packages
  (versioned `intentd_<version>_<arch>.deb` plus constant-named
  `intentd_amd64.deb` / `intentd_arm64.deb` copies on `sitter-latest`), the
  `install.sh` / `install.ps1` scripts, all with `.sha256` sidecars.
  `sitter-latest` always tracks the newest sitter — do not consume the tag itself;
  download its assets.
- **`vX.Y.Z` releases** — daemon platform archives the sitter downloads
  (`intentd-<target>.tar.xz` / `.zip`) and their `.sha256` sidecars.
- **`channel-beta` / `channel-stable` releases** — machine-readable `beta.json` /
  `stable.json` manifests pointing at the latest daemon release per channel.
  Download the manifest asset; do not consume the tags themselves.
