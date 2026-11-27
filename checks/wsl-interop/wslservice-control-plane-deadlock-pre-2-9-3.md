---
id: wslservice-control-plane-deadlock-pre-2-9-3
category: wsl-interop
severity: high
requires_admin: true
shell: powershell
reversible: true
applies_to: Windows 10/11 (wsl2, ai-agents)
symptoms:
  - "`wsl -l`, `wsl --status`, and `wsl --shutdown` all hang indefinitely (waiting on `LpcReply`) even though the Linux VM itself is still alive"
  - "`WslService` accumulates hundreds of threads and thousands of handles after an uncompleted distro/session termination"
  - "All AI agent `wsl.exe` tool calls time out until the host is rebooted or `WslService` is forcefully restarted"
---

# `WslService` Control-Plane Deadlock (`LpcReply` Hang) on WSL `< 2.9.3`

## Failure Mode

WSL versions prior to `2.9.3` (such as `2.7.12`) can suffer from a Windows host `WslService` control-plane deadlock: when a distro/session termination hangs without a deadline, `WslService` holds its lifecycle lock indefinitely. Subsequent `wsl.exe` control commands (`wsl -l`, `wsl --status`, `wsl --shutdown`) queue behind it waiting for an LPC reply even while the guest Linux VM data plane is still running. Updating to WSL `2.9.3+` resolves known VM termination timeout deadlocks.

## Detect (Read-Only)

```powershell
$wslPkg = Get-AppxPackage -Name 'MicrosoftCorporationII.WindowsSubsystemForLinux' -ErrorAction SilentlyContinue
if (-not $wslPkg) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'WSL MSIX/Store package not installed' }
    return
}

$ver = [version]$wslPkg.Version
$svc = Get-Process -Name 'wslservice' -ErrorAction SilentlyContinue
$threads = if ($svc) { $svc.Threads.Count } else { 0 }
$handles = if ($svc) { $svc.HandleCount } else { 0 }
$needsUpgrade = ($ver -lt [version]'2.9.3.0')

[PSCustomObject]@{
    Status            = if ($needsUpgrade -or $threads -ge 200) { 'FAIL' } else { 'PASS' }
    WslVersion        = $ver.ToString()
    NeedsUpgrade      = $needsUpgrade
    WslServiceThreads = $threads
    WslServiceHandles = $handles
}
```

- **PASS**: `Status -eq 'PASS'` (`WslVersion -ge 2.9.3.0` AND `WslServiceThreads -lt 200`).
- **FAIL**: `Status -eq 'FAIL'` (`WslVersion -lt 2.9.3.0` OR `WslServiceThreads -ge 200`).
- **SKIP**: `Status -eq 'SKIP'` (WSL MSIX package is not installed).

## Fix

> **Warning**: `wsl.exe --update` requires elevation (UAC) and restarts the WSL VM, terminating all running WSL distros and any agents/servers inside them. Drain active WSL sessions first.

```powershell
wsl.exe --update
```

If `WslService` is already deadlocked (causing `wsl --shutdown` / `wsl --update` to hang), restart `WslService` from an elevated PowerShell prompt (or reboot Windows):

```powershell
Restart-Service -Name WslService -Force
```

## Verify

```powershell
$wslPkg = Get-AppxPackage -Name 'MicrosoftCorporationII.WindowsSubsystemForLinux' -ErrorAction SilentlyContinue
$ver = if ($wslPkg) { [version]$wslPkg.Version } else { [version]'0.0.0.0' }
$job = Start-Job -ScriptBlock { & wsl.exe --status 2>&1 | Out-String }
$ok = [bool](Wait-Job -Job $job -Timeout 10)
Stop-Job -Job $job -ErrorAction SilentlyContinue
Remove-Job -Job $job -Force -ErrorAction SilentlyContinue

[PSCustomObject]@{
    WslVersion        = $ver.ToString()
    VersionOk         = ($ver -ge [version]'2.9.3.0')
    StatusRespondedOk = $ok
}
```

- **PASS**: `VersionOk -eq $true` AND `StatusRespondedOk -eq $true`.

## Rollback

No persistent config file is modified. If a WSL release introduces a regression, install a prior `.msixbundle` from the official Microsoft WSL GitHub Releases via `Add-AppxPackage`.

## Edge Cases

- Because `wsl --shutdown` itself goes through the deadlocked `WslService` LPC channel, running `wsl --shutdown` cannot unstick an already-deadlocked `WslService`; restarting `WslService` as Admin or rebooting Windows is required once wedged.
