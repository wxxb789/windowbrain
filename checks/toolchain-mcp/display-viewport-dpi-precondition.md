---
id: display-viewport-dpi-precondition
category: toolchain-mcp
severity: low
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (desktop UI / Electron / Tauri / WebView2 / Playwright tests)
symptoms:
  - Desktop UI or self-test suite fails with "viewport precondition" or clamped window dimensions
  - Requested 1280x800 or 1440x844 window silently clamps to smaller height/width on portrait or RDP display
  - UI tests pass on landscape monitor but fail locally over RDP or high-DPI (150%+) laptop screen
---

# Windows Display Orientation / DPI / RDP Work-Area Viewport Clamping

## Failure Mode

Windows enforces window size clamping to the primary monitor's **logical working area** (physical resolution divided by DPI scaling, minus taskbar). When a developer's primary display is rotated to portrait (e.g., `1152x2048`), scaled to 150%+ DPI, or connected via a small RDP session (`1280x800` -> work area `~1280x760`), desktop UI / WebView2 / Tauri / Electron tests requesting `1280x800` or `1440x844` fail with viewport precondition errors—often deep into a 10–25 minute test run. Agents then waste turns debugging non-existent UI layout bugs.

## Detect (Read-Only)

Run BEFORE executing local desktop UI or visual-regression tests:

```powershell
Add-Type -AssemblyName System.Windows.Forms
$screen = [System.Windows.Forms.Screen]::PrimaryScreen
[PSCustomObject]@{
  BoundsWidth       = $screen.Bounds.Width
  BoundsHeight      = $screen.Bounds.Height
  WorkAreaWidth     = $screen.WorkingArea.Width
  WorkAreaHeight    = $screen.WorkingArea.Height
  IsLandscape       = ($screen.Bounds.Width -ge $screen.Bounds.Height)
  IsRdpSession      = [System.Windows.Forms.SystemInformation]::TerminalServerSession
}
```

- **SKIP**: user is not running desktop GUI / browser-window viewport tests.
- **PASS**: `IsLandscape -eq $true` AND `WorkAreaWidth` / `WorkAreaHeight` meet or exceed the test suite's required window dimensions.
- **FAIL**: `IsLandscape -eq $false` OR `WorkAreaWidth`/`WorkAreaHeight` is smaller than the required test viewport.

## Fix

Do NOT weaken or edit the repository's UI test gates to fit a cramped local display:

1. **Landscape / RDP resize (human action)**: Ask the user to switch the primary display to landscape, lower DPI scaling temporarily, or resize the RDP client window so `WorkingArea` exceeds the required viewport.
2. **WebView2 / Chromium headless or CDP override (when supported by the test harness)**:
   For WebView2 apps that expose DevTools via `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS=--remote-debugging-port=<port>`, use CDP `Emulation.setDeviceMetricsOverride` to verify layout at the target viewport when the physical monitor cannot fit it.

## Verify

Re-run `Detect (Read-Only)` and confirm `IsLandscape -eq $true` and `WorkAreaWidth` / `WorkAreaHeight` exceed the test suite's minimum viewport.

## Rollback

No system settings are modified by this check.

## Edge Cases

- `System.Windows.Forms.Screen` reports DPI-virtualized coordinates unless the process is DPI-aware; compare against the logical CSS pixels requested by the UI framework.
- RDP sessions dynamically change `WorkingArea` when the user resizes or minimizes the RDP client window mid-run.
