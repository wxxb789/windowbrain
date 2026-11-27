---
id: system32-bash-shadows-git-bash
category: wsl-interop
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (git-bash, wsl2, codex, claude-code, ai-agents)
symptoms:
  - "Agent running `bash -lc ...` invokes `C:\\Windows\\System32\\bash.exe` (WSL) instead of Git Bash"
  - "`/c/Users/...: No such file or directory` because WSL expects `/mnt/c/...` while MSYS/Git Bash expects `/c/...`"
  - "Native Windows tools (`pwsh`, `git`, `node`, `uv`) return `exit 127` / `No such file or directory` inside `bash`"
---

# `System32\bash.exe` (WSL) Shadowing Git Bash on `PATH`

## Failure Mode

Windows ships a legacy WSL launcher at `$env:SystemRoot\System32\bash.exe`. Because `System32` precedes `Git\bin` / `Git\usr\bin` in default `PATH`, any AI coding agent or subcommand that resolves bare `bash` on Windows silently launches WSL instead of Git Bash. Inside WSL, MSYS-style paths (`/c/...`) fail with `No such file or directory`, Windows-installed CLIs are missing or require `.exe` suffixes, and WSL startup warnings pollute stdout/stderr.

## Detect (Read-Only)

```powershell
$resolved = Get-Command bash -ErrorAction SilentlyContinue
if (-not $resolved) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'bash not on PATH' }
    return
}

# Discover Git Bash dynamically via `git --exec-path` (covers Program Files, per-user, Scoop, portable)
$gitBash = $null
$gitExec = (& git --exec-path 2>$null | Out-String).Trim()
if ($gitExec) {
    $gitRoot = Split-Path -Parent (Split-Path -Parent (Split-Path -Parent $gitExec))
    $gitBash = @(
        (Join-Path $gitRoot 'bin\bash.exe'),
        (Join-Path $gitRoot 'usr\bin\bash.exe')
    ) | Where-Object { Test-Path $_ } | Select-Object -First 1
}
if (-not $gitBash) {
    $gitBash = @(
        (Join-Path $env:ProgramFiles 'Git\bin\bash.exe'),
        (Join-Path $env:LOCALAPPDATA 'Programs\Git\bin\bash.exe')
    ) | Where-Object { Test-Path $_ } | Select-Object -First 1
}

$isSystem32Wsl = ($resolved.Source -ieq (Join-Path $env:SystemRoot 'System32\bash.exe'))
$explicitBashEnv = [Environment]::GetEnvironmentVariable('CLAUDE_CODE_GIT_BASH_PATH', 'User')
if (-not $explicitBashEnv) { $explicitBashEnv = $env:CLAUDE_CODE_GIT_BASH_PATH }

$status = if ($isSystem32Wsl -and $gitBash -and -not ($explicitBashEnv -and (Test-Path $explicitBashEnv))) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status                 = $status
    ResolvedBash           = $resolved.Source
    IsSystem32Wsl          = $isSystem32Wsl
    GitBashPath            = $gitBash
    ExplicitGitBashEnvPath = $explicitBashEnv
}
```

- **PASS**: `Status -eq 'PASS'` (either `IsSystem32Wsl -eq $false` OR explicit agent Git Bash path `CLAUDE_CODE_GIT_BASH_PATH` points to an existing `bash.exe`).
- **FAIL**: `Status -eq 'FAIL'` (`IsSystem32Wsl -eq $true`, Git Bash is installed, and no explicit Git Bash path is configured).
- **SKIP**: `Status -eq 'SKIP'` (`bash` is not on `PATH`).

## Fix

Configure the explicit Git Bash path persistently (works without Admin rights, because `Machine PATH` always precedes `User PATH` on Windows so User `PATH` alone cannot beat `System32\bash.exe`):

```powershell
$gitExec = (& git --exec-path 2>$null | Out-String).Trim()
$gitRoot = if ($gitExec) { Split-Path -Parent (Split-Path -Parent (Split-Path -Parent $gitExec)) } else { Join-Path $env:ProgramFiles 'Git' }
$gitBash = @(
    (Join-Path $gitRoot 'bin\bash.exe'),
    (Join-Path $gitRoot 'usr\bin\bash.exe')
) | Where-Object { Test-Path $_ } | Select-Object -First 1

if (-not $gitBash) { throw 'Git Bash not found' }

[Environment]::SetEnvironmentVariable('CLAUDE_CODE_GIT_BASH_PATH', $gitBash, 'User')
$env:CLAUDE_CODE_GIT_BASH_PATH = $gitBash
$env:PATH = "$(Split-Path -Parent $gitBash);$env:PATH"
```

## Verify

```powershell
$uname = (& $env:CLAUDE_CODE_GIT_BASH_PATH -c 'uname -s' 2>&1 | Out-String).Trim()
[PSCustomObject]@{
    ExplicitGitBash = [Environment]::GetEnvironmentVariable('CLAUDE_CODE_GIT_BASH_PATH', 'User')
    Uname           = $uname
}
```

- **PASS**: `ExplicitGitBash` points to `...\Git\bin\bash.exe` AND `Uname -match '^MINGW'` (not `Linux`).

## Rollback

```powershell
[Environment]::SetEnvironmentVariable('CLAUDE_CODE_GIT_BASH_PATH', $null, 'User')
Remove-Item Env:CLAUDE_CODE_GIT_BASH_PATH -ErrorAction SilentlyContinue
```

## Edge Cases

- On Windows, `Machine` `PATH` is always evaluated before `User` `PATH`. Adding `Git\bin` only to `User` `PATH` will **not** beat `C:\Windows\System32\bash.exe`; use explicit agent shell configuration (`CLAUDE_CODE_GIT_BASH_PATH` or your harness's shell binary setting).
- If you *intentionally* want WSL Linux bash, invoke `wsl.exe -d <DistroName> -- bash` explicitly instead of relying on deprecated `System32\bash.exe`.
