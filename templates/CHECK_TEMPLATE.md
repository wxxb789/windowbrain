---
id: <category-short-slug>
category: shell-encoding | paths-fs | perf-defender | processes | wsl-interop | toolchain-mcp
severity: high | medium | low
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11
symptoms:
  - <exact error string or observable failure>
---

# <Check Title>

## Failure Mode

Concise technical description of how this Windows quirk breaks AI coding agents.

## Detect (Read-Only)

```powershell
# Strictly read-only commands. No mutations.
```

- **PASS**: exact condition.
- **FAIL**: exact condition.
- **SKIP**: exact condition (if applicable).

## Fix

```powershell
# Idempotent remediation commands.
```

## Verify

```powershell
# Independent post-fix verification commands.
```

- **PASS**: exact post-fix condition.

## Rollback

```powershell
# Exact commands to revert the fix.
```

## Edge Cases

- Restart/reboot/new-shell requirements, or known interactions with specific tools.
