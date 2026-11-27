---
id: pwsh-single-element-array-flattening
category: shell-encoding
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (pwsh, ai-agents)
symptoms:
  - "PowerShell script doing `@(@('old', 'new'))` replacement loops over a single pair silently corrupts files by replacing individual characters (`pair[0]` = first char, `pair[1]` = second char)"
  - "Multi-line operator command blocks pasted into PowerShell run line-by-line and continue executing destructive later lines even after an earlier line fails"
---

# PowerShell `@(@(...))` Single-Element Array Flattening & Unwrapped Paste Blocks

## Failure Mode

PowerShell's `@(...)` subexpression operator unrolls single-element nested arrays: `@(@('old', 'new'))` evaluates to the flat 2-string array `@('old', 'new')`, **not** a 1-element array of pairs. When an AI agent writes `foreach ($pair in $pairs) { $text = $text.Replace($pair[0], $pair[1]) }`, `$pair` iterates over the strings `'old'` and `'new'`, so `$pair[0]` and `$pair[1]` index single characters (`'o'` -> `'l'`, `'n'` -> `'e'`) and silently scramble the entire file. Separately, pasting multi-line PowerShell snippets into a terminal without wrapping in `& { $ErrorActionPreference = 'Stop'; ... }` executes lines individually so a failure on line 1 does not stop lines 2..N.

## Detect (Read-Only)

```powershell
$repoRoot = (& git rev-parse --show-toplevel 2>$null)
if ($LASTEXITCODE -ne 0 -or -not $repoRoot) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'Run inside a Git repository to audit PowerShell scripts' }
    return
}

$psFiles = @(& git -C $repoRoot ls-files '*.ps1' '*.psm1' | ForEach-Object { Join-Path $repoRoot $_ } | Where-Object { Test-Path $_ })
if ($psFiles.Count -eq 0) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'No tracked PowerShell files in repository' }
    return
}

# Match single-element nested array `@(@(...))` without flagging multi-element `@(@(...), @(...))`
$hits = @($psFiles | Select-String -Pattern '@\(\s*@\([^()]*\)\s*\)')

[PSCustomObject]@{
    Status                 = if ($hits.Count -gt 0) { 'FAIL' } else { 'PASS' }
    NestedArrayLiteralHits = $hits.Count
    Matches                = @($hits | Select-Object -First 10 Path, LineNumber, Line)
}
```

- **PASS**: `Status -eq 'PASS'` (`NestedArrayLiteralHits -eq 0`).
- **FAIL**: `Status -eq 'FAIL'` (`NestedArrayLiteralHits -gt 0`).
- **SKIP**: `Status -eq 'SKIP'` (not inside a Git repository or no tracked `.ps1`/`.psm1` files).

## Fix

Use an `[ordered]@{}` hashtable (or explicit unary comma `,@(...)`) for replacement pairs, and wrap pasted operator blocks in `& { $ErrorActionPreference = 'Stop'; ... }`:

```powershell
$replacements = [ordered]@{
    'old-token' = 'new-token'
}
foreach ($entry in $replacements.GetEnumerator()) {
    if (-not $content.Contains($entry.Key)) {
        throw "Expected token not found: $($entry.Key)"
    }
    $content = $content.Replace($entry.Key, $entry.Value)
}
```

## Verify

```powershell
[PSCustomObject]@{
    FlattenedBugCount = (@(@('old', 'new'))).Count
    OrderedSafeCount  = ([ordered]@{ old = 'new' }).Count
}
```

- **PASS**: `OrderedSafeCount -eq 1` and re-running `Detect (Read-Only)` reports `NestedArrayLiteralHits -eq 0`.

## Rollback

No system state changed; restore edited files from Git (`git checkout -- <file>`) if corruption occurred before the fix.

## Edge Cases

- Prefer dedicated file-editing tools over inline PowerShell string-replacement scripts for untracked or intent-to-add files with no committed backup.
