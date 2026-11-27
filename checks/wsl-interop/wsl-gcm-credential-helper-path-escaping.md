---
id: wsl-gcm-credential-helper-path-escaping
category: wsl-interop
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (wsl2, git, gcm, ai-agents)
symptoms:
  - "Git operations inside WSL fail to launch Windows Git Credential Manager (GCM) or hang waiting for interactive username/password"
  - "`git credential fill` fails when `credential.helper` points to `/mnt/c/Program Files/Git/...` without backslash-escaping the space"
  - "Git for Windows moved GCM from `mingw64/bin` to `ucrt64/bin` on newer installs, breaking hardcoded `mingw64` helper paths"
---

# WSL2 Git Credential Manager (`ucrt64`) Path & Space Escaping

## Failure Mode

To share Git HTTPS authentication between Windows and WSL2 without duplicating tokens or hanging headless agents on `Username for 'https://...':`, WSL Git is commonly configured to invoke the Windows `git-credential-manager.exe` via the WSL mount path. Two Windows-specific pitfalls break this silently:
1. Git splits `credential.helper` on unescaped whitespace, so `/mnt/c/Program Files/Git/...` fails unless spaces are backslash-escaped (`/mnt/c/Program\ Files/Git/...`).
2. Newer Git for Windows installations place `git-credential-manager.exe` under `ucrt64\bin\` rather than the legacy `mingw64\bin\` path, leaving hardcoded WSL `.gitconfig` paths broken after a Git upgrade.

## Detect (Read-Only)

```powershell
if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'wsl.exe not installed' }
    return
}

$winGcmCandidates = @(
    (Join-Path $env:ProgramFiles 'Git\ucrt64\bin\git-credential-manager.exe'),
    (Join-Path $env:ProgramFiles 'Git\mingw64\bin\git-credential-manager.exe'),
    (Join-Path $env:LOCALAPPDATA 'Programs\Git\ucrt64\bin\git-credential-manager.exe'),
    (Join-Path $env:LOCALAPPDATA 'Programs\Git\mingw64\bin\git-credential-manager.exe')
)
$actualWinGcm = $winGcmCandidates | Where-Object { Test-Path $_ } | Select-Object -First 1
if (-not $actualWinGcm) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'Windows git-credential-manager.exe not found' }
    return
}

$job = Start-Job -ScriptBlock {
    & wsl.exe -e sh -c 'h=$(git config --get-all credential.helper 2>/dev/null | tail -n 1); printf "HELPER=%s\n" "$h"; p=$(printf "%s" "$h" | sed "s/\\\\ / /g"); [ -x "$p" ] && echo "EXEC=YES" || echo "EXEC=NO"' 2>&1 | Out-String
}
if (-not (Wait-Job -Job $job -Timeout 15)) {
    Stop-Job -Job $job -ErrorAction SilentlyContinue
    Remove-Job -Job $job -Force -ErrorAction SilentlyContinue
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'wsl.exe timed out (check wslservice-control-plane-deadlock-pre-2-9-3)' }
    return
}
$out = Receive-Job -Job $job
Remove-Job -Job $job -Force -ErrorAction SilentlyContinue

$wslHelper = if ($out -match '(?m)^HELPER=(.*)$') { $Matches[1].Trim() } else { '' }
$helperExecOk = ($out -match '(?m)^EXEC=YES$')
$unescapedSpace = ($wslHelper -match 'Program Files' -and $wslHelper -notmatch 'Program\\ Files')
$pointsToWinGcm = ($wslHelper -match 'git-credential-manager\.exe')

$status = if ($pointsToWinGcm -and ($unescapedSpace -or -not $helperExecOk)) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status            = $status
    WindowsGcmPath    = $actualWinGcm
    WslHelperConfig   = $wslHelper
    HasUnescapedSpace = $unescapedSpace
    HelperExecutable  = $helperExecOk
}
```

- **PASS**: `Status -eq 'PASS'` (WSL `credential.helper` either uses a non-Windows helper like `gh auth git-credential`, or points to an executable Windows GCM with escaped spaces).
- **FAIL**: `Status -eq 'FAIL'` (WSL `credential.helper` targets `git-credential-manager.exe` but has unescaped spaces or points to a non-existent path such as legacy `mingw64`).
- **SKIP**: `Status -eq 'SKIP'` (`wsl.exe` or Windows GCM is not installed, or WSL timed out).

## Fix

Back up existing WSL `credential.helper` values, resolve the live Windows GCM path dynamically via `wslpath -u` (honoring custom automount roots), escape spaces, and apply with `--replace-all`:

```powershell
$ErrorActionPreference = 'Stop'
$gcm = @(
    (Join-Path $env:ProgramFiles 'Git\ucrt64\bin\git-credential-manager.exe'),
    (Join-Path $env:ProgramFiles 'Git\mingw64\bin\git-credential-manager.exe'),
    (Join-Path $env:LOCALAPPDATA 'Programs\Git\ucrt64\bin\git-credential-manager.exe'),
    (Join-Path $env:LOCALAPPDATA 'Programs\Git\mingw64\bin\git-credential-manager.exe')
) | Where-Object { Test-Path $_ } | Select-Object -First 1
if (-not $gcm) { throw 'git-credential-manager.exe not found' }

# Back up prior WSL credential.helper values
$prior = (& wsl.exe -e git config --global --get-all credential.helper 2>$null) -join "`n"
$bakFile = Join-Path $env:TEMP 'windowbrain-wsl-gcm-prior.txt'
[IO.File]::WriteAllText($bakFile, $prior, [Text.UTF8Encoding]::new($false))

$wslRaw = (& wsl.exe -e wslpath -u $gcm 2>$null | Out-String).Trim()
if (-not $wslRaw) { throw "wslpath -u failed for $gcm" }
$wslEscaped = $wslRaw -replace ' ', '\ '

& wsl.exe -e git config --global --replace-all credential.helper $wslEscaped
```

## Verify

```powershell
$ver = (& wsl.exe -e sh -c 'h=$(git config --global --get-all credential.helper | tail -n 1); p=$(printf "%s" "$h" | sed "s/\\\\ / /g"); "$p" --version' 2>&1 | Out-String).Trim()
[PSCustomObject]@{ ExitCode = $LASTEXITCODE; GcmVersion = $ver }
```

- **PASS**: `ExitCode -eq 0` AND `GcmVersion` prints a non-empty version string with no interactive prompt.

## Rollback

```powershell
$bakFile = Join-Path $env:TEMP 'windowbrain-wsl-gcm-prior.txt'
& wsl.exe -e git config --global --unset-all credential.helper 2>$null
if (Test-Path $bakFile) {
    Get-Content -Path $bakFile | Where-Object { $_ -ne '' } | ForEach-Object {
        & wsl.exe -e git config --global --add credential.helper $_
    }
}
```

## Edge Cases

- When testing with `GIT_TRACE=1 git credential fill`, note that Git appends `get` to the helper command; to probe the version without triggering an interactive prompt, invoke the resolved binary directly with `--version` (or set `GIT_TERMINAL_PROMPT=0`).
