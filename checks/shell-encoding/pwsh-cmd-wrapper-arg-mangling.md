---
id: pwsh-cmd-wrapper-arg-mangling
category: shell-encoding
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (pwsh, cmd, azure-cli, npm, ai-agents)
symptoms:
  - "Passing a multi-line string (e.g., PR body, KQL/SQL query) from PowerShell to a `.cmd`-wrapped CLI (`az`, `npm`, `npx`, `gcloud`) silently drops everything after the first newline and drops all subsequent flags"
  - "URLs containing `&` passed to `.cmd` wrappers (e.g., `az rest --uri 'https://...?a=1&b=2'`) are split at `&` by `cmd.exe`, silently dropping query parameters such as `api-version`"
  - "Embedded double quotes inside arguments passed from `pwsh` (`PSNativeCommandArgumentPassing=Windows`) to `.cmd` wrappers are stripped or split the argument"
  - "Unit tests mocking the CLI with a `.ps1` stub pass, hiding all `.cmd` argument mangling in production"
---

# PowerShell to `.cmd` Wrapper Argument Mangling (Newlines, `&`, Quotes)

## Failure Mode

On Windows, major CLIs such as Azure CLI (`az.cmd`), npm/npx (`npm.cmd`), and Python/Node shims are batch scripts forwarding `%*` through `cmd.exe`. Even in PowerShell 7 (`$PSNativeCommandArgumentPassing = 'Windows'`, the default on Windows), `cmd.exe` parses arguments after PowerShell hands them off:
1. **Newlines (`\n`)**: `cmd.exe` truncates the command line at the first newline, silently dropping the rest of the value and any later flags (e.g., a multi-line PR description is saved as only `## Summary` with exit code `0`).
2. **Unescaped `&`**: `cmd.exe` treats `&` as a command separator, silently truncating URLs (`?a=1&b=2` becomes `?a=1`) and attempting to run `b=2` as a second command.
3. **Embedded `"`**: Unescaped double quotes cause `cmd.exe`/Python to split or strip tokens.

See also `windows-cmd-shim-batbadbut-injection` for the security/injection dimension of `.cmd` shims.

## Detect (Read-Only)

```powershell
$cmdWrappers = @('az', 'npm', 'npx', 'pnpm', 'yarn', 'gcloud' | ForEach-Object {
    $c = Get-Command $_ -ErrorAction SilentlyContinue
    if ($c -and $c.Source -match '\.(cmd|bat)$') {
        [PSCustomObject]@{ Tool = $_; Source = $c.Source }
    }
})

if ($cmdWrappers.Count -eq 0) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'No inspected CLIs resolve to .cmd/.bat' }
    return
}

# Advisory/repo check: if inside a git repo, check tracked .ps1 scripts for direct & in --uri or inline multiline args to .cmd CLIs
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
$riskyLines = @()
if ($LASTEXITCODE -eq 0 -and $repoRoot) {
    $psFiles = @(& git -C $repoRoot ls-files '*.ps1' '*.psm1' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ })
    if ($psFiles.Count -gt 0) {
        $riskyLines = @($psFiles | Select-String -Pattern '\baz\s+rest\b.*--uri\s+[''"].*&|\baz\s+repos\s+pr\b.*--description\s+\$' | Select-Object -First 10 Path, LineNumber, Line)
    }
}

[PSCustomObject]@{
    Status                         = if ($riskyLines.Count -gt 0) { 'FAIL' } else { 'PASS' }
    PSNativeCommandArgumentPassing = $PSNativeCommandArgumentPassing
    CmdWrappedClis                 = $cmdWrappers
    RiskyScriptCallSites           = $riskyLines
}
```

- **PASS**: `Status -eq 'PASS'` (`RiskyScriptCallSites.Count -eq 0`; callers pass multi-line bodies and `&`-bearing URLs via `@file` / `--url-parameters` / `stdin`).
- **FAIL**: `Status -eq 'FAIL'` (`RiskyScriptCallSites.Count -gt 0`).
- **SKIP**: `Status -eq 'SKIP'` (no inspected CLIs resolve to `.cmd`/`.bat`).

## Fix

1. **For multi-line bodies / JSON payloads**: Write the payload to a UTF-8 file (PS 5.1 & 7+ compatible) and pass `@file.json` (or `stdin`), never inline `--description $multilineBody` to a `.cmd` wrapper:

```powershell
$tmp = Join-Path ([IO.Path]::GetTempPath()) ('payload-' + [guid]::NewGuid().ToString('N') + '.json')
try {
    $json = @{ description = $body } | ConvertTo-Json -Depth 5
    [IO.File]::WriteAllText($tmp, $json, [Text.UTF8Encoding]::new($false))
    & az rest --method patch --uri $baseUri --url-parameters 'api-version=7.1' --body "@$tmp"
} finally {
    Remove-Item -LiteralPath $tmp -Force -ErrorAction SilentlyContinue
}
```

2. **For URLs with `&`**: Use separate parameter flags (e.g., `az rest --url-parameters a=1 b=2`) instead of embedding `&` in `--uri`.
3. **For tests**: Mock `.cmd`-wrapped CLIs with a real `.cmd` shim forwarding `%*` to a `.ps1`/Python stub — a bare `.ps1` stub bypasses `cmd.exe` and gives false green tests.

## Verify

```powershell
# Re-run repo scan to confirm zero risky direct-arg call sites remain
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
$psFiles = if ($LASTEXITCODE -eq 0 -and $repoRoot) { @(& git -C $repoRoot ls-files '*.ps1' '*.psm1' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ }) } else { @() }
$hits = if ($psFiles.Count -gt 0) { @($psFiles | Select-String -Pattern '\baz\s+rest\b.*--uri\s+[''"].*&|\baz\s+repos\s+pr\b.*--description\s+\$') } else { @() }
[PSCustomObject]@{ RemainingRiskyCallSites = $hits.Count }
```

- **PASS**: `RemainingRiskyCallSites -eq 0` and affected CLI calls deliver multi-line bodies / query parameters verbatim.

## Rollback

No persistent system state is modified; revert script changes with `git checkout -- <file>` if needed.

## Edge Cases

- Changing `$PSNativeCommandArgumentPassing = 'Standard'` fixes some native `.exe` quoting in PowerShell 7.3+, **not** `cmd.exe`'s own newline and `&` parsing inside `.cmd` batch wrappers. Always use `@file` / `stdin` / dedicated parameter flags for `.cmd` targets.
