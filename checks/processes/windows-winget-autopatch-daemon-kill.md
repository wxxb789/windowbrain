---
id: windows-winget-autopatch-daemon-kill
category: processes
severity: low
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (devbox, winget, agent-daemons)
symptoms:
  - "Long-running desktop agent orchestrator or daemon vanishes overnight with no graceful shutdown log"
  - "Child agent transcripts show flush timestamps matching the exact crash minute (victims, not the root cause)"
  - "Windows Event Log shows automated `winget upgrade` / endpoint patching killing the app without relaunching it"
---

# Unattended `winget` / Endpoint Patching Hard-Killing Agent Daemons

## Failure Mode

On managed Windows workstations and Dev Boxes, automated patch managers (e.g., Intune/endpoint patching or scheduled `winget upgrade --all`) silently upgrade desktop agent hosts and daemons using NSIS/Inno installers that hard-kill the running process (`TerminateProcess`) and never restart it. Because no graceful shutdown log is written, operators and agents waste time chasing phantom crashes or blaming child sessions.

## Detect (Read-Only)

```powershell
if (-not (Get-Command winget -ErrorAction SilentlyContinue)) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'winget not installed' }
    return
}

$since = (Get-Date).AddHours(-48)
$pcaEvents = @(Get-WinEvent -FilterHashtable @{
    LogName   = 'Microsoft-Windows-Application-Experience/Program-Compatibility-Assistant'
    Id        = 17
    StartTime = $since
} -ErrorAction SilentlyContinue | Select-Object -First 5 TimeCreated, Message)

$pinned = (& winget pin list 2>$null | Out-String)
$hasPins = ($pinned -match 'Pinning|Blocking')

[PSCustomObject]@{
    Status                = if ($pcaEvents.Count -gt 0 -and -not $hasPins) { 'FAIL' } else { 'PASS' }
    RecentInstallerEvents = $pcaEvents
    WingetPinsConfigured  = $hasPins
}
```

- **PASS**: `Status -eq 'PASS'` (no recent unexplained installer kills, or critical always-on agent daemons are pinned in `winget`).
- **FAIL**: `Status -eq 'FAIL'` (recent PCA Event ID 17 installer runs occurred in the last 48h and no `winget` blocking pins are configured).
- **SKIP**: `Status -eq 'SKIP'` (`winget` is not installed).

## Fix

Pin long-running agent host apps from unattended `winget upgrade --all` (upgrade them intentionally between sessions):

```powershell
# Replace with the exact package ID from `winget list`
$PackageId = 'Publisher.AgentHost'
winget pin add --id $PackageId --blocking
```

## Verify

```powershell
$PackageId = 'Publisher.AgentHost'
$pins = (& winget pin list 2>$null | Out-String)
[PSCustomObject]@{ Pinned = ($pins -match [regex]::Escape($PackageId)) }
```

- **PASS**: `Pinned -eq $true`.

## Rollback

```powershell
$PackageId = 'Publisher.AgentHost'
winget pin remove --id $PackageId
```

## Edge Cases

- Corporate endpoint management jobs may pass `--include-pinned` or run outside user-scoped `winget` pins; in that case, use the PCA Event ID 17 + RestartManager log query as a 10-second root-cause diagnostic before debugging the daemon itself, and register a Windows Scheduled Task / watchdog if unattended uptime is required.
- See also `winget-inplace-upgrade-corrupts-mise-shims` for WinGet non-atomic in-place upgrades of `mise`.
