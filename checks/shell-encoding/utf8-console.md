---
id: utf8-console
category: shell-encoding
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11
symptoms:
  - UnicodeEncodeError: 'gbk' or 'charmap' codec can't encode character
  - Mojibake / garbled non-ASCII output in agent tool logs
  - Git shows octal-escaped paths like \344\270\255 instead of UTF-8 filenames
---

# UTF-8 User Environment, PowerShell & Git Encoding

## Failure Mode

Windows defaults legacy codepages (e.g., CP936/CP1252) for Python stdio, PowerShell pipe encoding, and Git path quoting. AI agents reading stdout/stderr or running Python/Git tools over pipes hit `UnicodeEncodeError`, corrupted diffs, or octal-escaped filenames.

## Detect (Read-Only)

```powershell
$pyUtf8 = [System.Environment]::GetEnvironmentVariable('PYTHONUTF8', 'User')
$hasGit = [bool](Get-Command git -ErrorAction SilentlyContinue)
$gitQuotePath = if ($hasGit) { (git config --global --get core.quotepath 2>$null) } else { 'NO_GIT' }
$gitLogEnc    = if ($hasGit) { (git config --global --get i18n.logoutputencoding 2>$null) } else { 'NO_GIT' }
$gitCommitEnc = if ($hasGit) { (git config --global --get i18n.commitencoding 2>$null) } else { 'NO_GIT' }
$psEnc = $OutputEncoding.WebName

[PSCustomObject]@{
  PYTHONUTF8_User      = $pyUtf8
  Git_CoreQuotePath    = $gitQuotePath
  Git_LogEncoding      = $gitLogEnc
  Git_CommitEncoding   = $gitCommitEnc
  PS_OutputEncoding    = $psEnc
  PS_ProfilePath       = $PROFILE.CurrentUserAllHosts
  PS_ProfileHasUtf8    = (Test-Path $PROFILE.CurrentUserAllHosts) -and (Select-String -Path $PROFILE.CurrentUserAllHosts -Pattern 'windowbrain:utf8' -Quiet)
}
```

- **PASS**:
  - `PYTHONUTF8_User -eq '1'`
  - If `git` is installed: `Git_CoreQuotePath -eq 'false'`, `Git_LogEncoding -eq 'utf-8'`, `Git_CommitEncoding -eq 'utf-8'`
  - `PS_OutputEncoding -eq 'utf-8'` OR `PS_ProfileHasUtf8 -eq $true`
- **FAIL**: any of the above conditions is not met.

## Fix

Apply only the missing items:

```powershell
# 1. Enable UTF-8 mode for Python (User scope, no admin required)
[System.Environment]::SetEnvironmentVariable('PYTHONUTF8', '1', 'User')
$env:PYTHONUTF8 = '1'

# 2. Configure Git UTF-8 output and unescaped paths (if git is installed)
if (Get-Command git -ErrorAction SilentlyContinue) {
  git config --global core.quotepath false
  git config --global i18n.logoutputencoding utf-8
  git config --global i18n.commitencoding utf-8
}

# 3. Ensure PowerShell CurrentUserAllHosts profile sets UTF-8 console/pipe encoding
$profilePath = $PROFILE.CurrentUserAllHosts
$profileDir = Split-Path -Parent $profilePath
if (-not (Test-Path $profileDir)) { New-Item -ItemType Directory -Path $profileDir -Force | Out-Null }
if (-not (Test-Path $profilePath)) { New-Item -ItemType File -Path $profilePath -Force | Out-Null }
if (-not (Select-String -Path $profilePath -Pattern 'windowbrain:utf8' -Quiet)) {
  $block = @"

# >>> windowbrain:utf8 >>>
`$utf8NoBom = New-Object System.Text.UTF8Encoding(`$false)
[Console]::InputEncoding  = `$utf8NoBom
[Console]::OutputEncoding = `$utf8NoBom
`$OutputEncoding           = `$utf8NoBom
# <<< windowbrain:utf8 <<<
"@
  Add-Content -Path $profilePath -Value $block -Encoding UTF8
}
```

## Verify

Re-run the `Detect (Read-Only)` command block.
- **PASS**: `PYTHONUTF8_User` is `'1'`, Git keys match expected values (if Git exists), and `PS_ProfileHasUtf8` is `$true`.

## Rollback

```powershell
# 1. Remove PYTHONUTF8 user env var
[System.Environment]::SetEnvironmentVariable('PYTHONUTF8', $null, 'User')

# 2. Unset Git encoding keys (if previously unset)
if (Get-Command git -ErrorAction SilentlyContinue) {
  git config --global --unset core.quotepath
  git config --global --unset i18n.logoutputencoding
  git config --global --unset i18n.commitencoding
}

# 3. Remove windowbrain:utf8 block from PowerShell profile
$profilePath = $PROFILE.CurrentUserAllHosts
if (Test-Path $profilePath) {
  $content = Get-Content -Path $profilePath -Raw
  $updated = [regex]::Replace($content, '(?ms)\r?\n?# >>> windowbrain:utf8 >>>.*?# <<< windowbrain:utf8 <<<\r?\n?', "`r`n")
  Set-Content -Path $profilePath -Value $updated.TrimEnd() -Encoding UTF8
}
```

## Edge Cases

- New `PYTHONUTF8` user environment variable takes effect in newly launched terminals/agents; current process needs `$env:PYTHONUTF8 = '1'`.
- PowerShell 5.1 and PowerShell 7+ have separate `CurrentUserAllHosts` profile paths (`WindowsPowerShell\profile.ps1` vs `PowerShell\profile.ps1`). Run the profile snippet in whichever PowerShell runtime(s) the user's agents invoke.
- System-wide Beta UTF-8 (`HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage` `ACP=65001`) is intentionally NOT toggled here because it requires a reboot and can break legacy non-Unicode Windows apps.
