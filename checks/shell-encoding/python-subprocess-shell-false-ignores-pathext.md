---
id: python-subprocess-shell-false-ignores-pathext
category: shell-encoding
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (python, mcp, ai-agents)
symptoms:
  - "`FileNotFoundError: [WinError 2] The system cannot find the file specified` when Python calls `subprocess.run(['tool', ...], shell=False)` even though `tool` works in PowerShell/cmd"
  - "Agent plugins or MCP health checks silently report `is_available() == False` on Windows because the CLI is installed as `tool.cmd` / `tool.bat` via npm or pip"
---

# Python `subprocess(..., shell=False)` Ignoring `PATHEXT` for `.cmd` Shims

## Failure Mode

On POSIX, `subprocess.run(["mycli", ...], shell=False)` resolves `mycli` on `PATH` directly. On Windows, `CreateProcess` (used by `subprocess` when `shell=False`) only auto-appends `.exe` — it **ignores `PATHEXT`** (`.CMD`, `.BAT`, `.COM`). Any Python agent wrapper, MCP server, or skill script that calls `subprocess.run(["npx", ...])` or `subprocess.run(["mycli", ...])` without first resolving via `shutil.which()` raises `FileNotFoundError` (`WinError 2`) whenever the target is a `.cmd`/`.bat` shim, silently disabling plugins or failing tool calls.

## Detect (Read-Only)

```powershell
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
if ($LASTEXITCODE -ne 0 -or -not $repoRoot) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'Run inside a Git repository to audit Python subprocess calls' }
    return
}

$pyFiles = @(& git -C $repoRoot ls-files '*.py' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ })
if ($pyFiles.Count -eq 0) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'No tracked Python files in repository' }
    return
}

$hits = @($pyFiles | Select-String -Pattern 'subprocess\.(run|Popen|check_output|call)\(\s*\[\s*["''](npm|npx|pnpm|yarn|az|claude|codex)["'']')

[PSCustomObject]@{
    Status                      = if ($hits.Count -gt 0) { 'FAIL' } else { 'PASS' }
    UnresolvedCmdSubprocessHits = $hits.Count
    Matches                     = @($hits | Select-Object -First 10 Path, LineNumber, Line)
}
```

- **PASS**: `Status -eq 'PASS'` (`UnresolvedCmdSubprocessHits -eq 0` — callers resolve executables via `shutil.which()` or invoke the underlying `.exe` directly).
- **FAIL**: `Status -eq 'FAIL'` (`UnresolvedCmdSubprocessHits -gt 0`).
- **SKIP**: `Status -eq 'SKIP'` (not inside a Git repository or no tracked `.py` files).

## Fix

Resolve the executable with `shutil.which()` (which honors `PATHEXT` on Windows) before spawning, and follow `windows-cmd-shim-batbadbut-injection` whenever the resolved target is a `.cmd`/`.bat` shim:

```python
import shutil, subprocess

exe = shutil.which("mycli")
if not exe:
    raise FileNotFoundError("mycli not found on PATH")
subprocess.run([exe, "--version"], check=True)
```

## Verify

```powershell
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
$pyFiles = if ($LASTEXITCODE -eq 0 -and $repoRoot) { @(& git -C $repoRoot ls-files '*.py' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ }) } else { @() }
$hits = if ($pyFiles.Count -gt 0) { @($pyFiles | Select-String -Pattern 'subprocess\.(run|Popen|check_output|call)\(\s*\[\s*["''](npm|npx|pnpm|yarn|az|claude|codex)["'']') } else { @() }
[PSCustomObject]@{ RemainingUnresolvedHits = $hits.Count }
```

- **PASS**: `RemainingUnresolvedHits -eq 0` and unit tests confirm `shutil.which()` resolves `.cmd` shims without raising `WinError 2`.

## Rollback

No system state changed; revert code changes with `git checkout -- <file>` if needed.

## Edge Cases

- Do **not** "fix" `WinError 2` by flipping `shell=True` blindly — that introduces shell command injection (see `windows-cmd-shim-batbadbut-injection`). Resolve with `shutil.which()`, or better yet invoke the underlying `.exe` (`node.exe <entry.js>` / `sys.executable -m <pkg>`).
