---
id: wsl-mirrored-gsa-dns-blackhole
category: wsl-interop
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (wsl2, vpn, corporate-network, ai-agents)
symptoms:
  - "wsl: Due to a current compatibility issue with Global Secure Access Client, DNS Tunneling is disabled."
  - "WSL2 cannot resolve external hosts (`getent hosts`, `curl`, `apt`, `pip`, `git fetch` time out) while the Windows host has full internet access"
  - "`wsl --list` or WSL startup hangs indefinitely under `networkingMode=mirrored`"
  - "Every `wsl.exe` invocation prints `localhostForwarding has no effect when using mirrored networking mode`"
---

# WSL2 Mirrored Networking & Corporate SASE/VPN DNS Blackhole

## Failure Mode

When `$env:USERPROFILE\.wslconfig` sets `networkingMode=mirrored` (often paired with `dnsTunneling=true` and `autoProxy=true`) on a Windows host running a corporate SASE/VPN client such as Microsoft Global Secure Access (GSA), WSL disables DNS tunneling at boot and writes the corporate DNS server directly into `/etc/resolv.conf`. Under mirrored networking, UDP/TCP DNS queries from inside WSL to both corporate and public resolvers silently time out, cutting off all package managers, `git`, and agent network calls inside WSL even though the Windows host resolves normally. Separately, setting `localhostForwarding=true` alongside `networkingMode=mirrored` spams a warning on every `wsl.exe` invocation, corrupting structured stdout/stderr parsers.

## Detect (Read-Only)

```powershell
$wslCmd = Get-Command wsl.exe -ErrorAction SilentlyContinue
if (-not $wslCmd) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'wsl.exe not installed' }
    return
}

$wslCfgPath = Join-Path $env:USERPROFILE '.wslconfig'
$cfgText = if (Test-Path $wslCfgPath) { Get-Content -Raw -Path $wslCfgPath } else { '' }
$isMirrored = $cfgText -match '(?im)^\s*networkingMode\s*=\s*mirrored\b'
$hasLocalhostFwd = $cfgText -match '(?im)^\s*localhostForwarding\s*='
$gsaRunning = @(Get-Service -Name 'GlobalSecureAccess*' -ErrorAction SilentlyContinue | Where-Object Status -eq 'Running').Count -gt 0

# Bound wsl.exe probe with a timeout so a wedged WSL control plane does not hang the audit
$job = Start-Job -ScriptBlock {
    & wsl.exe -e sh -c 'getent hosts github.com >/dev/null 2>&1 && echo DNS_OK || echo DNS_FAIL' 2>&1 | Out-String
}
$completed = Wait-Job -Job $job -Timeout 15
if (-not $completed) {
    Stop-Job -Job $job -ErrorAction SilentlyContinue
    Remove-Job -Job $job -Force -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Status              = 'FAIL'
        MirroredMode        = $isMirrored
        HasLocalhostFwd     = $hasLocalhostFwd
        GsaServiceRunning   = $gsaRunning
        TimedOut            = $true
        WslDnsOk            = $false
    }
    return
}
$probe = Receive-Job -Job $job
Remove-Job -Job $job -Force -ErrorAction SilentlyContinue

$dnsWarn = ($probe -match 'DNS Tunneling is disabled')
$dnsOk = ($probe -match 'DNS_OK')
$status = if (($isMirrored -and -not $dnsOk) -or ($isMirrored -and $hasLocalhostFwd)) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status              = $status
    MirroredMode        = $isMirrored
    HasLocalhostFwd     = $hasLocalhostFwd
    GsaServiceRunning   = $gsaRunning
    DnsTunnelingWarning = $dnsWarn
    WslDnsOk            = $dnsOk
    TimedOut            = $false
}
```

- **PASS**: `Status -eq 'PASS'` (`WslDnsOk -eq $true`, `TimedOut -eq $false`, and NOT (`MirroredMode -and HasLocalhostFwd`)).
- **FAIL**: `Status -eq 'FAIL'` (`TimedOut -eq $true`, OR (`MirroredMode -and -not $WslDnsOk`), OR (`MirroredMode -and $HasLocalhostFwd`)).
- **SKIP**: `wsl.exe` is not installed.

## Fix

Edit only `networkingMode` / `localhostForwarding` in `$env:USERPROFILE\.wslconfig` (preserving `memory`, `processors`, `swap`, `kernel`, and other settings), and back up once before modifying:

```powershell
$ErrorActionPreference = 'Stop'
$wslCfgPath = Join-Path $env:USERPROFILE '.wslconfig'
$backupPath = Join-Path $env:USERPROFILE ('.wslconfig.bak.' + (Get-Date -Format 'yyyyMMdd-HHmmss'))

$lines = @()
if (Test-Path $wslCfgPath) {
    Copy-Item -Path $wslCfgPath -Destination $backupPath -Force
    if (-not (Test-Path $backupPath)) { throw "Backup failed: $backupPath" }
    $lines = Get-Content -Path $wslCfgPath
}

# Set $SwitchToNat = $true when DNS fails under mirrored mode; set $false if only removing localhostForwarding
$SwitchToNat = $true
$out = [System.Collections.Generic.List[string]]::new()
$sawSection = $false
$wroteMode = $false

foreach ($line in $lines) {
    if ($line -match '^\s*\[wsl2\]\s*$') { $sawSection = $true }
    if ($line -match '^\s*localhostForwarding\s*=') { continue }
    if ($SwitchToNat -and $line -match '^\s*networkingMode\s*=') {
        $out.Add('networkingMode=nat')
        $wroteMode = $true
        continue
    }
    $out.Add($line)
}
if ($SwitchToNat -and -not $wroteMode) {
    if (-not $sawSection) { $out.Add('[wsl2]') }
    $out.Add('networkingMode=nat')
}

[IO.File]::WriteAllText($wslCfgPath, (($out -join "`n") + "`n"), [Text.UTF8Encoding]::new($false))
Write-Output "Updated $wslCfgPath (backup: $backupPath)"
```

> **Warning**: The next command (`wsl.exe --shutdown`) terminates **all** running WSL distros and any background agents/servers inside them. Confirm active WSL sessions are drained before running it.

```powershell
wsl.exe --shutdown
```

## Verify

```powershell
$out = & wsl.exe -e sh -c 'ip route | head -n 1; getent hosts github.com | head -n 1' 2>&1 | Out-String
[PSCustomObject]@{
    ExitCode   = $LASTEXITCODE
    Output     = $out.Trim()
    HasWarning = ($out -match 'localhostForwarding has no effect')
}
```

- **PASS**: `ExitCode -eq 0`, `Output` contains a resolved IP for `github.com`, and `HasWarning -eq $false`.

## Rollback

```powershell
$wslCfgPath = Join-Path $env:USERPROFILE '.wslconfig'
$latestBak = Get-ChildItem -Path $env:USERPROFILE -Filter '.wslconfig.bak.*' -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1
if ($latestBak) {
    Copy-Item -Path $latestBak.FullName -Destination $wslCfgPath -Force
    # Warning: terminates all running WSL distros
    wsl.exe --shutdown
}
```

## Edge Cases

- On hosts *without* GSA/conflicting VPN clients, `networkingMode=mirrored` works and provides bilateral `localhost`; if keeping `mirrored`, set `$SwitchToNat = $false` so only `localhostForwarding` is removed.
- If dotfiles are managed by a template tool (e.g., `chezmoi`), update the source template rather than editing `$env:USERPROFILE\.wslconfig` directly, or the next apply will revert it.
