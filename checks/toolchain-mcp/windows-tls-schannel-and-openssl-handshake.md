---
id: windows-tls-schannel-and-openssl-handshake
category: toolchain-mcp
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (curl, node, npm, python, uv, corporate-network, ai-agents)
symptoms:
  - "`curl.exe` fails with `curl: (35) schannel: next InitializeSecurityContext failed: SEC_E_ILLEGAL_MESSAGE (0x80090326)` or `CRYPT_E_NO_REVOCATION_CHECK`"
  - "`npm` / `node` fails with `ERR_SSL_SSLV3_ALERT_HANDSHAKE_FAILURE` (`ssl3_read_bytes:sslv3 alert handshake failure`) or `UNABLE_TO_VERIFY_LEAF_SIGNATURE` while `gh` or browser succeeds"
  - "AI agents fail to fetch packages, wheels, or docs over HTTPS behind TLS-inspecting proxies or strict Schannel policies"
---

# Windows Schannel vs OpenSSL CA Store Split (`curl (35)` / Node TLS Failure)

## Failure Mode

Windows CLI tools split across two TLS stacks: `curl.exe` (Windows inbox) uses Windows **Schannel**, while Node.js/npm/Python/uv typically use bundled **OpenSSL/rustls** CA stores. Behind corporate TLS inspection or strict cipher/revocation policies, Schannel fails with `curl: (35) SEC_E_ILLEGAL_MESSAGE` / `CRYPT_E_NO_REVOCATION_CHECK` while Node/Python fail with `ERR_SSL_SSLV3_ALERT_HANDSHAKE_FAILURE` or `UNABLE_TO_VERIFY_LEAF_SIGNATURE` when custom root CAs are not exposed to OpenSSL runtimes.

## Detect (Read-Only)

```powershell
$curlOut = & curl.exe -fsSI --max-time 8 https://registry.npmjs.org/ 2>&1 | Out-String
$curlExit = $LASTEXITCODE
$curlSchannelErr = ($curlOut -match 'schannel:')

$missingCaVars = @('NODE_EXTRA_CA_CERTS', 'SSL_CERT_FILE', 'REQUESTS_CA_BUNDLE' | ForEach-Object {
    $v = [Environment]::GetEnvironmentVariable($_)
    if ($v -and -not (Test-Path -LiteralPath $v)) { [PSCustomObject]@{ Var = $_; Path = $v } }
})

$status = if ($curlSchannelErr -or $missingCaVars.Count -gt 0) { 'FAIL' } else { 'PASS' }

[PSCustomObject]@{
    Status            = $status
    CurlExitCode      = $curlExit
    CurlSchannelError = $curlSchannelErr
    MissingCaEnvFiles = $missingCaVars
    NodeExtraCaCerts  = $env:NODE_EXTRA_CA_CERTS
    UvNativeTls       = $env:UV_NATIVE_TLS
    GitSslBackend     = (& git config --global --get http.sslBackend 2>$null)
}
```

- **PASS**: `Status -eq 'PASS'` (`CurlSchannelError -eq $false` AND `MissingCaEnvFiles.Count -eq 0`).
- **FAIL**: `Status -eq 'FAIL'` (`CurlSchannelError -eq $true` OR any configured CA env var points to a non-existent file).
- **SKIP**: Offline / air-gapped machine with no internet access.

## Fix

1. **Git**: Configure Git on Windows to trust the Windows certificate store (record prior value first for exact rollback):

```powershell
$prior = (& git config --global --get http.sslBackend 2>$null)
$bakFile = Join-Path $env:TEMP 'windowbrain-git-sslbackend-prior.txt'
[IO.File]::WriteAllText($bakFile, $(if ($prior) { $prior } else { '' }), [Text.UTF8Encoding]::new($false))
git config --global http.sslBackend schannel
```

2. **Node.js (additive only)** & **`uv` / Python (native Windows TLS store)**:
   - Export Windows root CAs into a PEM file for **`NODE_EXTRA_CA_CERTS`** (Node treats this as *additive* to its built-in Mozilla roots).
   - For `uv` / Python, prefer native OS store integration (`UV_NATIVE_TLS=true`, or `pip install truststore`) rather than setting `SSL_CERT_FILE` / `REQUESTS_CA_BUNDLE` to a partial Windows export — because Windows populates `Cert:\LocalMachine\Root` lazily on demand, replacing OpenSSL's entire default bundle can break public HTTPS hosts.

```powershell
$caDir = Join-Path $env:LOCALAPPDATA 'certs'
if (-not (Test-Path $caDir)) { New-Item -ItemType Directory -Path $caDir -Force | Out-Null }
$pemPath = Join-Path $caDir 'windows-root-ca.pem'

$certs = Get-ChildItem Cert:\LocalMachine\Root | ForEach-Object {
    "-----BEGIN CERTIFICATE-----`n" +
    [Convert]::ToBase64String($_.RawData, 'InsertLineBreaks') +
    "`n-----END CERTIFICATE-----"
}
[IO.File]::WriteAllText($pemPath, (($certs -join "`n") + "`n"), [Text.Encoding]::ASCII)

[Environment]::SetEnvironmentVariable('NODE_EXTRA_CA_CERTS', $pemPath, 'User')
[Environment]::SetEnvironmentVariable('UV_NATIVE_TLS', 'true', 'User')
```

3. **`curl.exe` revocation failures only**: If `curl.exe` fails specifically with `CRYPT_E_NO_REVOCATION_CHECK` behind an inspecting proxy, pass `--ssl-no-revoke` (or add `ssl-no-revoke` to `$env:USERPROFILE\.curlrc`).

## Verify

```powershell
$pem = [Environment]::GetEnvironmentVariable('NODE_EXTRA_CA_CERTS', 'User')
$curlOut = & curl.exe -fsSI --max-time 8 https://registry.npmjs.org/ 2>&1 | Out-String
[PSCustomObject]@{
    CurlExit       = $LASTEXITCODE
    NodeExtraCaOk  = ($pem -and (Test-Path -LiteralPath $pem))
    UvNativeTls    = [Environment]::GetEnvironmentVariable('UV_NATIVE_TLS', 'User')
    GitSslBackend  = (& git config --global --get http.sslBackend 2>$null)
}
```

- **PASS**: `CurlExit -eq 0`, `NodeExtraCaOk -eq $true`, `GitSslBackend -eq 'schannel'`.

## Rollback

```powershell
$pemPath = Join-Path $env:LOCALAPPDATA 'certs\windows-root-ca.pem'
[Environment]::SetEnvironmentVariable('NODE_EXTRA_CA_CERTS', $null, 'User')
[Environment]::SetEnvironmentVariable('UV_NATIVE_TLS', $null, 'User')
if (Test-Path $pemPath) { Remove-Item -LiteralPath $pemPath -Force }

$bakFile = Join-Path $env:TEMP 'windowbrain-git-sslbackend-prior.txt'
if (Test-Path $bakFile) {
    $prior = (Get-Content -Raw -Path $bakFile).Trim()
    if ($prior) { git config --global http.sslBackend $prior } else { git config --global --unset http.sslBackend }
}
```

## Edge Cases

- `SEC_E_ILLEGAL_MESSAGE (0x80090326)` combined with Node `SSL alert number 40` (`handshake_failure`) can also indicate a middlebox/firewall blocking direct IP or SNI-mismatched connections to a public registry (e.g., when a corporate proxy/feed is required). Check `netsh winhttp show proxy` and `$env:HTTPS_PROXY` before assuming a missing CA cert.
- New User environment variables (`NODE_EXTRA_CA_CERTS`, `UV_NATIVE_TLS`) take effect in newly opened shells.
