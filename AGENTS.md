# AGENTS.md

Maintenance contract for AI agents editing the `windowbrain` repository.

## Repo Contract

- Pure Markdown, prompt-first repository. Do not add runtime code, build tooling, or package manifests unless explicitly requested.
- Target audience split:
  - `README.md` and `README.zh-CN.md`: human + AI agent entrypoints.
  - All other `.md` files (`GUIDE.md`, `AGENTS.md`, `checks/**`, `templates/**`): strictly AI-agent-facing (`en-US`). Optimize for unambiguous execution, exact commands, and deterministic pass/fail predicates. Omit human prose padding.

## Hard Constraints (Public & Portable)

- Public MIT repository. Never include:
  - Personal usernames, emails (except `LICENSE`), home paths, non-standard drive letters, or machine-specific paths.
  - Employer/corporate names, internal domains, private proxies, internal hostnames, or credentials.
- Every command in `checks/**` must discover targets dynamically (`$env:USERPROFILE`, `git`, registry queries, `Get-Command`, etc.) and work on any standard Windows 10/11 machine.

## Authoring a Check

1. Copy structure from `templates/CHECK_TEMPLATE.md` into `checks/<category>/<slug>.md`.
2. Valid categories:
   - `shell-encoding`
   - `paths-fs`
   - `perf-defender`
   - `processes`
   - `wsl-interop`
   - `toolchain-mcp`
3. Every check MUST define:
   - Valid YAML frontmatter (`id`, `category`, `severity`, `requires_admin`, `shell`, `reversible`, `applies_to`, `symptoms`).
   - `Detect (Read-Only)`: strictly zero mutations; explicit PASS / FAIL / SKIP predicates.
   - `Fix`: idempotent remediation commands.
   - `Verify`: independent post-fix check proving the condition is resolved.
   - `Rollback`: exact commands to revert the change.
4. Update `checks/INDEX.md` in the same change. Remove `.gitkeep` from a category directory once it contains its first check.
5. Do not add speculative or unverified Windows commands. Prefer built-in PowerShell (`pwsh` / `powershell.exe`) and native Windows/Git tooling.
