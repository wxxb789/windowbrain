---
id: gcm-empty-user-binding-popup-storm
category: toolchain-mcp
severity: medium
requires_admin: false
shell: powershell
reversible: true
applies_to: Windows 10/11 (git, gcm, ai-agents)
symptoms:
  - "Repeated Windows account-picker / Git Credential Manager GUI popups interrupt the desktop every few minutes while AI agents or orchestrators run background `git fetch`"
  - "Popups keep reappearing even though `gh` / `az` CLI sessions are already logged in"
  - "`git config --show-origin --get-regexp 'credential.*username'` shows an empty `username` binding for a host"
---

# Empty GCM `credential.<url>.username` Triggering Background Popup Storms

## Failure Mode

When Git Credential Manager (GCM) is configured as `credential.helper=manager` on Windows and a host-scoped `[credential "<url>"]` block has an empty `username =` binding (or duplicate empty bindings across `$env:USERPROFILE\.gitconfig` and an `includeIf` file), GCM cannot silently select the Windows account and pops up an interactive Windows account-confirmation dialog. When an AI coding agent or workspace orchestrator runs background `git fetch` loops, this triggers a recurring popup storm and stalls headless Git operations.

## Detect (Read-Only)

```powershell
if (-not (Get-Command git -ErrorAction SilentlyContinue)) {
    [PSCustomObject]@{ Status = 'SKIP'; Reason = 'git not installed' }
    return
}

$helpers = @(& git config --list --show-origin 2>$null | Where-Object { $_ -match 'credential\.' })
$emptyUsers = @($helpers | Where-Object { $_ -match 'credential\..*\.username=$' })

[PSCustomObject]@{
    Status                = if ($emptyUsers.Count -gt 0) { 'FAIL' } else { 'PASS' }
    EmptyUsernameBindings = $emptyUsers
}
```

- **PASS**: `Status -eq 'PASS'` (`EmptyUsernameBindings.Count -eq 0`).
- **FAIL**: `Status -eq 'FAIL'` (`EmptyUsernameBindings.Count -gt 0`).
- **SKIP**: `Status -eq 'SKIP'` (`git` is not installed).

## Fix

Edit the exact config file reported by `--show-origin` (an empty binding in an included file overrides `--global`), setting the explicit account username for that host:

```powershell
# Replace with the exact file path from EmptyUsernameBindings, host URL, and account username:
$ConfigFile = Join-Path $env:USERPROFILE '.gitconfig'
$HostUrl    = 'https://your-git-host'
$Account    = 'user@example.com'

git config --file $ConfigFile "credential.$HostUrl.username" $Account
```

In headless agent / CI shells, also suppress interactive GUI prompts so missing auth fails fast instead of hanging:

```powershell
$env:GIT_TERMINAL_PROMPT = '0'
$env:GCM_INTERACTIVE = 'never'
```

## Verify

```powershell
$emptyUsers = @(& git config --list --show-origin 2>$null | Where-Object { $_ -match 'credential\..*\.username=$' })
[PSCustomObject]@{ RemainingEmptyBindings = $emptyUsers.Count }
```

- **PASS**: `RemainingEmptyBindings -eq 0` (and in a repo using that host, `GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=never git ls-remote --heads origin` exits `0` with no GUI popup).

## Rollback

```powershell
$ConfigFile = Join-Path $env:USERPROFILE '.gitconfig'
$HostUrl    = 'https://your-git-host'
git config --file $ConfigFile --unset "credential.$HostUrl.username"
```

## Edge Cases

- If a host is intended to authenticate via `gh auth git-credential` (e.g., GitHub) rather than GCM, clear the inherited helper chain first (`credential.<url>.helper =` followed by `credential.<url>.helper = !gh auth git-credential`).
- If dotfiles are managed by a template tool (e.g., `chezmoi`), update the source template rather than editing the generated `.gitconfig` directly.
