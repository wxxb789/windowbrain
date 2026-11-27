---
id: devdrive-defender-policy-and-cpu-caps
category: perf-defender
severity: high
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (heavy Rust/C++/Node builds and multi-agent workloads)
symptoms:
  - Machine UI freezes or CPU hits 100% with high Processor Queue Length during parallel agent builds
  - High % Privileged Time inside cargo/rustc/link.exe/node despite MsMpEng looking modest in Task Manager
  - Trusted Dev Drive still suffers slow Defender OnOpen real-time scans
  - Set-MpPreference -PerformanceModeStatus Enabled succeeds silently but stays Disabled
---

# Defender Dev Drive Policy Trap & Parallel Build CPU Caps

## Failure Mode

1. **Defender filter-driver cost is billed to the caller**: `WdFilter` / `MsSecFlt` scan every freshly written `.exe`/`.dll` on first open (3–6 s per newly linked test binary). That time appears as kernel time (`% Privileged Time`) inside `cargo`/`rustc`/`link.exe`, hiding AV overhead in simple CPU tables.
2. **Dev Drive trust != async performance mode**: Even on a trusted ReFS Dev Drive, Defender's `PerformanceModeStatus` uses an inverted enum (`0` = Enabled, `1` = Disabled). On enterprise/managed Windows hosts, Group Policy silently overrides `Set-MpPreference -PerformanceModeStatus Enabled`—the command exits 0, yet `PerformanceModeStatus` reads back as `1` and scans stay synchronous (`OnOpen`, `Not skipped`).
3. **Uncapped parallel builds starve UI and agents**: `cargo` defaults `build.jobs` to all logical CPUs; two concurrent agent worktrees easily spawn 30+ compiler/linker processes, spiking `Processor Queue Length` far above core count.

## Detect (Read-Only)

```powershell
$cores = [Environment]::ProcessorCount
$mp = Get-MpPreference -ErrorAction SilentlyContinue
$perfMode = if ($null -ne $mp) { $mp.PerformanceModeStatus } else { 'UNAVAILABLE' }
$perfLabel = switch ($perfMode) {
  0 { 'Enabled (async on trusted Dev Drive)' }
  1 { 'Disabled (synchronous OnOpen scans)' }
  default { "$perfMode" }
}

$cargoHome = if ($env:CARGO_HOME) { $env:CARGO_HOME } else { Join-Path $env:USERPROFILE '.cargo' }
$cargoCfg = Join-Path $cargoHome 'config.toml'
$hasJobsCap = (Test-Path $cargoCfg) -and (Select-String -Path $cargoCfg -Pattern '^\s*jobs\s*=' -Quiet)

$counters = Get-Counter '\Processor(_Total)\% Processor Time','\Processor(_Total)\% Privileged Time','\System\Processor Queue Length' -SampleInterval 1 -MaxSamples 2 -ErrorAction SilentlyContinue

[PSCustomObject]@{
  LogicalCores             = $cores
  RecommendedCargoJobsCap  = [Math]::Max(2, [Math]::Floor($cores / 2))
  DefenderPerfModeRaw      = $perfMode
  DefenderPerfModeMeaning  = $perfLabel
  CargoInstalled           = [bool](Get-Command cargo -ErrorAction SilentlyContinue)
  CargoConfigPath          = $cargoCfg
  CargoHasJobsCap          = $hasJobsCap
}
```

- **PASS**:
  - If `cargo` is installed: `CargoHasJobsCap -eq $true`; AND
  - `DefenderPerfModeRaw -eq 0` (or documented as GPO-managed / no Dev Drive).
- **FAIL**: `cargo` is installed without a `[build] jobs` cap, OR `DefenderPerfModeRaw -eq 1` on a host with a Dev Drive.

## Fix

1. **Non-admin: cap global Cargo parallelism** to ~50% of logical cores so UI and concurrent agents stay responsive:
```powershell
$cores = [Environment]::ProcessorCount
$cap = [Math]::Max(2, [Math]::Floor($cores / 2))
$cargoHome = if ($env:CARGO_HOME) { $env:CARGO_HOME } else { Join-Path $env:USERPROFILE '.cargo' }
$cargoCfg = Join-Path $cargoHome 'config.toml'
if (-not (Test-Path $cargoHome)) { New-Item -ItemType Directory -Path $cargoHome -Force | Out-Null }
if (-not (Test-Path $cargoCfg)) { New-Item -ItemType File -Path $cargoCfg -Force | Out-Null }
if (-not (Select-String -Path $cargoCfg -Pattern '^\s*jobs\s*=' -Quiet)) {
  Add-Content -Path $cargoCfg -Value "`r`n# >>> windowbrain:cargo-jobs >>>`r`n[build]`r`njobs = $cap`r`n# <<< windowbrain:cargo-jobs <<<" -Encoding UTF8
}
```

2. **Non-admin: launch heavy builds at `BelowNormal` priority** (Windows inherits priority class downward to child processes, keeping UI responsive):
```powershell
$p = Start-Process -FilePath 'cargo' -ArgumentList 'test' -PassThru -NoNewWindow
$p.PriorityClass = 'BelowNormal'
$p.WaitForExit()
```

3. **Admin (optional, Dev Drive hosts only)**: attempt enabling Defender Performance Mode and IMMEDIATELY read back to detect silent GPO override:
```powershell
Set-MpPreference -PerformanceModeStatus Enabled
$after = (Get-MpPreference).PerformanceModeStatus
if ($after -ne 0) {
  Write-Warning "PerformanceModeStatus remained $after (Disabled). Managed Group Policy overrides local settings; do not retry—escalate to IT or reduce new .exe churn."
}
```

## Verify

- Re-run `Detect (Read-Only)` and confirm `CargoHasJobsCap -eq $true`.
- If Admin step 3 ran, verify `DefenderPerfModeRaw -eq 0` (or record GPO override clearly).
- For deep Defender scan attribution (Admin), record 60s of ETW data:
  `New-MpPerformanceRecording -RecordTo $env:TEMP\mp.etl -Seconds 60` followed by `Get-MpPerformanceReport -Path $env:TEMP\mp.etl -TopFiles 15 -TopExtensions 10 -TopProcesses 10 -TopScans 10`.

## Rollback

Remove the `# >>> windowbrain:cargo-jobs >>>` block from `$cargoCfg`.

## Edge Cases

- Avoid `Get-Counter '\Process(*)\...'` for per-process CPU triage on Windows: it aborts the entire sample whenever any short-lived process exits mid-sample. Use `Get-CimInstance Win32_PerfFormattedData_PerfProc_Process` joined with `Win32_Process` instead.
- If Defender scans remain synchronous under GPO, reduce fresh `.exe` churn by consolidating many small Rust integration test binaries (`tests/*.rs`) into one binary (`tests/it/main.rs`) and reusing warm worktrees.
