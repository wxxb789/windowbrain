---
id: local-agent-proxy-public-firewall-exposure
category: toolchain-mcp
severity: high
requires_admin: true
shell: powershell
reversible: true
applies_to: Windows 10/11 (bun, node, python, deno, local-proxy, mcp, ai-agents)
symptoms:
  - "Local AI API proxy or MCP HTTP server binds to `0.0.0.0` / `::` (all interfaces) instead of `127.0.0.1`"
  - "Windows Defender Firewall has an `Allow` inbound rule for `bun.exe` / `node.exe` / `python.exe` / `deno.exe` on `Public`, `Domain`, or `Any` profiles"
  - "Unauthenticated local agent endpoints or token routes become reachable from the LAN / corporate network"
---

# Local Agent Proxy / MCP Server Exposed via Wildcard Bind & Windows Firewall

## Failure Mode

When developers run local AI agent proxies, MCP HTTP servers, or dev servers on Windows using `bun.exe`, `node.exe`, `python.exe`, or `deno.exe`, clicking "Allow" once on the Windows Defender Firewall popup permanently creates a broad inbound `Allow` rule for that entire runtime binary (often including `Public`/`Domain` profiles). If any local proxy or MCP server binds to `0.0.0.0` / `::` instead of `127.0.0.1` (and lacks authentication or uses permissive CORS), its endpoints and credentials are silently exposed to the entire network.

## Detect (Read-Only)

```powershell
$runtimeRegex = '(bun|node|python|python3|deno)\.exe$'

# 1. Find listening TCP sockets bound to 0.0.0.0 or :: on non-privileged ports for agent runtimes
$wildcardListeners = @(Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
    Where-Object { $_.LocalAddress -in @('0.0.0.0', '::') -and $_.LocalPort -ge 1024 } |
    ForEach-Object {
        $p = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
        if ($p -and $p.ProcessName -match '^(node|bun|python|python3|uv|deno)$') {
            [PSCustomObject]@{
                Process      = $p.ProcessName
                PID          = $_.OwningProcess
                LocalAddress = $_.LocalAddress
                LocalPort    = $_.LocalPort
            }
        }
    })

# 2. Check Windows Firewall inbound Allow rules for agent runtimes on Public/Domain/Any profiles
$fwRules = @(Get-NetFirewallApplicationFilter -ErrorAction SilentlyContinue |
    Where-Object { $_.Program -match $runtimeRegex } |
    ForEach-Object {
        $rule = $_ | Get-NetFirewallRule -ErrorAction SilentlyContinue
        if ($rule -and $rule.Enabled -eq 'True' -and $rule.Direction -eq 'Inbound' -and $rule.Action -eq 'Allow' -and ($rule.Profile.ToString() -match 'Public|Domain|Any')) {
            [PSCustomObject]@{
                Name     = $rule.Name
                RuleName = $rule.DisplayName
                Profile  = $rule.Profile.ToString()
                Program  = $_.Program
            }
        }
    })

$status = if ($wildcardListeners.Count -gt 0 -or $fwRules.Count -gt 0) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status                   = $status
    WildcardAgentListeners   = $wildcardListeners
    RuntimeInboundAllowRules = $fwRules
}
```

- **PASS**: `Status -eq 'PASS'` (`WildcardAgentListeners.Count -eq 0` AND `RuntimeInboundAllowRules.Count -eq 0`).
- **FAIL**: `Status -eq 'FAIL'` (any agent runtime listens on `0.0.0.0`/`::` OR any enabled Windows Firewall inbound `Allow` rule permits `bun`/`node`/`python`/`deno` on `Public`/`Domain`/`Any`).
- **SKIP**: Not applicable (runs on all Windows hosts).

## Fix

1. **Non-admin (application config)**: Bind every local AI proxy and MCP HTTP server explicitly to loopback (`--host 127.0.0.1` / `hostname: "127.0.0.1"`).
2. **Admin**: Record the exact unique rule `Name` IDs to a rollback manifest, then disable the overly broad `Public`/`Domain`/`Any` inbound `Allow` rules for those runtimes:

```powershell
$ErrorActionPreference = 'Stop'
$runtimeRegex = '(bun|node|python|python3|deno)\.exe$'
$manifest = Join-Path $env:TEMP 'windowbrain-fw-disabled.txt'

$rules = @(Get-NetFirewallApplicationFilter -ErrorAction SilentlyContinue |
    Where-Object { $_.Program -match $runtimeRegex } |
    Get-NetFirewallRule -ErrorAction SilentlyContinue |
    Where-Object {
        $_.Enabled -eq 'True' -and
        $_.Direction -eq 'Inbound' -and
        $_.Action -eq 'Allow' -and
        ($_.Profile.ToString() -match 'Public|Domain|Any')
    })

if ($rules.Count -gt 0) {
    $names = @($rules | Select-Object -ExpandProperty Name -Unique)
    [IO.File]::WriteAllText($manifest, (($names -join "`n") + "`n"), [Text.UTF8Encoding]::new($false))
    $rules | Disable-NetFirewallRule
    Write-Output "Disabled $($rules.Count) rule(s); saved rollback manifest to $manifest"
} else {
    Write-Output 'No matching inbound Allow rules to disable.'
}
```

## Verify

```powershell
$runtimeRegex = '(bun|node|python|python3|deno)\.exe$'
$remaining = @(Get-NetFirewallApplicationFilter -ErrorAction SilentlyContinue |
    Where-Object { $_.Program -match $runtimeRegex } |
    Get-NetFirewallRule -ErrorAction SilentlyContinue |
    Where-Object {
        $_.Enabled -eq 'True' -and
        $_.Direction -eq 'Inbound' -and
        $_.Action -eq 'Allow' -and
        ($_.Profile.ToString() -match 'Public|Domain|Any')
    })
[PSCustomObject]@{ RemainingInboundAllowRules = $remaining.Count }
```

- **PASS**: `RemainingInboundAllowRules -eq 0` (and re-running `Detect (Read-Only)` reports `WildcardAgentListeners.Count -eq 0` for local proxies/MCP servers).

## Rollback

```powershell
$manifest = Join-Path $env:TEMP 'windowbrain-fw-disabled.txt'
if (Test-Path $manifest) {
    Get-Content -Path $manifest | Where-Object { $_ } | ForEach-Object {
        Enable-NetFirewallRule -Name $_
    }
}
```

## Edge Cases

- Loopback traffic (`127.0.0.1` / `::1`) never requires a Windows Firewall inbound `Allow` rule; disabling inbound rules for `bun.exe`/`node.exe`/`python.exe`/`deno.exe` will **not** break local loopback proxies or MCP servers.
- Disabling runtime-level firewall rules affects all LAN-exposed dev servers running under that runtime binary; if a specific LAN dev server is intentional, scope a rule to its specific port on the `Private` profile only.
- GPO-managed firewall rules cannot be modified locally; if `Disable-NetFirewallRule` reports access denied on a GPO rule, bind the service strictly to `127.0.0.1`.
