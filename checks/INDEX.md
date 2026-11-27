# Checks Index

Machine-readable registry of all `windowbrain` checks.

## Categories

| Category | Directory | Scope |
|---|---|---|
| `shell-encoding` | `checks/shell-encoding/` | PowerShell/Bash/CMD quoting, UTF-8/GBK/codepage, CRLF/LF, ConPTY/TUI |
| `paths-fs` | `checks/paths-fs/` | `MAX_PATH`, slashes/drives, symlinks, `EBUSY`/file locks, case sensitivity |
| `perf-defender` | `checks/perf-defender/` | Windows Defender real-time scan overhead, Dev Drive, CPU contention |
| `processes` | `checks/processes/` | Orphan child processes, lingering ports, zombie MCP/node/python, signal handling |
| `wsl-interop` | `checks/wsl-interop/` | `wsl.exe` quoting, `/mnt/c` I/O latency, proxy/localhost forwarding, clipboard/auth |
| `toolchain-mcp` | `checks/toolchain-mcp/` | `npx`/`uvx` shims, cross-shell `PATH` drift, UAC boundaries, corporate CA/SSL |

## Registered Checks

| ID | Category | Severity | Requires Admin | Path | Key Symptoms |
|---|---|---|---|---|---|
| `utf8-console` | `shell-encoding` | `high` | `false` | `checks/shell-encoding/utf8-console.md` | Mojibake, `UnicodeEncodeError`, escaped non-ASCII git paths, garbled tool output |
| `pwsh-get-content-tail-spin` | `shell-encoding` | `high` | `false` | `checks/shell-encoding/pwsh-get-content-tail-spin.md` | `pwsh` pins CPU indefinitely on `Get-Content -Tail` over NDJSON/long-line logs |
| `windows-cmd-shim-batbadbut-injection` | `shell-encoding` | `high` | `false` | `checks/shell-encoding/windows-cmd-shim-batbadbut-injection.md` | `.cmd`/`.bat` shim command injection (CVE-2024-1874 / `%VAR%` quote breakout), Node 20+ `EINVAL` |
| `runtime-crlf-jq-and-msys-pathconv` | `shell-encoding` | `high` | `false` | `checks/shell-encoding/runtime-crlf-jq-and-msys-pathconv.md` | Native `jq.exe` CRLF breaks Bash variables (`jq -b`), MSYS2 rewrites `/c` & `/grant` flags |
| `pwsh-cmd-wrapper-arg-mangling` | `shell-encoding` | `high` | `false` | `checks/shell-encoding/pwsh-cmd-wrapper-arg-mangling.md` | `pwsh` -> `.cmd` (`az`, `npm`) truncates at newlines, splits `&` in URLs, mangles quotes |
| `python-subprocess-shell-false-ignores-pathext` | `shell-encoding` | `high` | `false` | `checks/shell-encoding/python-subprocess-shell-false-ignores-pathext.md` | Python `subprocess.run(['tool'], shell=False)` throws `WinError 2` on `.cmd` shims |
| `pwsh-single-element-array-flattening` | `shell-encoding` | `medium` | `false` | `checks/shell-encoding/pwsh-single-element-array-flattening.md` | `@(@('old','new'))` flattens to 2 strings causing char-level corruption; unwrapped multi-line paste |
| `long-paths` | `paths-fs` | `high` | `true` | `checks/paths-fs/long-paths.md` | `Filename too long`, `ENAMETOOLONG`, `PathTooLongException`, failed git checkout / npm install |
| `windows-sandbox-env-leak-appdata` | `paths-fs` | `high` | `false` | `checks/paths-fs/windows-sandbox-env-leak-appdata.md` | Sandbox `HOME` override leaks into real `APPDATA`/`LOCALAPPDATA` junctions and `HKCU:\Environment\Path` |
| `refs-cow-dir-reparse-fallback` | `paths-fs` | `medium` | `false` | `checks/paths-fs/refs-cow-dir-reparse-fallback.md` | `os error 5` on ReFS worktree block-clone, slow checkout fallback with pnpm junctions or submodule dir symlinks |
| `projfs-vs-refs-cow` | `paths-fs` | `low` | `true` | `checks/paths-fs/projfs-vs-refs-cow.md` | Slow huge-repo agent sandboxes, `Client-ProjFS` / `prjflt` not enabled, ReFS vs ProjFS backend tradeoffs |
| `devdrive-same-volume-caches` | `perf-defender` | `medium` | `false` | `checks/perf-defender/devdrive-same-volume-caches.md` | Cross-volume cache copies on Dev Drive/ReFS, slow `uv`/`pnpm`/`bun`/`npm` installs, `npm warn Unknown config store-dir/cache-dir` |
| `devdrive-defender-policy-and-cpu-caps` | `perf-defender` | `high` | `false` | `checks/perf-defender/devdrive-defender-policy-and-cpu-caps.md` | 100% CPU / UI freeze on parallel builds, Defender `OnOpen` scan on Dev Drive, `MsSense`/DLP CPU overhead |
| `orphan-processes-and-junction-rm` | `processes` | `high` | `false` | `checks/processes/orphan-processes-and-junction-rm.md` | `EBUSY` / `Permission denied` deleting worktrees or `pnpm` junctions, PEB CWD locks, Named Pipe recovery race, SQLite `database is locked` |
| `windows-console-window-flash` | `processes` | `medium` | `false` | `checks/processes/windows-console-window-flash.md` | Background agent subprocesses flash cmd windows (`CREATE_NO_WINDOW` vs `DETACHED_PROCESS`, PE `WINDOWS_CUI`) |
| `windows-winget-autopatch-daemon-kill` | `processes` | `low` | `false` | `checks/processes/windows-winget-autopatch-daemon-kill.md` | Unattended `winget` / endpoint patching hard-kills running agent daemons without relaunch |
| `wsl-nested-quoting-and-mnt-io` | `wsl-interop` | `high` | `false` | `checks/wsl-interop/wsl-nested-quoting-and-mnt-io.md` | `wsl.exe bash -lc` variable expansion hazards, slow `/mnt/c` builds, UTF-16LE `wsl -l` output |
| `wsl-mirrored-gsa-dns-blackhole` | `wsl-interop` | `high` | `false` | `checks/wsl-interop/wsl-mirrored-gsa-dns-blackhole.md` | `networkingMode=mirrored` + SASE/VPN disables DNS Tunneling causing WSL DNS timeouts |
| `system32-bash-shadows-git-bash` | `wsl-interop` | `high` | `false` | `checks/wsl-interop/system32-bash-shadows-git-bash.md` | `System32\bash.exe` (WSL) shadows Git Bash on `PATH`, causing `/c/...` `ENOENT` and CLI `exit 127` |
| `wslservice-control-plane-deadlock-pre-2-9-3` | `wsl-interop` | `high` | `true` | `checks/wsl-interop/wslservice-control-plane-deadlock-pre-2-9-3.md` | `wsl -l` / `--status` / `--shutdown` hang on `LpcReply` (`WslService < 2.9.3` lifecycle deadlock) |
| `wsl-default-distro-hijack` | `wsl-interop` | `medium` | `false` | `checks/wsl-interop/wsl-default-distro-hijack.md` | Bare `wsl` runs Docker Desktop / BusyBox `/bin/sh` instead of user's Linux dev distro |
| `wsl-gcm-credential-helper-path-escaping` | `wsl-interop` | `medium` | `false` | `checks/wsl-interop/wsl-gcm-credential-helper-path-escaping.md` | WSL Git fails to launch Windows GCM due to `ucrt64` path move or unescaped space in `Program Files` |
| `mise-path-shim-drift` | `toolchain-mcp` | `high` | `false` | `checks/toolchain-mcp/mise-path-shim-drift.md` | `go tool version` mismatch, system `PATH` shadowing `mise` shims, unpinned shims / cache lock `os error 5` |
| `winget-inplace-upgrade-corrupts-mise-shims` | `toolchain-mcp` | `high` | `false` | `checks/toolchain-mcp/winget-inplace-upgrade-corrupts-mise-shims.md` | WinGet in-place upgrade deletes `mise-shim.exe` then fails `Access is denied`, breaking `node.exe` for `npx.ps1` |
| `local-agent-proxy-public-firewall-exposure` | `toolchain-mcp` | `high` | `true` | `checks/toolchain-mcp/local-agent-proxy-public-firewall-exposure.md` | Local AI proxy/MCP server binds `0.0.0.0`/`::` while Windows Firewall allows `bun`/`node`/`python`/`deno` inbound |
| `interactive-shell-path-bleed-and-mise-cache-lock` | `toolchain-mcp` | `medium` | `false` | `checks/toolchain-mcp/interactive-shell-path-bleed-and-mise-cache-lock.md` | Unguarded `mise activate` in PowerShell profiles explodes child shell `PATH` to 100-200+ entries |
| `windows-tls-schannel-and-openssl-handshake` | `toolchain-mcp` | `medium` | `false` | `checks/toolchain-mcp/windows-tls-schannel-and-openssl-handshake.md` | `curl (35) schannel` vs Node/Python OpenSSL CA store split (`NODE_EXTRA_CA_CERTS`, `http.sslBackend schannel`) |
| `gcm-empty-user-binding-popup-storm` | `toolchain-mcp` | `medium` | `false` | `checks/toolchain-mcp/gcm-empty-user-binding-popup-storm.md` | Empty `credential.<url>.username=` causes background `git fetch` to trigger recurring GCM account popups |
| `scoop-shim-broken-by-self-update` | `toolchain-mcp` | `medium` | `false` | `checks/toolchain-mcp/scoop-shim-broken-by-self-update.md` | `topgrade`/`mise self-update` moves Scoop-managed `mise.exe`, breaking `current\bin` shims |
| `corporate-registry-lockfile-pollution` | `toolchain-mcp` | `medium` | `false` | `checks/toolchain-mcp/corporate-registry-lockfile-pollution.md` | `ERR_PNPM_TARBALL_URL_MISMATCH` in public CI, internal feed URLs / `sha1-` written into lockfiles on corporate Windows host |
| `display-viewport-dpi-precondition` | `toolchain-mcp` | `low` | `false` | `checks/toolchain-mcp/display-viewport-dpi-precondition.md` | Desktop UI / WebView2 / Playwright tests fail with clamped viewport on portrait, high-DPI, or RDP display |
