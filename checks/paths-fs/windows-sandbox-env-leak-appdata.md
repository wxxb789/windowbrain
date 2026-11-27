---
id: windows-sandbox-env-leak-appdata
category: paths-fs
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (installers, pytest, ai-agents, dotnet)
symptoms:
  - "Test or installer run with `HOME` / `USERPROFILE` overridden to a temp dir still overwrites live files or junctions in `$env:LOCALAPPDATA` or `$env:APPDATA`"
  - "Deleted temp sandbox leaves dangling directory junctions inside real user agent/skill directories"
  - "Test runs permanently pollute `HKCU:\\Environment\\Path` (e.g., `.NET` global tools appending 20+ stale entries to User `PATH`)"
---

# Sandbox `HOME` Override Leaking into Real `APPDATA`/`LOCALAPPDATA` & User `PATH`

## Failure Mode

On POSIX systems, overriding `HOME` isolates most CLI tools and installers. On Windows, overriding `HOME` and `USERPROFILE` is **not** sufficient: Windows tools, Node/Python installers, and agent harnesses read `$env:APPDATA`, `$env:LOCALAPPDATA`, `$env:XDG_CONFIG_HOME`, `$env:XDG_DATA_HOME`, `$env:XDG_STATE_HOME`, and tool-specific vars (`CODEX_HOME`, `CLAUDE_CONFIG_DIR`, etc.), which continue pointing at the real user profile. Worse, some toolchains (such as `.NET` SDK / global tools unless `DOTNET_ADD_GLOBAL_TOOLS_TO_PATH=false` is set) write directly to the persistent Windows Registry (`HKCU:\Environment\Path`), leaking temp/test directories into the host's permanent User `PATH`.

## Detect (Read-Only)

```powershell
# 1. Read raw (unexpanded REG_EXPAND_SZ) User PATH and check for duplicates or stale temp entries
$rawUserPath = (Get-Item 'HKCU:\Environment').GetValue('Path', '', 'DoNotExpandEnvironmentNames')
$entries = @($rawUserPath -split ';' | Where-Object { $_ -ne '' })
$dupes = @($entries | Group-Object { $_.TrimEnd('\').ToLowerInvariant() } | Where-Object Count -gt 1 | Select-Object -ExpandProperty Name)
$staleTemp = @($entries | Where-Object {
    $expanded = [Environment]::ExpandEnvironmentVariables($_)
    ($expanded -match '(?i)\\Temp\\|\\tmp\\') -and (-not (Test-Path -LiteralPath $expanded))
})

# 2. Check for dangling junctions/symlinks inside common agent config dirs
$agentDirs = @(
    (Join-Path $env:USERPROFILE '.agents'),
    (Join-Path $env:USERPROFILE '.claude'),
    (Join-Path $env:LOCALAPPDATA 'Programs')
) | Where-Object { Test-Path $_ }

$dangling = @(foreach ($d in $agentDirs) {
    Get-ChildItem -Path $d -Recurse -Depth 3 -ErrorAction SilentlyContinue |
        Where-Object { ($_.Attributes -band [IO.FileAttributes]::ReparsePoint) -and -not (Test-Path -LiteralPath $_.FullName) } |
        Select-Object -ExpandProperty FullName
})

$status = if ($staleTemp.Count -gt 0 -or $dupes.Count -gt 0 -or $dangling.Count -gt 0) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status             = $status
    UserPathCount      = $entries.Count
    StaleTempEntries   = $staleTemp
    DuplicateEntries   = $dupes
    DanglingLinks      = $dangling
}
```

- **PASS**: `Status -eq 'PASS'` (`StaleTempEntries.Count -eq 0`, `DuplicateEntries.Count -eq 0`, and `DanglingLinks.Count -eq 0`).
- **FAIL**: `Status -eq 'FAIL'` (stale temp or duplicate entries exist in `HKCU:\Environment\Path`, or dangling reparse points exist in live agent directories).
- **SKIP**: Not applicable.

## Fix

1. **Back up raw `HKCU:\Environment\Path` (preserving `REG_EXPAND_SZ` tokens like `%USERPROFILE%`) and prune only duplicates + non-existent temp entries**:

```powershell
$ErrorActionPreference = 'Stop'
$rawPath = (Get-Item 'HKCU:\Environment').GetValue('Path', '', 'DoNotExpandEnvironmentNames')
$backup = Join-Path $env:TEMP ('user-path-backup-' + (Get-Date -Format 'yyyyMMdd-HHmmss') + '.txt')
[IO.File]::WriteAllText($backup, $rawPath, [Text.UTF8Encoding]::new($false))
if (-not (Test-Path $backup)) { throw "Backup write failed: $backup" }

$seen = [System.Collections.Generic.HashSet[string]]::new([System.StringComparer]::OrdinalIgnoreCase)
$clean = @($rawPath -split ';' | Where-Object {
    if ($_ -eq '') { return $false }
    $expanded = [Environment]::ExpandEnvironmentVariables($_)
    $isStaleTemp = ($expanded -match '(?i)\\Temp\\|\\tmp\\') -and (-not (Test-Path -LiteralPath $expanded))
    (-not $isStaleTemp) -and $seen.Add($_.TrimEnd('\'))
})

Set-ItemProperty -Path 'HKCU:\Environment' -Name 'Path' -Value ($clean -join ';') -Type ExpandString
Write-Output "Cleaned HKCU:\Environment\Path (backup: $backup). Open a new shell to pick up changes."
```

2. **Remove confirmed dangling junctions** reported in `DanglingLinks` (replace `$LinkPath` with each confirmed dangling path; `cmd /c rmdir` removes the reparse point without touching targets):

```powershell
$LinkPath = '<confirmed-dangling-link-path>'
if ($LinkPath -ne '<confirmed-dangling-link-path>' -and (Test-Path -LiteralPath $LinkPath -ErrorAction SilentlyContinue) -eq $false) {
    cmd.exe /c "rmdir `"$LinkPath`""
}
```

3. **Sandbox test harness template (do NOT run in your active remediation shell — copy into isolated child test runners only)**:

```text
# Run inside a child process (e.g., pwsh -NoProfile -Command ...) so the parent shell env is untouched:
$sandbox = Join-Path ([IO.Path]::GetTempPath()) ("sandbox-" + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $sandbox -Force | Out-Null
$env:HOME = $sandbox
$env:USERPROFILE = $sandbox
$env:APPDATA = Join-Path $sandbox 'AppData\Roaming'
$env:LOCALAPPDATA = Join-Path $sandbox 'AppData\Local'
$env:XDG_CONFIG_HOME = Join-Path $sandbox '.config'
$env:XDG_DATA_HOME = Join-Path $sandbox '.local\share'
$env:XDG_STATE_HOME = Join-Path $sandbox '.local\state'
$env:DOTNET_ADD_GLOBAL_TOOLS_TO_PATH = 'false'
```

## Verify

```powershell
$raw = (Get-Item 'HKCU:\Environment').GetValue('Path', '', 'DoNotExpandEnvironmentNames')
$entries = @($raw -split ';' | Where-Object { $_ -ne '' })
$dupes = @($entries | Group-Object { $_.TrimEnd('\').ToLowerInvariant() } | Where-Object Count -gt 1)
[PSCustomObject]@{ UserPathCount = $entries.Count; DuplicateCount = $dupes.Count }
```

- **PASS**: `DuplicateCount -eq 0` and re-running `Detect (Read-Only)` reports `Status -eq 'PASS'`.

## Rollback

```powershell
$latestBackup = Get-ChildItem -Path $env:TEMP -Filter 'user-path-backup-*.txt' -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1
if (-not $latestBackup) { throw 'No user-path-backup-*.txt found in $env:TEMP' }
$oldPath = Get-Content -Raw -Path $latestBackup.FullName
Set-ItemProperty -Path 'HKCU:\Environment' -Name 'Path' -Value $oldPath.TrimEnd("`r", "`n") -Type ExpandString
```

## Edge Cases

- Never use `setx PATH ...` to update User `PATH`: `setx` silently truncates values at 1,024 characters and merges Machine `PATH` into User `PATH` if `%PATH%` is expanded.
- Registry changes under `HKCU:\Environment` take effect in **new** shells/processes; already-open terminals retain their old `PATH`.
- Deliberately non-temp missing directories (e.g., offline drives or not-yet-installed tool directories) are preserved by the Fix above so valid configurations are not stripped.
