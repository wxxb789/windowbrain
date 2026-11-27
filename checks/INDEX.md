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
| `long-paths` | `paths-fs` | `high` | `true` | `checks/paths-fs/long-paths.md` | `Filename too long`, `ENAMETOOLONG`, `PathTooLongException`, failed git checkout / npm install |
| `refs-cow-dir-reparse-fallback` | `paths-fs` | `medium` | `false` | `checks/paths-fs/refs-cow-dir-reparse-fallback.md` | `os error 5` on ReFS worktree block-clone, slow checkout fallback with pnpm junctions or submodule dir symlinks |
| `projfs-vs-refs-cow` | `paths-fs` | `low` | `true` | `checks/paths-fs/projfs-vs-refs-cow.md` | Slow huge-repo agent sandboxes, `Client-ProjFS` / `prjflt` not enabled, ReFS vs ProjFS backend tradeoffs |
| `devdrive-same-volume-caches` | `perf-defender` | `medium` | `false` | `checks/perf-defender/devdrive-same-volume-caches.md` | Cross-volume cache copies on Dev Drive/ReFS, slow `uv`/`pnpm`/`bun`/`npm` installs, `npm warn Unknown config store-dir/cache-dir` |
