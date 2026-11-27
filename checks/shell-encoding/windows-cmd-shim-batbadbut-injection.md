---
id: windows-cmd-shim-batbadbut-injection
category: shell-encoding
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (python, node, rust, mcp, ai-agents)
symptoms:
  - "Spawning `.cmd`/`.bat` shims (`npx.cmd`, `uvx.cmd`, `npm.cmd`, `pnpm.cmd`) with untrusted or prompt-derived arguments executes arbitrary commands via `&`, `|`, `>`, or `%VAR%` expansion (BatBadBut: CVE-2024-24576 / CVE-2024-27980 / CVE-2024-1874)"
  - "Node.js 18.20.2+ / 20.12.2+ / 21.7.3+ throws `EINVAL` when `spawn()` targets a `.cmd`/`.bat` file with `shell: false`"
  - "Arguments containing quotes, trailing backslashes, or `%ENV_VAR%` are silently corrupted or break out of quoting when forwarded through `%*`"
---

# Windows `.cmd`/`.bat` Shim Command Injection (BatBadBut)

## Failure Mode

On Windows, many CLI tools (`npm`, `npx`, `pnpm`, `yarn`, `claude`, `codex`, and Python/Node shims) are installed as `.cmd` or `.bat` batch wrappers. Calling `CreateProcess` (even with `shell=False` / argv list form) on a `.cmd`/`.bat` file implicitly invokes `cmd.exe /c`, which parses metacharacters (`&`, `|`, `<`, `>`, `^`, `%VAR%`) before handing arguments to the target binary (known as **BatBadBut**: CVE-2024-24576 in Rust, CVE-2024-27980 in Node.js, CVE-2024-1874 in PHP). Even when every argument is quoted with MSVCRT rules, an argument containing `%VAR%` where the environment variable's value holds a `"` plus `&` breaks out of quotes during `cmd.exe` expansion and executes injected commands.

See also `pwsh-cmd-wrapper-arg-mangling` and `python-subprocess-shell-false-ignores-pathext`.

## Detect (Read-Only)

```powershell
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
if ($LASTEXITCODE -ne 0 -or -not $repoRoot) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'Run inside a Git repository to audit .cmd/shell=True spawn sites' }
    return
}

$files = @(& git -C $repoRoot ls-files '*.py' '*.ts' '*.js' '*.mjs' '*.cjs' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ })
if ($files.Count -eq 0) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'No tracked Python/JS/TS files in repository' }
    return
}

$riskySpawns = @($files | Select-String -Pattern 'shell\s*[:=]\s*[Tt]rue|spawn(?:Sync)?\([^)]*\.(cmd|bat)' | Select-Object -First 15 Path, LineNumber, Line)

[PSCustomObject]@{
    Status          = if ($riskySpawns.Count -gt 0) { 'FAIL' } else { 'PASS' }
    RiskySpawnCount = $riskySpawns.Count
    RiskySpawns     = $riskySpawns
}
```

- **PASS**: `Status -eq 'PASS'` (`RiskySpawnCount -eq 0`).
- **FAIL**: `Status -eq 'FAIL'` (`RiskySpawnCount -gt 0` — callers use `shell=True` or spawn `.cmd`/`.bat` wrappers directly).
- **SKIP**: `Status -eq 'SKIP'` (not inside a Git repository or no tracked Python/JS/TS files).

## Fix

1. **Preferred (eliminates `cmd.exe` entirely)**: Bypass `.cmd` shims in MCP configs and agent wrappers by invoking the real binary directly:
   - Instead of `npx.cmd -y <pkg>`, invoke `node.exe` directly on the installed package entry script (or use a native `.exe` launcher like `bunx.exe` / `uvx.exe`).
   - In Python/Node subprocess wrappers, invoke `[sys.executable, "-m", ...]` or `["node", entry_js, ...]`.
2. **Defense-in-depth when `.cmd` is unavoidable**: Reject `\r`, `\n`, and unescaped `%` in arguments, and pass dynamic/LLM-generated payloads via `stdin` or a temp file instead of positional CLI arguments.

## Verify

```powershell
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
$files = if ($LASTEXITCODE -eq 0 -and $repoRoot) { @(& git -C $repoRoot ls-files '*.py' '*.ts' '*.js' '*.mjs' '*.cjs' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ }) } else { @() }
$hits = if ($files.Count -gt 0) { @($files | Select-String -Pattern 'shell\s*[:=]\s*[Tt]rue|spawn(?:Sync)?\([^)]*\.(cmd|bat)') } else { @() }
[PSCustomObject]@{ RemainingRiskySpawns = $hits.Count }
```

- **PASS**: `RemainingRiskySpawns -eq 0`, and any remaining `.cmd` wrapper rejects or neutralizes `%VAR%` / `&` / `\n` payloads in an isolated child-process unit test.

## Rollback

Revert wrapper changes via `git checkout -- <file>` if a specific tool requires batch-script environment setup; keep untrusted payload passing over `stdin`/files.

## Edge Cases

- On Node.js >= 18.20.2 / 20.12.2 / 21.7.3+, `child_process.spawn('foo.cmd', args, { shell: false })` throws `EINVAL` by design as a BatBadBut mitigation; blindly setting `shell: true` re-opens command injection. Resolve the underlying `.exe` or pass arguments safely via `stdin`/file.
