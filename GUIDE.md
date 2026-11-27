# windowbrain Execution Protocol (AI Agents)

Master execution prompt for diagnosing and fixing Windows environment issues affecting AI coding agents.

## 0. Fetch & Scope

1. If you are not already reading a local checkout of `windowbrain`:
   - Preferred: `git clone --depth 1 https://github.com/wxxb789/windowbrain.git "$env:TEMP\windowbrain"` (or `/tmp/windowbrain` in Git Bash) and read files locally.
   - Fallback (no `git` or clone blocked): fetch `https://raw.githubusercontent.com/wxxb789/windowbrain/main/checks/INDEX.md`, then fetch only required `checks/<category>/<slug>.md` files from the same raw base URL.
2. Delete any temporary clone directory at the end of the session.

## 1. Mode Selection

Determine mode from the user's prompt:
- **FULL_AUDIT** (default when user asks to audit, check, scan, or "fix all findings"): run all checks listed in `checks/INDEX.md`.
- **SYMPTOM_TRIAGE** (when user pastes a specific error or symptom): match `symptoms` / `category` in `checks/INDEX.md` and load only relevant checks.

## 2. Host Probe (Read-Only)

Run one read-only probe before checks to establish runtime context:
```powershell
$isAdmin = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
[PSCustomObject]@{
  OS         = [System.Environment]::OSVersion.VersionString
  PSVersion  = $PSVersionTable.PSVersion.ToString()
  IsAdmin    = $isAdmin
  HasGit     = [bool](Get-Command git -ErrorAction SilentlyContinue)
  HasWsl     = [bool](Get-Command wsl -ErrorAction SilentlyContinue)
  Shell      = $env:ComSpec
}
```
Record `IsAdmin` to plan elevation in Phase 4.

## 3. Phase 1 — Read-Only Detection

- Load `checks/INDEX.md` and every selected check file.
- Execute ONLY each check's `Detect (Read-Only)` section.
- NEVER modify system state, files, profiles, registry keys, environment variables, or processes during Phase 1.
- Classify every check as:
  - `PASS`: meets expected state.
  - `FAIL`: defect confirmed with concrete observed output.
  - `SKIP`: prerequisite missing or not applicable (state reason).

## 4. Phase 2 — Report & Single Confirmation Gate

If zero checks are `FAIL`, report the clean audit summary, clean up temp files, and stop.

Otherwise, present ONE consolidated findings table:

| # | Check ID | Severity | Observed | Proposed Fix | Requires Admin | Rollback Summary |
|---|---|---|---|---|---|---|

Follow the table with ONE confirmation prompt offering:
- `all`: fix every `FAIL` item (non-admin + admin)
- `safe`: fix only `Requires Admin = no` items
- `<numbers>` (e.g. `1,3`): fix selected items only
- `none`: stop without changes

STOP and wait for user confirmation before executing any `Fix` step.
Exception: if the user already confirmed specific items in the current turn after reviewing the exact findings table, proceed directly to Phase 3.

## 5. Phase 3 — Automated Remediation

Execute all user-confirmed fixes automatically:

1. **Non-admin fixes (`requires_admin: false`)**:
   - Run each check's `Fix` commands directly in order.
   - Capture stdout, stderr, and exit code.
2. **Admin fixes (`requires_admin: true`)**:
   - If `IsAdmin == $true`: run directly in the current shell.
   - If `IsAdmin == $false`: batch ALL confirmed admin fixes into a single temporary PowerShell script (`$env:TEMP\windowbrain-admin-fix.ps1`) with `$ErrorActionPreference = 'Stop'` and explicit per-check log output to `$env:TEMP\windowbrain-admin-fix.log`.
   - Show the script path and contents, then trigger ONE UAC prompt:
     ```powershell
     Start-Process powershell.exe -Verb RunAs -Wait -ArgumentList @(
       '-NoProfile',
       '-ExecutionPolicy', 'Bypass',
       '-File', "$env:TEMP\windowbrain-admin-fix.ps1"
     )
     ```
   - Read `$env:TEMP\windowbrain-admin-fix.log` after exit. If the user cancels UAC or a command fails, record the exact error without silently ignoring it.
   - Remove the temporary script and log after reading.

## 6. Phase 4 — Verification & Final Report

1. Re-run `Verify` for every check touched in Phase 3.
2. Note any fix requiring a new shell session or reboot to take full effect.
3. Output final status table:

| Check ID | Action | Verify Result | Notes / Next Step |
|---|---|---|---|

4. Clean up any temporary clone or helper files created during the run.
