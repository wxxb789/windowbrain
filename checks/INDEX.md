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
| `long-paths` | `paths-fs` | `high` | `true` | `checks/paths-fs/long-paths.md` | `Filename too long`, `ENAMETOOLONG`, `PathTooLongException`, failed git checkout / npm install |
| `refs-cow-dir-reparse-fallback` | `paths-fs` | `medium` | `false` | `checks/paths-fs/refs-cow-dir-reparse-fallback.md` | `os error 5` on ReFS worktree block-clone, slow checkout fallback with pnpm junctions or submodule dir symlinks |
| `projfs-vs-refs-cow` | `paths-fs` | `low` | `true` | `checks/paths-fs/projfs-vs-refs-cow.md` | Slow huge-repo agent sandboxes, `Client-ProjFS` / `prjflt` not enabled, ReFS vs ProjFS backend tradeoffs |
| `devdrive-same-volume-caches` | `perf-defender` | `medium` | `false` | `checks/perf-defender/devdrive-same-volume-caches.md` | Cross-volume cache copies on Dev Drive/ReFS, slow `uv`/`pnpm`/`bun`/`npm` installs, `npm warn Unknown config store-dir/cache-dir` |
| `devdrive-defender-policy-and-cpu-caps` | `perf-defender` | `high` | `false` | `checks/perf-defender/devdrive-defender-policy-and-cpu-caps.md` | 100% CPU / UI freeze on parallel builds, Defender `OnOpen` scan on Dev Drive, silent GPO override on `PerformanceModeStatus` |
| `orphan-processes-and-junction-rm` | `processes` | `high` | `false` | `checks/processes/orphan-processes-and-junction-rm.md` | `EBUSY` / `Permission denied` deleting worktrees or `pnpm` `node_modules` junctions, orphaned daemons holding locks |
| `wsl-nested-quoting-and-mnt-io` | `wsl-interop` | `high` | `false` | `checks/wsl-interop/wsl-nested-quoting-and-mnt-io.md` | `wsl.exe bash -lc` variable expansion hazards, slow `/mnt/c` builds, UTF-16LE `wsl -l` output |
| `wsl-default-distro-hijack` | `wsl-interop` | `medium` | `false` | `checks/wsl-interop/wsl-default-distro-hijack.md` | Bare `wsl` runs Docker Desktop / BusyBox `/bin/sh` instead of user's Linux dev distro |
| `mise-path-shim-drift` | `toolchain-mcp` | `high` | `false` | `checks/toolchain-mcp/mise-path-shim-drift.md` | `go tool version` mismatch, system `PATH` shadowing `mise` shims, cross-shell toolchain drift |
| `display-viewport-dpi-precondition` | `toolchain-mcp` | `low` | `false` | `checks/toolchain-mcp/display-viewport-dpi-precondition.md` | Desktop UI / WebView2 / Playwright tests fail with clamped viewport on portrait, high-DPI, or RDP display |
| `corporate-registry-lockfile-pollution` | `toolchain-mcp` | `medium` | `false` | `checks/toolchain-mcp/corporate-registry-lockfile-pollution.md` | `ERR_PNPM_TARBALL_URL_MISMATCH` in public CI, internal feed URLs / `sha1-` written into lockfiles on corporate Windows host |
