---
id: refs-cow-dir-reparse-fallback
category: paths-fs
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (ReFS / Dev Drive with Git worktrees or CoW clone tools)
symptoms:
  - worktree clone fell back to plain checkout: Access is denied. (os error 5)
  - Slow (30s-2min) worktree creation on ReFS Dev Drive despite CoW support
  - Block-clone fails on repos with pnpm node_modules, workspace junctions, or git submodules containing directory symlinks
---

# ReFS Block-Clone Failure on Directory Symlinks / Junctions (`os error 5`)

## Failure Mode

On Windows ReFS volumes, CoW worktree/clone tools that call `FSCTL_DUPLICATE_EXTENTS_TO_FILE` per file can fail mid-copy with `Access is denied. (os error 5)` and fall back to slow plain checkout when traversing **directory reparse points**:
- **Directory symlinks** (`ReparseTag = 0xA000000C`, `IO_REPARSE_TAG_SYMLINK`) — common inside checked-out third-party git submodules or skill repos.
- **Directory junctions** (`ReparseTag = 0xA0000003`, `IO_REPARSE_TAG_MOUNT_POINT`) — created by `pnpm` (`node_modules/<pkg>` -> `node_modules/.pnpm/...`), `bun --linker=isolated`, and `npm`/`bun` workspace links.

Important distinction on Windows:
- **Hard links** (`uv`, `bun` hoisted, `pnpm` inner store files): NOT reparse points -> safe for ReFS block-clone.
- **File symlinks** (`0xA000000C` on files): safe for most block-clone implementations.
- **Directory symlinks / junctions**: require `FILE_FLAG_BACKUP_SEMANTICS` + `FILE_FLAG_OPEN_REPARSE_POINT` handling; tools that open them like regular files hit `ERROR_ACCESS_DENIED (5)` after already spending time cloning thousands of earlier files.

## Detect (Read-Only)

Run inside the target repository (or pass `-Path <repo>`):

```powershell
$repoRoot = (Get-Location).Path
$driveLetter = [System.IO.Path]::GetPathRoot($repoRoot).TrimEnd(':\')
$fsType = (Get-Volume -DriveLetter $driveLetter -ErrorAction SilentlyContinue).FileSystemType

# Scan for directory reparse points (symlinks / junctions)
$dirReparse = Get-ChildItem -Path $repoRoot -Directory -Recurse -Force -ErrorAction SilentlyContinue |
  Where-Object { ($_.Attributes -band [IO.FileAttributes]::ReparsePoint) -ne 0 } |
  Select-Object -First 50 FullName, LinkType, Target

[PSCustomObject]@{
  RepoRoot             = $repoRoot
  FileSystem           = $fsType
  DirReparseSampleCount = @($dirReparse).Count
  SamplePaths          = @($dirReparse | Select-Object -First 10 -ExpandProperty FullName)
}
```

- **SKIP**: repository is not on a `ReFS` volume, or user is not experiencing CoW worktree clone failures.
- **PASS**: `DirReparseSampleCount -eq 0` (no directory symlinks or junctions blocking ReFS block-clone).
- **FAIL**: `FileSystem -eq 'ReFS'` AND `DirReparseSampleCount -gt 0` (list whether they originate from `pnpm` `node_modules`, workspace junctions, or checked-out git submodules).

## Fix

Choose remediation based on the origin of the detected directory reparse points (never delete tracked files or alter submodule commits):

1. **Origin = unused checked-out git submodules** (e.g., reference submodules not needed locally):
```powershell
# Clears local submodule working directories without modifying committed gitlink pointers
git submodule deinit --all
```

2. **Origin = `bun` isolated linker**:
- Switch to hoisted linker (`bun install --linker=hoisted`) if strict isolation is not required, eliminating top-level directory junctions.

3. **Origin = `pnpm` `node_modules` or required workspace junctions** (cannot remove junctions):
- Do NOT disable ReFS globally.
- Workaround A: Use plain `git worktree add` (fast for source files) and selectively copy/clone build caches, or use a ReFS-aware worktree CLI that handles reparse points properly (e.g., `riftri`).
- Workaround B: For ephemeral AI agent sandboxes only, use a host agent's `ProjFS` isolation backend (see `checks/paths-fs/projfs-vs-refs-cow.md`).

## Verify

- If Option 1 or 2 was applied, re-run `Detect (Read-Only)` and confirm `DirReparseSampleCount -eq 0`.
- Test creating a temporary worktree with the affected tool and verify it completes via block-clone without `os error 5` fallback.

## Rollback

```powershell
# Re-populate submodules if deinitialized in Option 1
git submodule update --init --recursive
```

## Edge Cases

- `git submodule deinit --all` fails if a submodule has uncommitted local modifications unless `--force` is passed; never pass `--force` without checking `git submodule status` first.
- This is a tool-level bug in block-clone walkers, not a ReFS permission error; running as Administrator does NOT fix `ERROR_ACCESS_DENIED (5)` caused by opening a directory reparse point without `FILE_FLAG_BACKUP_SEMANTICS`.
