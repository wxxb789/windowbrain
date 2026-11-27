---
id: winget-inplace-upgrade-corrupts-mise-shims
category: toolchain-mcp
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (winget, mise, node, ai-agents)
symptoms:
  - "`The term 'node.exe' is not recognized` when running `npx` / `npm` even though `mise` shows Node installed"
  - "WinGet / UniGetUI upgrade of `mise` fails mid-flight with `Access is denied` while `mise.exe` is in use, deleting `mise-shim.exe` first"
  - "`mise` silently falls back from `exe` shims to `file` shims (`node.cmd` only, no `node.exe`), breaking PowerShell wrappers like `npx.ps1` that explicitly invoke `node.exe`"
---

# WinGet In-Place Upgrade Deleting `mise-shim.exe` & Breaking `node.exe`

## Failure Mode

When `mise` is installed via WinGet (especially with auto-updaters like UniGetUI or scheduled `winget upgrade --all`), WinGet upgrades `mise` by deleting the old package directory in place. The deletion is **not atomic**: companion files such as `mise-shim.exe` are deleted first, then deletion fails with `Access is denied` because an active agent shell or daemon holds `mise.exe` open. Left in this half-broken state, `mise` silently falls back to `file` shim mode (generating `node.cmd` instead of `node.exe`), breaking `npx.ps1` and any tool that explicitly invokes `node.exe` / `python.exe`.

## Detect (Read-Only)

```powershell
$miseCmd = Get-Command mise -ErrorAction SilentlyContinue
if (-not $miseCmd) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'mise not installed' }
    return
}

# Resolve symlink/reparse target (e.g. WinGet\Links\mise.exe -> WinGet\Packages\jdx.mise_...\mise\bin\mise.exe)
$item = Get-Item -LiteralPath $miseCmd.Source -ErrorAction SilentlyContinue
$realExe = if ($item.Target) {
    $t = if ($item.Target -is [array]) { $item.Target[0] } else { $item.Target }
    if ([IO.Path]::IsPathRooted($t)) { $t } else { Join-Path (Split-Path -Parent $miseCmd.Source) $t }
} else {
    $miseCmd.Source
}

$miseDir = Split-Path -Parent $realExe
$shimExeOk = Test-Path (Join-Path $miseDir 'mise-shim.exe')
$isWinGetManaged = ($miseCmd.Source -match '(?i)\\WinGet\\') -or ($realExe -match '(?i)\\WinGet\\')

$shimsDir = if ($env:MISE_DATA_DIR) { Join-Path $env:MISE_DATA_DIR 'shims' } else { Join-Path $env:LOCALAPPDATA 'mise\shims' }
$hasNodeCmdOnly = (Test-Path (Join-Path $shimsDir 'node.cmd')) -and (-not (Test-Path (Join-Path $shimsDir 'node.exe')))

$wingetPinned = $false
if ($isWinGetManaged -and (Get-Command winget -ErrorAction SilentlyContinue)) {
    $wingetPinned = ((& winget pin list 2>$null | Out-String) -match 'jdx\.mise')
}

$status = if ((-not $shimExeOk) -or $hasNodeCmdOnly -or ($isWinGetManaged -and -not $wingetPinned)) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status          = $status
    MiseSource      = $miseCmd.Source
    ResolvedExe     = $realExe
    IsWinGetManaged = $isWinGetManaged
    MiseShimExeOk   = $shimExeOk
    HasNodeCmdOnly  = $hasNodeCmdOnly
    WingetPinned    = $wingetPinned
}
```

- **PASS**: `Status -eq 'PASS'` (`MiseShimExeOk -eq $true`, `HasNodeCmdOnly -eq $false`, and if `IsWinGetManaged -eq $true` then `WingetPinned -eq $true`).
- **FAIL**: `Status -eq 'FAIL'` (`MiseShimExeOk -eq $false` OR `HasNodeCmdOnly -eq $true` OR (`IsWinGetManaged -eq $true` AND `WingetPinned -eq $false`)).
- **SKIP**: `Status -eq 'SKIP'` (`mise` is not installed).

## Fix

1. If `IsWinGetManaged -eq $true`, pin `jdx.mise` in WinGet to block unattended in-place upgrades:

```powershell
winget pin add --id jdx.mise --blocking
```

2. If `MiseShimExeOk -eq $false` or `HasNodeCmdOnly -eq $true`, close active shells/processes holding `mise.exe`, repair `mise`, re-pin, and rebuild shims:

```powershell
winget pin remove --id jdx.mise
winget install --id jdx.mise --force
winget pin add --id jdx.mise --blocking
mise reshim --force
```

## Verify

```powershell
$npxVer = & npx --version 2>&1 | Out-String
$pinned = if (Get-Command winget -ErrorAction SilentlyContinue) { ((& winget pin list 2>$null | Out-String) -match 'jdx\.mise') } else { $true }
[PSCustomObject]@{
    NpxExit      = $LASTEXITCODE
    NpxVersion   = $npxVer.Trim()
    WingetPinned = $pinned
}
```

- **PASS**: `NpxExit -eq 0`, `WingetPinned -eq $true`, and re-running `Detect (Read-Only)` reports `Status -eq 'PASS'`.

## Rollback

```powershell
winget pin remove --id jdx.mise
```

## Edge Cases

- Pinning `jdx.mise` in WinGet only blocks unattended upgrades of the `mise` binary itself; `mise upgrade` for managed runtimes (Node, Python, Go, Bun) continues working normally.
- See also `windows-winget-autopatch-daemon-kill` for pinning long-running agent daemons against unattended WinGet upgrades.
