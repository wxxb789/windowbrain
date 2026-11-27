---
id: projfs-vs-refs-cow
category: paths-fs
severity: low
requires_admin: true
shell: powershell
reversible: true
applies_to: Windows 10 (1809+) / Windows 11
symptoms:
  - Agent sandbox / isolated worktree creation takes 30s-2min on huge repositories
  - Unsure whether to enable Windows Projected File System (Client-ProjFS) alongside ReFS Dev Drive
  - ProjFS backend fails with missing ProjectedFSLib.dll or prjflt service not running
---

# Windows Projected File System (ProjFS) Readiness & ReFS Coexistence

## Failure Mode

AI agents supporting `ProjFS` (Windows Projected File System) can create O(1) millisecond isolated sandboxes even on massive repos (100k+ files), avoiding O(N) file enumeration. However, `Client-ProjFS` is an optional Windows feature disabled by default.

Architecture rules for agents advising users on `ProjFS` vs `ReFS block-clone`:
1. **Not mutually exclusive**: `ReFS` is the volume filesystem; `ProjFS` (`prjflt.sys`) is a minifilter that works on both NTFS and ReFS (including modern Windows 11 Dev Drives).
2. **Host-process dependent**: enabling `Client-ProjFS` does NOT accelerate ordinary `git`, `cp`, or builds. Only hosts that actively run a ProjFS provider (`PrjStartVirtualizing`, e.g., agent isolation sandboxes or VFS for Git) benefit.
3. **Choose by lifecycle**:
   - **Ephemeral agent sandboxes** (short-lived isolated tasks touching a few files in a huge repo): `ProjFS` excels (O(1) startup, unaccessed dirs/submodules cost zero).
   - **Persistent worktrees** (long-lived PR branches, IDE/multi-tool access, survives restarts): `ReFS block-clone` excels (static independent files, native read speed, zero provider-process dependency).
   - Default recommendation when a tool supports both: keep `auto` rather than forcing `projfs` globally for persistent worktrees.

## Detect (Read-Only)

```powershell
$svc = Get-Service -Name prjflt -ErrorAction SilentlyContinue
$dll = Test-Path "$env:SystemRoot\System32\ProjectedFSLib.dll"
$feature = Get-WindowsOptionalFeature -Online -FeatureName Client-ProjFS -ErrorAction SilentlyContinue

[PSCustomObject]@{
  PrjFltServiceStatus = if ($svc) { $svc.Status.ToString() } else { 'NOT_INSTALLED' }
  ProjectedFSLibDll   = $dll
  OptionalFeature     = if ($feature) { $feature.State.ToString() } else { 'REQUIRES_ADMIN_TO_QUERY' }
}
```

- **PASS**: `PrjFltServiceStatus -eq 'Running'` AND `ProjectedFSLibDll -eq $true`.
- **FAIL**: `PrjFltServiceStatus -ne 'Running'` OR `ProjectedFSLibDll -eq $false` (only flag as actionable if the user uses an agent/tool with a ProjFS backend).
- **SKIP**: user does not use any ProjFS-capable host tool.

## Fix

Requires Administrator (elevated PowerShell):

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Client-ProjFS -NoRestart
```

## Verify

Re-run `Detect (Read-Only)`:
- **PASS**: `ProjectedFSLibDll` is `$true` and `PrjFltServiceStatus` is `'Running'` (or becomes `'Running'` after reboot if `Enable-WindowsOptionalFeature` reported `RestartNeeded = True`).

## Rollback

Requires Administrator (elevated PowerShell):

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName Client-ProjFS -NoRestart
```

## Edge Cases

- `Enable-WindowsOptionalFeature` may require a system reboot on some Windows builds (`RestartNeeded = True`).
- If a ProjFS host provider exits or crashes, unhydrated virtual files in its projection root become unreadable; never store long-lived uncommitted work solely inside an ephemeral ProjFS projection.
- Full-tree scans (`rg`, full compilation) inside a fresh ProjFS root incur kernel-to-user provider callback overhead on first read of each file.
