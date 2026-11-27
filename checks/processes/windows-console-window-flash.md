---
id: windows-console-window-flash
category: processes
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (python, node, rust, gui-session, ai-agents)
symptoms:
  - "Background agent workers, hooks, or test runners rapidly flash black cmd/console windows on the Windows desktop"
  - "Subprocesses launched with `DETACHED_PROCESS` still spawn visible console windows for every child console app"
  - "GUI desktop helper binaries built with PE Subsystem `WINDOWS_CUI (3)` flash a console window on launch"
---

# Background Subprocess Console Window Flash (`CREATE_NO_WINDOW` vs `DETACHED_PROCESS`)

## Failure Mode

On Windows GUI sessions, spawning console-subsystem child processes (`python`, `git`, `node`, `pwsh`, helper CLIs) from a GUI host, daemon, or background agent runner without `CREATE_NO_WINDOW` (`0x08000000`) allocates a visible console window that flashes and steals focus. Using `DETACHED_PROCESS` (`0x00000008`) instead of `CREATE_NO_WINDOW` is a classic trap: it detaches the immediate child from the parent's console, causing Windows to create a brand-new visible console window for every grandchild console binary it invokes. Similarly, Rust/native GUI helpers compiled with the default `WINDOWS_CUI` (subsystem `3`) PE header open a console window unless linked with `#![windows_subsystem = "windows"]` (`WINDOWS_GUI`, subsystem `2`).

## Detect (Read-Only)

```powershell
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
if ($LASTEXITCODE -ne 0 -or -not $repoRoot) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'Run inside a Git repository to audit Python subprocess spawn sites' }
    return
}

$pyFiles = @(& git -C $repoRoot ls-files '*.py' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ })
if ($pyFiles.Count -eq 0) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'No tracked Python files in repository' }
    return
}

$detachedHits = @($pyFiles | Select-String -Pattern 'DETACHED_PROCESS')
$spawnNoFlag = @($pyFiles | Select-String -Pattern 'subprocess\.(Popen|run|check_output|call)\(' |
    Where-Object { $_.Line -notmatch 'CREATE_NO_WINDOW|creationflags' })

[PSCustomObject]@{
    Status              = if ($detachedHits.Count -gt 0) { 'FAIL' } else { 'PASS' }
    DetachedProcessHits = $detachedHits.Count
    UnflaggedPySpawns   = $spawnNoFlag.Count
    SampleDetached      = @($detachedHits | Select-Object -First 5 Path, LineNumber, Line)
}
```

- **PASS**: `Status -eq 'PASS'` (`DetachedProcessHits -eq 0`; review `UnflaggedPySpawns` for GUI/daemon-invoked call sites).
- **FAIL**: `Status -eq 'FAIL'` (`DETACHED_PROCESS` used in tracked Python spawn sites).
- **SKIP**: `Status -eq 'SKIP'` (not inside a Git repository or no tracked `.py` files).

## Fix

In Python runners/hooks on Windows, pass `creationflags=subprocess.CREATE_NO_WINDOW` (never `DETACHED_PROCESS` for console children):

```python
import subprocess, sys

CREATE_NO_WINDOW = getattr(subprocess, "CREATE_NO_WINDOW", 0)
kwargs = {"creationflags": CREATE_NO_WINDOW} if sys.platform == "win32" else {}
subprocess.run(["git", "status"], **kwargs)
```

For PowerShell background launches, use `Start-Process -WindowStyle Hidden` and avoid `cmd /c start`. For Rust/C++ GUI helpers, set `#![windows_subsystem = "windows"]` (or `/SUBSYSTEM:WINDOWS`).

## Verify

```powershell
# Optional PE Subsystem check for a compiled GUI helper binary (2 = WINDOWS_GUI, 3 = WINDOWS_CUI):
function Get-PeSubsystem([string]$Path) {
    $b = [IO.File]::ReadAllBytes($Path)
    $pe = [BitConverter]::ToInt32($b, 0x3C)
    [BitConverter]::ToUInt16($b, $pe + 24 + 68)
}

$repoRoot = (& git rev-parse --show-toplevel 2>$null)
$pyFiles = if ($LASTEXITCODE -eq 0 -and $repoRoot) { @(& git -C $repoRoot ls-files '*.py' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ }) } else { @() }
$detached = if ($pyFiles.Count -gt 0) { @($pyFiles | Select-String -Pattern 'DETACHED_PROCESS') } else { @() }
[PSCustomObject]@{ RemainingDetachedProcessHits = $detached.Count }
```

- **PASS**: `RemainingDetachedProcessHits -eq 0`, and any compiled GUI helper returns `2` (`WINDOWS_GUI`) from `Get-PeSubsystem`.

## Rollback

Remove `CREATE_NO_WINDOW` only for explicitly interactive terminal commands that require a visible console window.

## Edge Cases

- Per Win32 `CreateProcess` documentation, `CREATE_NO_WINDOW` is **ignored** if combined with `DETACHED_PROCESS`; never OR them together.
- In Node.js/Bun, pass `windowsHide: true` to `child_process.spawn` / `execFile` (Node sets `CREATE_NO_WINDOW` automatically when `windowsHide: true` and `detached: false`).
