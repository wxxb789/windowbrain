---
id: orphan-processes-and-junction-rm
category: processes
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (multi-agent worktrees, pnpm, local databases/daemons)
symptoms:
  - rm -rf fails with Permission denied or Directory not empty on worktree/node_modules
  - EBUSY / Access is denied / sharing violation when deleting temp dirs or git worktrees
  - Lingering background daemons (node, python, postgres, test servers) holding ports or file locks
  - Agent kills a still-active tool binary because its versioned path looks old behind a mise/scoop junction
---

# Orphan File-Locking Processes & MSYS `rm -rf` Junction Failures

## Failure Mode

Deleting old worktrees, temp directories, or `node_modules` frequently fails on Windows for two distinct reasons:
1. **Live handle lock**: An orphaned child daemon (test server, `postgres`, `node`, `python`, `pwsh`) still holds a file or working-directory handle inside the folder. Windows refuses to delete open executables or in-use directories (`EBUSY` / `ERROR_SHARING_VIOLATION`).
2. **MSYS `rm -rf` vs directory junctions**: Git Bash's `rm -rf` often fails with `Permission denied` on `pnpm` `node_modules` directory junctions (`ReparsePoint`) even when no process holds a lock. Windows-native deletion (`cmd /c rmdir /s /q` or `Remove-Item`) handles junctions cleanly without traversing into targets.

Safety hazard when hunting stale processes:
- **Resolve tool junctions before killing**: Version managers (`mise`, `scoop`) use a `latest` / `current` directory junction pointing to a versioned folder (e.g., `installs\tool\latest -> installs\tool\1.2.3`). A process running from a versioned path may be the active tool, not a stale orphan.

## Detect (Read-Only)

Given a locked target directory `<TARGET_DIR>` (or scanning for common orphaned dev servers):

```powershell
# 1. Find processes whose executable or command line references a target directory
$target = '<TARGET_DIR>' # replace with actual path when diagnosing a locked directory
if ($target -ne '<TARGET_DIR>' -and (Test-Path $target)) {
  $escaped = [regex]::Escape($target)
  Get-CimInstance Win32_Process |
    Where-Object { $_.ExecutablePath -match $escaped -or $_.CommandLine -match $escaped } |
    Select-Object ProcessId, ParentProcessId, Name, ExecutablePath, CommandLine
}

# 2. List high-memory or long-lived orphaned dev processes (read-only overview)
Get-CimInstance Win32_Process |
  Where-Object { $_.Name -match '^(node|bun|python|python3|pwsh|postgres)\.exe$' } |
  Select-Object ProcessId, ParentProcessId, Name, CreationDate, @{n='WS_MB';e={[math]::Round($_.WorkingSetSize/1MB,1)}} |
  Sort-Object WS_MB -Descending |
  Select-Object -First 20
```

- **PASS**: no unexpected processes holding handles in `<TARGET_DIR>` or consuming leaked memory/ports.
- **FAIL**: confirmed orphaned processes locking `<TARGET_DIR>` or holding stale ports.

## Fix

1. **Before killing any process**, inspect its parent chain and resolve any `latest`/`current` junction so active agent daemons are NOT killed:
```powershell
Get-Item "<TOOL_DIR>\latest" -ErrorAction SilentlyContinue | Select-Object FullName, LinkType, Target
```

2. **Stop confirmed orphaned processes**:
   - For PostgreSQL scratch clusters: `pg_ctl stop -m fast -D "<DATA_DIR>"` (preserves data integrity).
   - For other confirmed orphan PIDs: `Stop-Process -Id <PID> -Force` (or `taskkill /PID <PID> /T /F` to terminate the process tree).

3. **Delete directories containing `pnpm` junctions using Windows-native `rmdir`** instead of Git Bash `rm -rf`:
```powershell
cmd.exe /c "rmdir /s /q `"<TARGET_DIR>`""
```

## Verify

- Confirm `Test-Path "<TARGET_DIR>"` returns `$false`.
- Re-run `Get-CimInstance Win32_Process` for the stopped PIDs and confirm none remain.

## Rollback

Process termination and directory deletion are not reversible; always confirm the exact target directory and PID list with the user before executing `Fix`.

## Edge Cases

- Shared worker daemons spawned by coding agents may intentionally have an exited parent PID (orphan-looking PPID) because they are detached and shared across sessions; always check command line and active session ownership before killing.
- Never run `Remove-Item -Recurse` on a directory containing junctions in legacy Windows PowerShell 5.1 without verifying junction handling; `cmd /c rmdir /s /q` removes junctions safely without deleting junction targets.
