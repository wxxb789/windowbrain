---
id: pwsh-get-content-tail-spin
category: shell-encoding
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (PowerShell 5.1 & PowerShell 7+)
symptoms:
  - pwsh.exe or powershell.exe pins 1+ CPU cores indefinitely while reading log/NDJSON files
  - Orphaned pwsh -NonInteractive processes left behind with high accumulated CPU and no children
  - Get-Content -Tail hangs for minutes on a few-megabyte JSONL/NDJSON file
---

# PowerShell `Get-Content -Tail` O(n²) Spin on Long Lines

## Failure Mode

PowerShell's `Get-Content -Tail N` scans backwards character-by-character and prepends each character into a `StringBuilder` via `Insert(0, ...)`, making runtime **O(n²) per line**. On files with few, very long lines (e.g., NDJSON/JSONL agent event logs with 200 KB–3 MB lines), `Get-Content -Tail 3` pins 1–1.6 CPU cores for 15–20+ minutes. When the calling agent times out and moves on, the `pwsh` process remains orphaned and saturates CPU.

## Detect (Read-Only)

Check for currently spinning orphaned `pwsh` / `powershell` processes executing `Get-Content` with `-Tail`, and scan agent scripts/profiles if requested:

```powershell
Get-CimInstance Win32_Process |
  Where-Object { $_.Name -match '^(pwsh|powershell)\.exe$' -and $_.CommandLine -match 'Get-Content.*-Tail' } |
  Select-Object ProcessId, ParentProcessId, CreationDate, CommandLine
```

- **PASS**: zero spinning `pwsh`/`powershell` processes stuck in `Get-Content -Tail`.
- **FAIL**: one or more `pwsh`/`powershell` processes are stuck running `Get-Content -Tail`.

## Fix

1. **Terminate confirmed stuck `Get-Content -Tail` processes** (only PIDs matched in `Detect`):
```powershell
Stop-Process -Id <PID> -Force
```

2. **Agent runtime rule (replace `Get-Content -Tail` on logs/NDJSON)**:
Never use `Get-Content -Tail` on NDJSON, JSONL, minified files, or unknown-line-length logs. Seek a `FileStream` from the end and decode the trailing bytes instead:
```powershell
function Get-FileTailBytes {
  param([string]$Path, [int]$Bytes = 16384)
  $fs = [IO.File]::Open($Path, [IO.FileMode]::Open, [IO.FileAccess]::Read, [IO.FileShare]::ReadWrite)
  try {
    $start = [Math]::Max(0L, $fs.Length - $Bytes)
    $fs.Seek($start, [IO.SeekOrigin]::Begin) | Out-Null
    $buf = New-Object byte[] ($fs.Length - $start)
    $read = $fs.Read($buf, 0, $buf.Length)
    [Text.Encoding]::UTF8.GetString($buf, 0, $read)
  } finally {
    $fs.Dispose()
  }
}
```

## Verify

Re-run the `Detect (Read-Only)` query and confirm zero stuck `Get-Content -Tail` processes remain.

## Rollback

No persistent system configuration is modified; process termination is not reversible.

## Edge Cases

- `FileShare::ReadWrite` in `Get-FileTailBytes` avoids `IOException` sharing violations when another process is actively appending to the log file.
- If the tail slice cuts mid-UTF-8 multibyte sequence, drop the partial first line (`$text.Substring($text.IndexOf("`n") + 1)`).
