---
id: wsl-nested-quoting-and-mnt-io
category: wsl-interop
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 with WSL2
symptoms:
  - wsl.exe bash -lc '...' expands empty variables or writes to Linux root '/'
  - Shell variables inside wsl.exe -e bash -lc "..." are prematurely expanded by Windows PowerShell/Git Bash
  - Extremely slow git/npm/cargo/go builds inside WSL2 when running directly on /mnt/c or /mnt/d
---

# `wsl.exe` Nested Quoting Traps & Cross-Filesystem `/mnt/<drive>` Pitfalls

## Failure Mode

1. **Nested quoting expansion hazards**: Calling `wsl.exe -d <distro> bash -lc '...'` from Windows PowerShell or Git Bash passes strings through multiple quote/escape layers. Unescaped `$VAR` expands on the Windows side to empty string—turning `rm -rf "$DIR/foo"` or `> "$OUT/file"` into operations on Linux `/`.
2. **Cross-OS filesystem (`/mnt/<drive>`) performance cliff**: Accessing Windows drives via WSL2's 9P (`/mnt/c/...`) has high metadata/syscall latency. Running `npm install`, `cargo build`, or large `git` operations across `/mnt/<drive>` is often 5x–20x slower than inside the Linux ext4 filesystem (`~/` or `/tmp/`).

## Detect (Read-Only)

Check WSL2 availability, default distro version (WSL 1 vs WSL 2), and confirm safe stdin invocation works:

```powershell
if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
  [PSCustomObject]@{ WslInstalled = $false }
} else {
  $list = (wsl.exe -l -v 2>&1) -replace "`0", ''
  $stdinProbe = 'set -eu; echo "ARG1=$1 UNAME=$(uname -s)"' | wsl.exe bash -s -- 'hello world' 2>&1
  [PSCustomObject]@{
    WslInstalled = $true
    DistrosRaw   = ($list -join "`n").Trim()
    StdinProbe   = ($stdinProbe -join "`n").Trim()
  }
}
```

- **SKIP**: `WslInstalled -eq $false` or no WSL distros are registered.
- **PASS**: WSL2 distro is present and `StdinProbe` outputs `ARG1=hello world UNAME=Linux`.
- **FAIL**: `StdinProbe` fails, or WSL distro is stuck on version 1 when WSL2 is expected.

## Fix

This is primarily an **agent invocation contract** (no system mutation required unless upgrading a WSL1 distro):

1. **Always pass multi-line or variable-bearing scripts to `wsl.exe` via stdin (`bash -s -- <args>`) with `set -eu`**, NEVER via nested `bash -lc '...'`:
```powershell
@'
set -eu
target="$1"
echo "Running safely on $target"
'@ | wsl.exe -d Ubuntu-24.04 -- bash -s -- "/tmp/work"
```
In Git Bash:
```bash
wsl.exe -d Ubuntu-24.04 -- bash -s -- "/tmp/work" <<'EOF'
set -eu
target="$1"
echo "Running safely on $target"
EOF
```

2. **Keep heavy Linux build/test trees off `/mnt/<drive>`**: when running Linux-only tests (`go test -race`, `cargo`, `npm`), clone or sync hot build artifacts into Linux ext4 (`/tmp/...` or `~/work/...`) instead of compiling directly on `/mnt/c`.

3. **Strip UTF-16LE null bytes from `wsl.exe -l` output** in PowerShell/scripts (`-replace "\`0", ""` or `WSL_UTF8=1`):
```powershell
[System.Environment]::SetEnvironmentVariable('WSL_UTF8', '1', 'User')
$env:WSL_UTF8 = '1'
```

## Verify

Re-run `Detect (Read-Only)` and confirm `[System.Environment]::GetEnvironmentVariable('WSL_UTF8', 'User') -eq '1'` (if set) and `StdinProbe` returns `ARG1=hello world UNAME=Linux`.

## Rollback

```powershell
[System.Environment]::SetEnvironmentVariable('WSL_UTF8', $null, 'User')
```

## Edge Cases

- `WSL_UTF8=1` switches `wsl.exe` management output (`wsl -l -v`, `wsl --status`) from UTF-16LE to UTF-8, preventing mojibake and `\0` byte parsing failures in agent tools.
- Passing `--` before `bash -s --` prevents `wsl.exe` from consuming flags intended for `bash`.
