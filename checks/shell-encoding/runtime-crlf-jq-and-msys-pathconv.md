---
id: runtime-crlf-jq-and-msys-pathconv
category: shell-encoding
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (git-bash, msys2, jq, ai-agents)
symptoms:
  - "Bash scripts parsing JSON via native Windows `jq.exe` fail with subtle trailing `\\r` mismatches (e.g., invalid URLs, broken file paths, failed string comparisons) even when repo files are LF"
  - "Git for Windows `grep` silently strips trailing CR, hiding CRLF bugs until another tool consumes the string"
  - "Git Bash / MSYS2 silently rewrites Windows command flags like `/c`, `/grant`, or `/TN` into `C:/Program Files/Git/...` when agents run `cmd /c`, `icacls`, `schtasks`, or `reg` from Bash"
  - "Setting `MSYS_NO_PATHCONV=1` globally breaks shell scripts that rely on POSIX-to-Windows path translation"
---

# Runtime CRLF from Native CLIs (`jq -b`) & MSYS2 Argument Conversion

## Failure Mode

Two distinct Windows-shell boundary traps repeatedly break AI agent scripts:
1. **Runtime CRLF from native Windows CLIs**: Even when repository files are strictly LF, native Windows binaries like `jq.exe` emit `\r\n` (CRLF) to stdout by default. Bash command substitution `$(jq -r ...)` preserves the `\r`, corrupting variables, URLs, and paths unless `jq -b` (`--binary`) is used at the source.
2. **MSYS2 automatic argument conversion**: Git Bash converts any argument starting with `/` into a Windows path before passing it to native Windows executables (`icacls /grant`, `schtasks /TN`, `cmd /c`). Turning `MSYS_NO_PATHCONV=1` on *globally* breaks POSIX scripts that legitimately pass `/tmp/...` paths to Windows tools; instead, `MSYS_NO_PATHCONV=1` must be scoped per-command, or Windows admin CLIs (`icacls`, `takeown`, `schtasks`, `reg`, `cmd /c`) must be routed to PowerShell (`pwsh.exe`).

## Detect (Read-Only)

```powershell
$jq = Get-Command jq -ErrorAction SilentlyContinue
$jqEmitsCr = $false
if ($jq -and $jq.CommandType -eq 'Application' -and $jq.Source -match '\.exe$') {
    $psi = [Diagnostics.ProcessStartInfo]::new($jq.Source, '-r .a')
    $psi.RedirectStandardInput = $true
    $psi.RedirectStandardOutput = $true
    $psi.UseShellExecute = $false
    $p = [Diagnostics.Process]::Start($psi)
    $p.StandardInput.Write('{"a":"x"}')
    $p.StandardInput.Close()
    $raw = $p.StandardOutput.ReadToEnd()
    $p.WaitForExit()
    $jqEmitsCr = $raw.Contains("`r")
}

$globalNoPathconv = [Environment]::GetEnvironmentVariable('MSYS_NO_PATHCONV')
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
$shJqNoB = @()
if ($LASTEXITCODE -eq 0 -and $repoRoot) {
    $shFiles = @(& git -C $repoRoot ls-files '*.sh' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ })
    if ($shFiles.Count -gt 0) {
        $shJqNoB = @($shFiles | Select-String -Pattern '\bjq\s+(?![^\n]*(-b\b|--binary\b))' | Select-Object -First 10 Path, LineNumber, Line)
    }
}

$status = if ($globalNoPathconv -or ($jqEmitsCr -and $shJqNoB.Count -gt 0)) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status           = $status
    JqPath           = if ($jq) { $jq.Source } else { $null }
    JqEmitsCrlf      = $jqEmitsCr
    GlobalNoPathconv = $globalNoPathconv
    ShJqWithoutB     = $shJqNoB
}
```

- **PASS**: `Status -eq 'PASS'` (`GlobalNoPathconv` is unset AND no tracked `.sh` scripts call Windows `jq.exe` without `-b`).
- **FAIL**: `Status -eq 'FAIL'` (`MSYS_NO_PATHCONV` is set globally OR tracked `.sh` scripts call CRLF-emitting `jq.exe` without `-b`).
- **SKIP**: Not applicable.

## Fix

1. **Unset global `MSYS_NO_PATHCONV`** if present (scope it per-command instead, e.g., `MSYS_NO_PATHCONV=1 wsl.exe ...`, or route Windows-flag CLIs like `icacls`/`schtasks`/`reg`/`cmd /c` to `pwsh.exe`):

```powershell
[Environment]::SetEnvironmentVariable('MSYS_NO_PATHCONV', $null, 'User')
Remove-Item Env:MSYS_NO_PATHCONV -ErrorAction SilentlyContinue
```

2. **In Bash scripts running on Windows/Git Bash**, pass `-b` (`--binary`) to Windows `jq.exe` (or guard conditionally on Windows) so it outputs LF natively instead of sprinkling `| tr -d '\r'` downstream:

```bash
# On Windows jq.exe, -b forces LF output (check jq --help if targeting older non-Windows jq builds)
jq_bin() { jq -b "$@" 2>/dev/null || jq "$@"; }
val=$(jq_bin -r '.field' input.json)
```

## Verify

```powershell
$jq = Get-Command jq -ErrorAction SilentlyContinue
$crWithB = $false
if ($jq -and $jq.CommandType -eq 'Application' -and $jq.Source -match '\.exe$') {
    $psi = [Diagnostics.ProcessStartInfo]::new($jq.Source, '-b -r .a')
    $psi.RedirectStandardInput = $true
    $psi.RedirectStandardOutput = $true
    $psi.UseShellExecute = $false
    $p = [Diagnostics.Process]::Start($psi)
    $p.StandardInput.Write('{"a":"x"}')
    $p.StandardInput.Close()
    $raw = $p.StandardOutput.ReadToEnd()
    $p.WaitForExit()
    $crWithB = $raw.Contains("`r")
}
[PSCustomObject]@{
    GlobalNoPathconv = [Environment]::GetEnvironmentVariable('MSYS_NO_PATHCONV', 'User')
    JqCrWithBFlag    = $crWithB
}
```

- **PASS**: `GlobalNoPathconv` is empty AND `JqCrWithBFlag -eq $false`.

## Rollback

If an existing local workflow explicitly depended on `MSYS_NO_PATHCONV=1`, restore it via `[Environment]::SetEnvironmentVariable('MSYS_NO_PATHCONV', '1', 'User')`.

## Edge Cases

- Do **not** blindly set `git config --global core.safecrlf true` as a cure for runtime `jq.exe` CRLF: `core.safecrlf` only checks Git round-trip conversion and can cause `git add` to abort in repositories with CRLF `.gitattributes` rules.
- In PowerShell 7.4+ scripts, enable `$PSNativeCommandUseErrorActionPreference = $true` so non-zero exit codes from native CLIs fail fast instead of requiring manual `$LASTEXITCODE` checks after every command.
