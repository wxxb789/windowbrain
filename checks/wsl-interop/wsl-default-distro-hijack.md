---
id: wsl-default-distro-hijack
category: wsl-interop
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 with multiple WSL2 distros (Docker Desktop, Rancher, Alpine, Ubuntu)
symptoms:
  - "wsl -- bash -lc ..." executes inside BusyBox /bin/sh or fails with "ps: unrecognized option"
  - Bare "wsl" launches docker-desktop or an unexpected minimal distro instead of Ubuntu/Debian
  - Scripts assuming GNU coreutils or bash fail silently or exit 1 under wsl.exe without -d
---

# Bare `wsl.exe` Default-Distro Hijack (Docker Desktop / BusyBox / Minimal Distro)

## Failure Mode

Running `wsl` or `wsl.exe -- bash ...` without `-d <DistroName>` dispatches to whichever WSL distro is marked default (`*` in `wsl -l -v`). Installing Docker Desktop, Rancher Desktop, or a custom Alpine/BusyBox test distro can leave a non-interactive or BusyBox-based distro as the default. AI agents invoking bare `wsl` then hit missing GNU flags (`ps -p`, `BASH_VERSION` empty, missing compilers) or `docker-desktop` errors instead of the user's real Linux dev distro.

## Detect (Read-Only)

```powershell
if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
  [PSCustomObject]@{ WslInstalled = $false }
} else {
  $lines = ((wsl.exe --list --verbose 2>&1) -replace "`0", '') -split "`r?`n" | Where-Object { $_.Trim() }
  $defaultLine = $lines | Where-Object { $_ -match '^\s*\*' } | Select-Object -First 1
  $defaultName = if ($defaultLine) { ($defaultLine -replace '^\s*\*\s*', '').Split(' ', [StringSplitOptions]::RemoveEmptyEntries)[0] } else { 'NONE' }
  $isSuspicious = $defaultName -match '^(docker-desktop|docker-desktop-data|rancher-desktop|podman-machine)'
  [PSCustomObject]@{
    WslInstalled       = $true
    DefaultDistro      = $defaultName
    SuspiciousDefault  = $isSuspicious
    AllDistros         = ($lines -join "`n")
  }
}
```

- **SKIP**: `WslInstalled -eq $false` or zero distros installed.
- **PASS**: `DefaultDistro` is a full interactive Linux dev distro (e.g., `Ubuntu*`, `Debian`) and `SuspiciousDefault -eq $false`.
- **FAIL**: `SuspiciousDefault -eq $true`, OR `DefaultDistro` is a minimal BusyBox/scratch distro rather than the user's primary Linux dev distro.

## Fix

1. **Set the user's intended Linux dev distro as default** (replace `<DISTRO>` with a confirmed distro name from `AllDistros`, e.g., `Ubuntu-24.04`):
```powershell
wsl.exe --set-default "<DISTRO>"
```

2. **Agent invocation rule**: Always pass `-d <DistroName>` explicitly when invoking `wsl.exe` from scripts or agent commands instead of relying on the global default distro.

## Verify

Re-run `Detect (Read-Only)` and confirm `DefaultDistro` matches `<DISTRO>` and `SuspiciousDefault -eq $false`.

## Rollback

```powershell
wsl.exe --set-default "<PREVIOUS_DEFAULT_DISTRO>"
```

## Edge Cases

- Some minimal or Alpine-based WSL distros symlink `/bin/sh -> /bin/busybox` and may lack GNU `bash`/`ps`/`find` options even when `/bin/bash` exists if a wrapper forces `/bin/sh`. Always probe `cat /etc/os-release` and `command -v bash` inside the target distro when debugging unexpected shell behavior.
