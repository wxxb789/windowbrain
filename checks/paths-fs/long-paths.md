---
id: long-paths
category: paths-fs
severity: high
requires_admin: true
shell: powershell
reversible: true
applies_to: Windows 10 (1607+) / Windows 11
symptoms:
  - Filename too long
  - ENAMETOOLONG
  - PathTooLongException
  - git checkout / clone / worktree fails on deep paths
  - npm / pnpm / cargo / uv fails creating nested files
---

# Windows Long Paths & Git `core.longpaths`

## Failure Mode

Windows enforces a legacy 260-character `MAX_PATH` limit unless `LongPathsEnabled=1` is set in the registry, and Git for Windows defaults `core.longpaths` to disabled. AI agents operating in nested worktrees, `node_modules`, Cargo caches, or Python virtualenvs hit opaque `Filename too long` / `ENAMETOOLONG` errors.

## Detect (Read-Only)

```powershell
$regPath = 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem'
$osLongPaths = (Get-ItemProperty -Path $regPath -Name LongPathsEnabled -ErrorAction SilentlyContinue).LongPathsEnabled
$hasGit = [bool](Get-Command git -ErrorAction SilentlyContinue)
$gitLongPaths = if ($hasGit) { (git config --global --get core.longpaths 2>$null) } else { 'NO_GIT' }

[PSCustomObject]@{
  OS_LongPathsEnabled = $osLongPaths
  Git_CoreLongPaths   = $gitLongPaths
}
```

- **PASS**:
  - `OS_LongPathsEnabled -eq 1`
  - If `git` is installed: `Git_CoreLongPaths -eq 'true'`
- **FAIL**: `OS_LongPathsEnabled -ne 1` OR (`git` installed and `Git_CoreLongPaths -ne 'true'`).

## Fix

Split by privilege level:

1. **Non-admin (run in user shell if `Git_CoreLongPaths -ne 'true'`)**:
```powershell
if (Get-Command git -ErrorAction SilentlyContinue) {
  git config --global core.longpaths true
}
```

2. **Admin (run in elevated PowerShell if `OS_LongPathsEnabled -ne 1`)**:
```powershell
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name 'LongPathsEnabled' -Value 1 -Type DWord
```

## Verify

Re-run the `Detect (Read-Only)` command block.
- **PASS**: `OS_LongPathsEnabled` is `1` and `Git_CoreLongPaths` is `'true'` (if Git is installed).

## Rollback

```powershell
# Admin: restore default MAX_PATH behavior
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name 'LongPathsEnabled' -Value 0 -Type DWord

# User: unset git core.longpaths
if (Get-Command git -ErrorAction SilentlyContinue) {
  git config --global --unset core.longpaths
}
```

## Edge Cases

- Enabling `LongPathsEnabled` takes effect for newly spawned processes; existing terminals/IDEs should be restarted.
- Individual executables still need a `longPathAware` application manifest (modern Git, Node.js, Python, Rust/Cargo, PowerShell 7+, and Go binaries include it; legacy 32-bit tools may still fail >260 chars).
