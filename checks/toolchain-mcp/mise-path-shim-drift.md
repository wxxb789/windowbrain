---
id: mise-path-shim-drift
category: toolchain-mcp
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (mise / winget / scoop / system installers)
symptoms:
  - "compile: version \"go1.x.y\" does not match go tool version \"go1.x.z\""
  - Agent invokes wrong node / python / go / bun version despite mise.toml or .tool-versions
  - Tool works in interactive PowerShell but fails with command not found or wrong version in Git Bash / agent subshell
  - "mise ERROR No version is set for shim: <tool> when agents run outside a directory with mise.toml"
  - "mise WARN failed to write cache file: ...\\exec_env_*.msgpack.z Access is denied. (os error 5) during concurrent agent runs"
---

# `mise` / System `PATH` Ordering & Cross-Shell Toolchain Drift

## Failure Mode

On Windows, system installers (`C:\Program Files\Go\bin`, `C:\Program Files\nodejs`, `C:\ProgramData\...`) frequently sit ahead of user-level version manager shims (`%LOCALAPPDATA%\mise\shims`, `%USERPROFILE%\.local\bin`) in `PATH`, or non-interactive agent shells skip PowerShell/Bash profile activation (`mise activate`). Meanwhile, `mise` or project configs set env vars like `GOROOT` to the pinned version, causing fatal version mismatches (e.g., `compile: version "go1.27.3" does not match go tool version "go1.27.4"`).

## Detect (Read-Only)

```powershell
$hasMise = [bool](Get-Command mise -ErrorAction SilentlyContinue)
$miseShims = Join-Path $env:LOCALAPPDATA 'mise\shims'
$pathEntries = ($env:PATH -split ';') | Where-Object { $_ }
$shimsIdx = [Array]::IndexOf($pathEntries, $miseShims)

$conflicts = @()
foreach ($tool in @('go', 'node', 'python', 'bun', 'cargo', 'rg')) {
  $all = @(Get-Command $tool -All -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Source)
  if ($all.Count -gt 1) {
    $conflicts += [PSCustomObject]@{ Tool = $tool; First = $all[0]; All = ($all -join ' | ') }
  }
}

[PSCustomObject]@{
  HasMise            = $hasMise
  MiseShimsExists    = (Test-Path $miseShims)
  MiseShimsPathIndex = $shimsIdx
  HasLocalMiseToml   = (Test-Path 'mise.toml')
  MultiInstallTools  = $conflicts
}
```

- **SKIP**: `HasMise -eq $false` AND `MultiInstallTools.Count -eq 0`.
- **PASS**:
  - If `HasMise -eq $true`: `MiseShimsExists -eq $true` AND `MiseShimsPathIndex -ge 0` AND no system install (`C:\Program Files\...`) shadows `mise\shims` for tools managed by `mise`.
- **FAIL**: `mise` is installed but `mise\shims` is missing from `PATH` or shadowed by a system install earlier in `PATH`.

## Fix

1. **Agent execution rule (zero system mutation, immediate fix for pinned repos)**:
   Whenever a repository contains `mise.toml` or `.tool-versions`, invoke project commands through `mise exec -- <cmd>` (or `mise exec <tool>@<ver> -- <cmd>`) instead of hand-editing `PATH` or calling bare `go`/`node`/`bun`.

2. **Ensure `%LOCALAPPDATA%\mise\shims` is prepended to User `PATH`** (if `HasMise -eq $true` and missing/shadowed):
```powershell
$miseShims = Join-Path $env:LOCALAPPDATA 'mise\shims'
$userPath = [System.Environment]::GetEnvironmentVariable('Path', 'User')
$parts = @($userPath -split ';' | Where-Object { $_ -and $_ -ne $miseShims })
$newUserPath = (@($miseShims) + $parts) -join ';'
[System.Environment]::SetEnvironmentVariable('Path', $newUserPath, 'User')
```
Note: on Windows, Machine `PATH` is always evaluated BEFORE User `PATH`. If a system installer in Machine `PATH` (e.g., `C:\Program Files\Go\bin`) shadows a `mise`-managed tool, either use `mise exec -- <cmd>` (recommended) or remove the duplicate system installer.

## Verify

- Re-run `Detect (Read-Only)` and confirm `MiseShimsPathIndex -ge 0`.
- In a repo with `mise.toml`, verify `mise exec -- <tool> --version` matches the pinned version in `mise.toml`.

## Rollback

Remove `%LOCALAPPDATA%\mise\shims` from User `Path` if added in Step 2.

## Edge Cases

- Because Windows concatenates `Machine PATH + User PATH`, modifying User `PATH` alone CANNOT override a binary in `C:\Program Files\...` (Machine `PATH`). Using `mise exec -- <cmd>` or `mise activate` dynamically prepends the pinned install directory to the process `PATH`, cleanly overriding Machine `PATH` without requiring Admin rights.
- In Git Bash, `PATH` entries use POSIX conversion; prefer `mise exec --` for identical behavior across PowerShell, CMD, and Git Bash.
- **Unpinned global shims**: When agents invoke tools from temp dirs or worktrees without a local `mise.toml`, shims fail with `mise ERROR No version is set for shim: <tool>`. Pin global defaults with `mise use -g <tool>@<version>`.
- **Concurrent `mise` cache lock (`os error 5`)**: Parallel agent invocations on Windows can collide when atomically replacing `exec_env_*.msgpack.z` under `MISE_CACHE_DIR`. Isolate `MISE_CACHE_DIR` per worker or resolve the binary once with `mise which <tool>` before fanning out child processes.
