---
id: interactive-shell-path-bleed-and-mise-cache-lock
category: toolchain-mcp
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (pwsh, git-bash, mise, ai-agents)
symptoms:
  - "Launching Git Bash from an interactive PowerShell session with `mise activate` explodes `PATH` to 200+ duplicate/malformed entries"
  - "Non-interactive AI agent PowerShell subshells load slow interactive PowerShell profiles (prompts, completions, `mise activate`)"
---

# Interactive Shell `PATH` Bleed into Non-Interactive Agent Subshells

## Failure Mode

When PowerShell profiles (`$PROFILE.CurrentUserCurrentHost` or `$PROFILE.CurrentUserAllHosts`) run `mise activate pwsh` or heavy interactive setup unconditionally, every child shell (including Git Bash or non-interactive AI agent subshells spawned via `pwsh -Command ...` without `-NoProfile`) inherits the mutated, expanded `PATH` and re-appends entries, inflating `PATH` past 100–200 entries and slowing or breaking tool resolution.

See `mise-path-shim-drift` for unpinned global shims (`No version is set for shim`) and concurrent `mise` cache lock (`os error 5`) handling.

## Detect (Read-Only)

```powershell
$hasMise = [bool](Get-Command mise -ErrorAction SilentlyContinue)
$pathEntries = @($env:PATH -split ';' | Where-Object { $_ -ne '' })
$dupeCount = @($pathEntries | Group-Object { $_.TrimEnd('\').ToLowerInvariant() } | Where-Object Count -gt 1).Count

$profilePaths = @(
    $PROFILE.CurrentUserCurrentHost,
    $PROFILE.CurrentUserAllHosts,
    $PROFILE.AllUsersCurrentHost,
    $PROFILE.AllUsersAllHosts
) | Where-Object { $_ -and (Test-Path $_) } | Select-Object -Unique

$unguardedProfiles = @(foreach ($pp in $profilePaths) {
    $txt = Get-Content -Raw -Path $pp -ErrorAction SilentlyContinue
    if ($txt -match 'mise\s+activate' -and $txt -notmatch 'GetCommandLineArgs|-NonInteractive|CLAUDECODE|CODEX_CI|\$env:CI') {
        $pp
    }
})

$status = if ($pathEntries.Count -ge 100 -or $dupeCount -gt 5 -or $unguardedProfiles.Count -gt 0) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status              = $status
    HasMise             = $hasMise
    ProcessPathEntries  = $pathEntries.Count
    DuplicatePathCount  = $dupeCount
    UnguardedProfiles   = $unguardedProfiles
}
```

- **PASS**: `Status -eq 'PASS'` (`ProcessPathEntries -lt 100`, `DuplicatePathCount -le 5`, and `UnguardedProfiles.Count -eq 0`).
- **FAIL**: `Status -eq 'FAIL'` (`ProcessPathEntries -ge 100` OR `DuplicatePathCount -gt 5` OR any PowerShell profile runs unguarded `mise activate`).
- **SKIP**: Not applicable.

## Fix

Prefer `mise` shims + `mise exec -- <cmd>` (see `mise-path-shim-drift`) over `mise activate` in PowerShell profiles. Before editing any `$PROFILE`, back it up, and guard interactive-only sections using command-line arguments (note: `[Environment]::UserInteractive` is `$true` even for agent subshells spawned inside a desktop session, so do **not** rely on it):

```powershell
# Back up each unguarded profile before editing
$bakDir = Join-Path $env:TEMP ('windowbrain-profile-bak-' + (Get-Date -Format 'yyyyMMdd-HHmmss'))
New-Item -ItemType Directory -Path $bakDir -Force | Out-Null
foreach ($pp in @($PROFILE.CurrentUserCurrentHost, $PROFILE.CurrentUserAllHosts) | Where-Object { $_ -and (Test-Path $_) }) {
    Copy-Item -Path $pp -Destination (Join-Path $bakDir (Split-Path -Leaf $pp)) -Force
}
Write-Output "Backed up profiles to $bakDir"
```

Place this guard directly before interactive-only initialization (prompts, completions, aliases, `mise activate`) inside `$PROFILE` (after any minimal environment setup that non-interactive sessions still need):

```powershell
$cmdArgs = [Environment]::GetCommandLineArgs()
$isNonInteractive = ($cmdArgs -match '^-(NonInteractive|Command|c|File|f|EncodedCommand|e)$').Count -gt 0
if ($isNonInteractive -or $env:CI -or $env:CLAUDECODE -or $env:CODEX_CI) {
    return
}
```

## Verify

```powershell
$entries = @((& pwsh -Command '($env:PATH -split ";") | Where-Object { $_ }') 2>$null)
$dupes = @($entries | Group-Object { $_.TrimEnd('\').ToLowerInvariant() } | Where-Object Count -gt 1)
[PSCustomObject]@{
    SubshellPathEntries = $entries.Count
    SubshellDupes       = $dupes.Count
}
```

- **PASS**: `SubshellPathEntries -lt 100` AND `SubshellDupes -le 5` AND re-running `Detect (Read-Only)` reports `Status -eq 'PASS'`.

## Rollback

```powershell
$latestBak = Get-ChildItem -Path $env:TEMP -Directory -Filter 'windowbrain-profile-bak-*' -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1
if ($latestBak) {
    foreach ($f in Get-ChildItem -Path $latestBak.FullName -File) {
        Copy-Item -Path $f.FullName -Destination (Join-Path (Split-Path -Parent $PROFILE.CurrentUserCurrentHost) $f.Name) -Force
    }
}
```

## Edge Cases

- Some agent harnesses invoke `pwsh -Command ...` *without* `-NoProfile`, which is why guarding `$PROFILE` itself (not only relying on callers passing `-NoProfile`) prevents interactive leakage.
- Machine `PATH` and User `PATH` can legitimately share 1–5 overlapping directories on stock Windows, so `DuplicatePathCount -le 5` avoids false positives while catching 200-entry PATH explosions.
