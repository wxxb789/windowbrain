---
id: corporate-registry-lockfile-pollution
category: toolchain-mcp
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 enterprise/corporate dev machines with internal npm/pnpm/Cargo/PyPI mirrors
symptoms:
  - "ERR_PNPM_TARBALL_URL_MISMATCH" in public GitHub Actions CI after editing dependencies on Windows
  - pnpm-lock.yaml or package-lock.json contains private/internal feed tarball URLs or sha1 integrity hashes
  - Local "pnpm install --frozen-lockfile" passes while public CI rejects the lockfile
---

# Corporate Package Feed Polluting Public Repo Lockfiles

## Failure Mode

On corporate-managed Windows workstations, global `~/.npmrc`, `NPM_CONFIG_REGISTRY`, or `~/.cargo/config.toml` often route packages through an internal artifact proxy/feed. When an AI agent runs `pnpm add`/`pnpm update`/`npm install` inside a **public** repository on that machine, the package manager writes internal feed tarball URLs (and sometimes `sha1-` hashes) into `pnpm-lock.yaml` / `package-lock.json`. Local `--frozen-lockfile` passes against the same internal proxy, hiding the defect until public CI rejects the lockfile with `ERR_PNPM_TARBALL_URL_MISMATCH`.

## Detect (Read-Only)

Inspect global npm/pnpm registries and scan the current repository's lockfiles (if present) for non-public registry URLs:

```powershell
$npmReg = if (Get-Command npm -ErrorAction SilentlyContinue) { (npm config get registry 2>$null) } else { 'NO_NPM' }
$pnpmReg = if (Get-Command pnpm -ErrorAction SilentlyContinue) { (pnpm config get registry 2>$null) } else { 'NO_PNPM' }
$envReg = [System.Environment]::GetEnvironmentVariable('NPM_CONFIG_REGISTRY', 'User')

$lockHits = @()
foreach ($lf in @('pnpm-lock.yaml', 'package-lock.json')) {
  if (Test-Path $lf) {
    $hits = Select-String -Path $lf -Pattern 'tarball:\s*https?://(?!registry\.npmjs\.org)|"resolved":\s*"https?://(?!registry\.npmjs\.org)|sha1-' | Select-Object -First 5
    if ($hits) { $lockHits += "$lf ($($hits.Count)+ matches)" }
  }
}

[PSCustomObject]@{
  NpmGlobalRegistry  = $npmReg
  PnpmGlobalRegistry = $pnpmReg
  EnvNpmRegistry     = $envReg
  IsNonPublicGlobal  = ($npmReg -notmatch 'registry\.npmjs\.org|NO_NPM') -or ($pnpmReg -notmatch 'registry\.npmjs\.org|NO_PNPM')
  LockfilePollution  = $lockHits
}
```

- **SKIP**: `IsNonPublicGlobal -eq $false` AND `LockfilePollution.Count -eq 0`.
- **PASS**: `LockfilePollution.Count -eq 0` (and public repos explicitly pin `registry=https://registry.npmjs.org/` in repo-local `.npmrc` if global registry is private).
- **FAIL**: `LockfilePollution.Count -gt 0`, OR `IsNonPublicGlobal -eq $true` inside a public repo without a repo-local `.npmrc` pinning the public registry.

## Fix

1. **Prevent future pollution in public repos**: add or confirm a repo-local `.npmrc` pinning the public registry (when public registry is reachable):
```powershell
Add-Content -Path '.npmrc' -Value "registry=https://registry.npmjs.org/" -Encoding UTF8
```

2. **Before committing lockfile changes on a corporate-proxy host**, run a pre-commit check confirming zero private feed URLs or `sha1-` entries:
```powershell
Select-String -Path 'pnpm-lock.yaml' -Pattern 'tarball:\s*https?://(?!registry\.npmjs\.org)|sha1-'
```
If public `registry.npmjs.org` is blocked by corporate proxy, regenerate the lockfile in a clean CI/container environment or rewrite polluted tarball entries to standard `sha512-` integrity without private `tarball:` URLs after verifying package hashes.

## Verify

Re-run `Detect (Read-Only)` and confirm `LockfilePollution.Count -eq 0`.

## Rollback

Remove the added `registry=https://registry.npmjs.org/` line from repo-local `.npmrc` if the repo is actually private and requires the internal feed.

## Edge Cases

- Do NOT blindly overwrite the user's global `~/.npmrc` registry on a corporate machine—private/work repositories may depend on the internal feed. Scope overrides per-repository or per-command (`--registry=https://registry.npmjs.org/`).
