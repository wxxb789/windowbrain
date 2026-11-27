---
id: devdrive-same-volume-caches
category: perf-defender
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 11 (Dev Drive / ReFS) & multi-volume Windows 10/11
symptoms:
  - Slow uv / pnpm / bun / npm / cargo / NuGet installs on a Dev Drive or secondary data drive
  - Package manager falls back from hardlink/CoW to slow full file copy across volumes
  - Windows Defender real-time scan slows down package cache extraction on C:\
  - npm warn Unknown env/user config "store-dir" or "cache-dir"
---

# Same-Volume Package Caches on ReFS / Dev Drive (Hardlink vs CoW)

## Failure Mode

When repositories live on a secondary drive or Windows 11 ReFS Dev Drive (e.g., `D:\`, `E:\`), default package caches stay on `C:\Users\<user>\...`. Cross-volume boundaries break hardlinks (`CreateHardLinkW`) and ReFS block-cloning (`FSCTL_DUPLICATE_EXTENTS_TO_FILE`), forcing package managers (`uv`, `pnpm`, `bun`) to fall back to slow full file copies and subjecting caches on `C:\` to synchronous Defender real-time scans.

Key Windows ReFS performance facts (do NOT invert these):
- **Same-volume `hardlink` is faster than ReFS `block-clone` (`clone`)** for read-only dependency installs (`uv`, `pnpm`, `bun`):
  - Unlike macOS APFS (single-syscall directory `clonefile`), Windows ReFS requires 5+ syscalls per file (`CreateFile`, `FSCTL_SET_SPARSE`, integrity sync, `SetEndOfFile`, `FSCTL_DUPLICATE_EXTENTS_TO_FILE`), whereas `CreateHardLinkW` is 1 syscall with zero extra disk usage.
  - Keep `uv` (`hardlink` default), `pnpm` (`packageImportMethod: auto` -> hardlink first on Windows), and `bun` (`--backend=hardlink` default; no Windows `clone` backend) on hardlink unless the user explicitly edits files inside `.venv`/`node_modules` and needs write isolation (`UV_LINK_MODE=clone` / `packageImportMethod: clone`).
- **`npm`** caches compressed `.tgz` tarballs and extracts plain files; it has no hardlink/CoW import mode, but moving `npm` cache/prefix to the Dev Drive still benefits from Dev Drive asynchronous Defender scanning.
- **Avoid `NPM_CONFIG_STORE_DIR` / `NPM_CONFIG_CACHE_DIR` or `store-dir` / `cache-dir` in `~/.npmrc`**: those are `pnpm`-only keys and emit deprecation/unknown-config warnings in `npm 11+`. Configure `pnpm` in its own `config.yaml` (`%LOCALAPPDATA%\pnpm\config\config.yaml` or `~/.config/pnpm/config.yaml`) instead.

## Detect (Read-Only)

```powershell
# 1. Discover ReFS / Dev Drive volumes and current repo drive
$volumes = Get-Volume | Where-Object { $_.DriveLetter } | Select-Object DriveLetter, FileSystemType, FileSystemLabel, @{n='SizeGB';e={[math]::Round($_.Size/1GB,1)}}
$cwdRoot = [System.IO.Path]::GetPathRoot((Get-Location).Path)

# 2. Inspect effective cache paths for installed tools
$checks = [ordered]@{
  CwdDrive               = $cwdRoot
  UV_CACHE_DIR           = [System.Environment]::GetEnvironmentVariable('UV_CACHE_DIR', 'User')
  BUN_INSTALL_CACHE_DIR  = [System.Environment]::GetEnvironmentVariable('BUN_INSTALL_CACHE_DIR', 'User')
  CARGO_HOME             = [System.Environment]::GetEnvironmentVariable('CARGO_HOME', 'User')
  NUGET_PACKAGES         = [System.Environment]::GetEnvironmentVariable('NUGET_PACKAGES', 'User')
  NPM_CONFIG_CACHE       = [System.Environment]::GetEnvironmentVariable('npm_config_cache', 'User')
  Legacy_NPM_STORE_DIR   = [System.Environment]::GetEnvironmentVariable('NPM_CONFIG_STORE_DIR', 'User')
  Legacy_NPM_CACHE_DIR   = [System.Environment]::GetEnvironmentVariable('NPM_CONFIG_CACHE_DIR', 'User')
}

if (Get-Command uv -ErrorAction SilentlyContinue)   { $checks.UV_Effective   = (uv cache dir 2>$null) }
if (Get-Command pnpm -ErrorAction SilentlyContinue) { $checks.PNPM_Store     = (pnpm store path 2>$null) }
if (Get-Command npm -ErrorAction SilentlyContinue)  { $checks.NPM_Cache      = (npm config get cache 2>$null) }

$volumes | Format-Table -AutoSize
[PSCustomObject]$checks
```

- **SKIP**: all developer repos live on `C:\` and no secondary/ReFS Dev Drive exists, AND no legacy `NPM_CONFIG_STORE_DIR`/`NPM_CONFIG_CACHE_DIR` keys are set.
- **PASS**:
  - For every installed tool (`uv`, `pnpm`, `bun`, `npm`, `cargo`, `dotnet`), its effective cache/store path is on the same drive letter as the primary repo/Dev Drive volume; AND
  - Neither `NPM_CONFIG_STORE_DIR` nor `NPM_CONFIG_CACHE_DIR` is set (and `~/.npmrc` does not contain `store-dir` or `cache-dir`).
- **FAIL**: any installed tool's cache/store is on a different volume than the Dev Drive / repo volume, OR legacy `pnpm` keys pollute `npm` config.

## Fix

Replace `<TARGET_DRIVE>` with the detected Dev Drive / repo drive letter (e.g., `D:`) confirmed with the user. Do NOT invent a drive letter; derive it from `Get-Volume` / user repo root.

```powershell
$targetRoot = '<TARGET_DRIVE>\.dev-caches'
New-Item -ItemType Directory -Path $targetRoot -Force | Out-Null

# uv (keep default hardlink mode; only relocate cache to same volume)
if (Get-Command uv -ErrorAction SilentlyContinue) {
  $p = Join-Path $targetRoot 'uv\cache'
  New-Item -ItemType Directory -Path $p -Force | Out-Null
  [System.Environment]::SetEnvironmentVariable('UV_CACHE_DIR', $p, 'User')
  $env:UV_CACHE_DIR = $p
}

# bun (keep default hardlink backend; only relocate cache to same volume)
if (Get-Command bun -ErrorAction SilentlyContinue) {
  $p = Join-Path $targetRoot 'bun\cache'
  New-Item -ItemType Directory -Path $p -Force | Out-Null
  [System.Environment]::SetEnvironmentVariable('BUN_INSTALL_CACHE_DIR', $p, 'User')
  $env:BUN_INSTALL_CACHE_DIR = $p
}

# pnpm (configure via pnpm config, NOT ~/.npmrc or NPM_CONFIG_*)
if (Get-Command pnpm -ErrorAction SilentlyContinue) {
  $store = Join-Path $targetRoot 'pnpm\store'
  $cache = Join-Path $targetRoot 'pnpm\cache'
  New-Item -ItemType Directory -Path $store, $cache -Force | Out-Null
  pnpm config set store-dir $store --global
  pnpm config set cache-dir $cache --global
}

# npm (Dev Drive async AV benefit; remove pnpm-only keys if present)
if (Get-Command npm -ErrorAction SilentlyContinue) {
  $p = Join-Path $targetRoot 'npm\cache'
  New-Item -ItemType Directory -Path $p -Force | Out-Null
  npm config set cache $p --location=user
  [System.Environment]::SetEnvironmentVariable('NPM_CONFIG_STORE_DIR', $null, 'User')
  [System.Environment]::SetEnvironmentVariable('NPM_CONFIG_CACHE_DIR', $null, 'User')
  npm config delete store-dir --location=user 2>$null
  npm config delete cache-dir --location=user 2>$null
}

# cargo / NuGet (optional, if installed)
if (Get-Command cargo -ErrorAction SilentlyContinue) {
  $p = Join-Path $targetRoot 'cargo'
  New-Item -ItemType Directory -Path $p -Force | Out-Null
  [System.Environment]::SetEnvironmentVariable('CARGO_HOME', $p, 'User')
}
if (Get-Command dotnet -ErrorAction SilentlyContinue) {
  $p = Join-Path $targetRoot 'nuget\packages'
  New-Item -ItemType Directory -Path $p -Force | Out-Null
  [System.Environment]::SetEnvironmentVariable('NUGET_PACKAGES', $p, 'User')
}
```

## Verify

Re-run the `Detect (Read-Only)` block and verify:
- Every installed tool's effective cache/store path begins with `<TARGET_DRIVE>\`.
- If `npm` is installed, `npm config list 2>&1` emits zero `Unknown env config` / `Unknown user config` warnings for `store-dir` or `cache-dir`.

## Rollback

```powershell
foreach ($var in @('UV_CACHE_DIR','BUN_INSTALL_CACHE_DIR','CARGO_HOME','NUGET_PACKAGES')) {
  [System.Environment]::SetEnvironmentVariable($var, $null, 'User')
}
if (Get-Command pnpm -ErrorAction SilentlyContinue) {
  pnpm config delete store-dir --global
  pnpm config delete cache-dir --global
}
if (Get-Command npm -ErrorAction SilentlyContinue) {
  npm config delete cache --location=user
}
```

## Edge Cases

- Existing caches on `C:\` are not deleted automatically; the user may reclaim disk space after verifying new caches work.
- New user environment variables require opening a new shell/agent session to take effect in child processes.
- Do NOT force `UV_LINK_MODE=clone` or `pnpm package-import-method=clone` as a blanket performance optimization on Windows ReFS; benchmarked ~1.5x–2x slower than same-volume `hardlink`.
