---
id: scoop-shim-broken-by-self-update
category: toolchain-mcp
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (scoop, mise, topgrade, ai-agents)
symptoms:
  - "`Shim: Could not determine if target is a GUI app. Assuming console.` followed by `Shim: Could not create process with command '\"...\\scoop\\apps\\mise\\current\\bin\\mise.exe\" ...'`"
  - "Running `topgrade` or `mise self-update` moves or replaces a Scoop-managed binary, breaking the Scoop `current\\` junction and shims"
---

# Scoop Shim Broken by `mise self-update` / `topgrade`

## Failure Mode

When a CLI tool such as `mise` is installed via Scoop, Scoop creates a `current` junction and `.shim` wrappers pointing at `<scoop>\apps\<tool>\current\bin\<tool>.exe`. If `topgrade` or the user invokes `mise self-update`, the tool overwrites or moves its own executable out from under Scoop's versioned directory (and fails mid-swap if a background daemon holds `mise.exe` open), leaving Scoop's `current\bin\mise.exe` missing and breaking every command invoked through the Scoop shim.

## Detect (Read-Only)

```powershell
$scoopRoot = if ($env:SCOOP) { $env:SCOOP } else { Join-Path $env:USERPROFILE 'scoop' }
$scoopShim = Join-Path $scoopRoot 'shims\mise.exe'
$scoopMise = Join-Path $scoopRoot 'apps\mise\current\bin\mise.exe'
$selfUpdateFlag = [Environment]::GetEnvironmentVariable('MISE_SELF_UPDATE_AVAILABLE', 'User')
if (-not $selfUpdateFlag) { $selfUpdateFlag = $env:MISE_SELF_UPDATE_AVAILABLE }

if (-not (Test-Path $scoopShim)) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'mise is not managed by Scoop' }
    return
}

$targetOk = Test-Path $scoopMise
$disabled = ($selfUpdateFlag -ieq 'false')
$status = if ($targetOk -and $disabled) { 'PASS' } else { 'FAIL' }

[PSCustomObject]@{
    Status                 = $status
    ScoopRoot              = $scoopRoot
    ScoopShimExists        = $true
    ScoopTargetExists      = $targetOk
    MiseSelfUpdateDisabled = $disabled
}
```

- **PASS**: `Status -eq 'PASS'` (`ScoopTargetExists -eq $true` AND `MiseSelfUpdateDisabled -eq $true`).
- **FAIL**: `Status -eq 'FAIL'` (`ScoopTargetExists -eq $false` OR `MiseSelfUpdateDisabled -eq $false`).
- **SKIP**: `mise` is not managed by Scoop.

## Fix

Disable `mise`'s built-in self-updater so Scoop owns upgrades cleanly:

```powershell
[Environment]::SetEnvironmentVariable('MISE_SELF_UPDATE_AVAILABLE', 'false', 'User')
$env:MISE_SELF_UPDATE_AVAILABLE = 'false'
```

If `ScoopTargetExists -eq $false`, close any background processes holding `mise.exe` and repair via Scoop:

```powershell
scoop reset mise
scoop update mise
```

## Verify

```powershell
$flag = [Environment]::GetEnvironmentVariable('MISE_SELF_UPDATE_AVAILABLE', 'User')
$ver = & mise --version 2>&1 | Out-String
[PSCustomObject]@{
    UserEnvFlag = $flag
    MiseExit    = $LASTEXITCODE
    MiseVersion = $ver.Trim()
}
```

- **PASS**: `UserEnvFlag -eq 'false'` AND `MiseExit -eq 0` with zero `Shim: Could not create process` errors.

## Rollback

```powershell
[Environment]::SetEnvironmentVariable('MISE_SELF_UPDATE_AVAILABLE', $null, 'User')
Remove-Item Env:MISE_SELF_UPDATE_AVAILABLE -ErrorAction SilentlyContinue
```

## Edge Cases

- `topgrade`'s `mise` step checks stderr for `cannot update` when `MISE_SELF_UPDATE_AVAILABLE=false` and skips self-update cleanly without failing the rest of the upgrade run.
- Never run `mise self-update --yes` in a read-only verification step: if the env var is not honored in the active shell, it would perform the exact mutation this check prevents.
