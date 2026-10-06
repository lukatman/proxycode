# ProxyCode

Route Codex, Claude Code or any other command through a WireGuard tunnel without
changing your system's network settings. ProxyCode runs
[WireProxy](https://github.com/windtf/wireproxy) as an authenticated local HTTP
proxy and manages named tunnel profiles.

```text
proxycode claude
  → start and check the default profile's tunnel if needed
  → launch Claude Code with proxy variables set
```

## Install

You need a WireGuard configuration. Run the interactive installer without `sudo`:

```bash
curl -fL "https://github.com/lukatman/proxycode/releases/download/v0.1.0/install.sh" \
  -o install.sh && bash install.sh
```

Downloads are pinned and checksum-verified. If `proxycode` isn't found, add
`export PATH="$HOME/.local/bin:$PATH"` to `~/.bashrc` and open a new terminal.

<details>
<summary><strong>Requirements</strong></summary>

- Linux x86-64 or ARM64, Bash 4.4 or newer, readable `/proc`, and writable user/XDG
  directories. CI runs on Ubuntu 24.04 for both architectures.
- `curl` with HTTPS support and CA certificates, `flock` from util-linux, `tar`,
  GNU coreutils, `awk`, `grep`, and `sed`. Missing commands are reported by the
  installer; install them with your distribution's package manager.
- Loopback networking, access to GitHub over HTTPS, and outbound connectivity to
  your WireGuard endpoint and health-check URL.

There is no system VPN interface, service or automatic startup. macOS, Windows,
WSL and containers are not supported targets.

</details>

<details>
<summary><strong>Other installation options</strong></summary>

The installer downloads a fixed ProxyCode bundle and verifies its release
checksum, then downloads WireProxy **1.1.3** and checks the SHA-256 pinned in the
installer. Keep `install.sh` for maintenance, or delete it after setup. The
installer does not edit shell configuration.

For flag-based setup, use the downloaded installer with your configuration path
and a profile name:

```bash
bash install.sh \
  --wg-config "$HOME/Downloads/tunnel.conf" --name work --default
proxycode start
```

Flag-based installation does not start the tunnel; `start` does that and checks
HTTPS egress. To install first and import a profile later:

```bash
bash install.sh --install-only
proxycode profile import "$HOME/Downloads/tunnel.conf" --name work --default
```

For a versioned piped installation:

```bash
curl --proto '=https' --tlsv1.2 -fL \
  https://github.com/lukatman/proxycode/releases/download/v0.1.0/install.sh |
  bash -s -- --wg-config "$HOME/Downloads/tunnel.conf" --name work --default
```

To inspect the full fixed release before running it, download and unpack it in a
new directory:

```bash
mkdir proxycode-inspect && cd proxycode-inspect
release=https://github.com/lukatman/proxycode/releases/download/v0.1.0
curl --proto '=https' --tlsv1.2 -fLO "$release/proxycode-0.1.0.tar.gz"
curl --proto '=https' --tlsv1.2 -fLO "$release/proxycode-0.1.0.tar.gz.sha256"
sha256sum -c proxycode-0.1.0.tar.gz.sha256 && tar -xzf proxycode-0.1.0.tar.gz
cd proxycode-0.1.0
less install.sh bin/proxycode lib/proxycode.sh
bash install.sh --wg-config "$HOME/Downloads/tunnel.conf" --name work --default
```

An optional `--wireproxy-bin /absolute/path/to/wireproxy` installs a binary you
provide. It checks compatibility, but does not verify that binary against the
pinned release checksum.

</details>

## Usage

```bash
proxycode                         # Management menu
proxycode claude                  # Run Claude Code through the default profile
proxycode codex
proxycode --profile work curl https://example.com
proxycode start [NAME]            # Start a profile's tunnel
proxycode stop
proxycode switch NAME             # Stop the active tunnel and start another
proxycode status                  # Local process state
proxycode check                   # HTTPS check through the active tunnel
proxycode profile import FILE --name NAME [--default]
proxycode profile list
proxycode profile default NAME
proxycode profile remove NAME
```

The default profile is used when you don't name one. Only one profile can be
active at a time, and a wrapped command won't switch it silently. Exiting a
wrapped command leaves the tunnel running until `proxycode stop`.

## VS Code extension

To route Claude Code in the VS Code extension through ProxyCode, open the
Remote-SSH settings JSON file and add this line, using your home directory:

```jsonc
"claudeCode.claudeProcessWrapper": "/home/USER/.local/bin/proxycode"
```

## Limitations

- Only applications that honor `HTTP_PROXY`, `HTTPS_PROXY` and `ALL_PROXY` are
  routed. This is not a kill switch.
- Codex stdio MCP servers and Claude background agents may bypass the tunnel.
- Shared background services, such as the Codex app-server daemon, keep the
  environment they started with. Restart them under `proxycode`.

Don't publish profile files, proxy variables or logs.

<details>
<summary><strong>Settings and health checks</strong></summary>

The listener defaults to `127.0.0.1:25345`. Stop the tunnel before changing its port:

```bash
proxycode stop
proxycode settings --http-port 25346
proxycode profile settings work --probe cloudflare --expect-location US
```

Cloudflare is the default probe; without an expected country it checks egress
without requiring a location. Mullvad checks that the exit belongs to Mullvad.
A custom probe checks an HTTPS response's status and optional literal text:

```bash
proxycode profile settings work --probe mullvad --expect-location Sweden
proxycode profile settings work --probe custom \
  --url https://example.com/health --status 200 --contains ready
```

Run `proxycode help` for the full command syntax and `bash install.sh --help` for
installer options.

</details>

<details>
<summary><strong>Reinstall or remove</strong></summary>

To repair an installation or install a newer version, stop the tunnel and rerun
that version's installer. Profiles, credentials and settings are preserved:

```bash
proxycode stop
bash install.sh --install-only
```

If you use your own WireProxy binary, pass `--wireproxy-bin` again; otherwise the
installer installs its pinned binary. An older installer refuses to downgrade an
installation recorded as newer.

```bash
bash install.sh --uninstall   # Keep profiles, credentials and settings
bash install.sh --purge       # Remove all managed data
```

Both operations stop the managed tunnel safely and ask for confirmation. Add
`--yes` for unattended use. Neither deletes your original imported configuration.
After uninstall, rerunning `--install-only` makes the preserved profiles usable.

</details>

<details>
<summary><strong>Files and troubleshooting</strong></summary>

Default locations are below; `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, and
`XDG_STATE_HOME` override the corresponding roots.

| Location | Contents |
| --- | --- |
| `~/.local/bin/proxycode` | Installed command |
| `~/.config/proxycode/` | Listener settings and default profile |
| `~/.local/share/proxycode/` | WireProxy, its ISC notice, library and private profiles |
| `~/.local/state/proxycode/` | Installation metadata, active process state and profile logs |

Managed directories and executables are owner-only (`0700`); private files are
`0600`. Do not publish profile files, proxy environment values or logs. Logs are
under `logs/NAME/wireproxy.log` in the state directory. A lifecycle command rotates
a log at 10 MiB, keeping one `.old` copy; this is not a continuous size limit.

| Problem | What to do |
| --- | --- |
| `proxycode` not found | Add `~/.local/bin` to PATH. |
| No default selected | Run `proxycode profile default NAME`, or supply a name. |
| Port already in use | Stop its owner or choose a free port with `proxycode settings`. |
| Startup health check fails | Check the source configuration, endpoint connectivity, probe and expected location. Inspect the private log, then retry `start`. |
| `check` fails | The tunnel remains running. Inspect the log and network, then stop/start if needed. |
| `HTTP CONNECT` fails with `401` | The proxy rejected the credentials. Check for an older proxy or background service using different credentials on the same port. |
| Process identity is ambiguous | Do not kill the recorded PID blindly. Verify which process owns the listener. Only after confirming no managed WireProxy remains, remove the `active` file at the path reported by `status` and retry. |
| Missing or damaged installed files | Rerun the fixed-version installer after stopping the tunnel. |

</details>

See [Verification](docs/verification.md) for contributor checks.
