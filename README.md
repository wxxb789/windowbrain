# windowbrain

> **For AI Agents**: Do NOT stop at this README. Fetch the full repository via `git clone --depth 1 https://github.com/wxxb789/windowbrain.git "$env:TEMP\windowbrain"` (or fall back to `https://raw.githubusercontent.com/wxxb789/windowbrain/main/GUIDE.md` and `checks/INDEX.md` if `git` is unavailable), then follow [`GUIDE.md`](GUIDE.md) strictly. Run read-only checks first, present one consolidated findings table, wait for user confirmation, then apply and verify all confirmed fixes automatically.

[中文文档 (README.zh-CN.md)](README.zh-CN.md)

**windowbrain** is a pure-Markdown, AI-native playbook that helps AI coding agents diagnose and fix weird Windows environment pitfalls—encoding bugs, path limits, Defender stalls, orphan processes, WSL2 quirks, and broken toolchains.

No CLI to install. No runtime dependencies. Just point your AI agent at this repo.

## Usage

Paste this into any AI coding agent running on Windows (Claude Code, Codex CLI, OpenCode, Cursor, Copilot CLI, etc.):

```text
read and take action by this guide: https://github.com/wxxb789/windowbrain
and fix all findings on this windows machine
```

Or ask it to troubleshoot a specific error:

```text
read https://github.com/wxxb789/windowbrain and diagnose this error on my Windows machine: <paste error>
```

## How It Works

1. **Read-Only Audit**: Your agent probes your Windows machine without changing anything and runs every relevant check in [`checks/INDEX.md`](checks/INDEX.md).
2. **Single Confirmation Gate**: You get one table listing every finding, severity, proposed fix, admin requirement, and rollback command.
3. **Automated Fix & Verify**: Once you confirm (`all`, `safe`, or specific item numbers), the agent applies all non-admin fixes directly, batches any admin fixes into a single UAC prompt, and re-verifies every change.

## Coverage Categories

| Category | Directory | Examples |
|---|---|---|
| Shell & Encoding | `checks/shell-encoding/` | UTF-8 console/Python/Git encoding, PowerShell vs Bash quoting, CRLF, ConPTY |
| Paths & Filesystem | `checks/paths-fs/` | `MAX_PATH` / `core.longpaths`, symlinks, `EBUSY` locks, case sensitivity |
| Performance & Defender | `checks/perf-defender/` | Defender real-time scan overhead, Dev Drive, CPU contention |
| Processes & Signals | `checks/processes/` | Orphan child processes, lingering ports, zombie MCP/node/python processes |
| WSL2 Interop | `checks/wsl-interop/` | `wsl.exe` quoting traps, `/mnt/c` I/O latency, proxy/localhost forwarding |
| Toolchain & MCP | `checks/toolchain-mcp/` | `npx`/`uvx` shims, cross-shell `PATH` drift, UAC boundaries, corporate CA/SSL |

## Repository Structure

```text
windowbrain/
├── README.md                 # Human + AI entrypoint (EN)
├── README.zh-CN.md           # Human + AI entrypoint (ZH)
├── GUIDE.md                  # Master execution protocol (AI-agent-facing)
├── AGENTS.md                 # Repo authoring & sanitization rules (AI-agent-facing)
├── templates/
│   └── CHECK_TEMPLATE.md     # Standard schema for adding a check
└── checks/
    ├── INDEX.md              # Machine-readable check registry
    ├── shell-encoding/
    ├── paths-fs/
    ├── perf-defender/
    ├── processes/
    ├── wsl-interop/
    └── toolchain-mcp/
```

## Contributing a Pitfall

All files other than `README*.md` are written in `en-US` and optimized directly for AI agent execution:

1. Copy [`templates/CHECK_TEMPLATE.md`](templates/CHECK_TEMPLATE.md) into `checks/<category>/<slug>.md`.
2. Keep commands generic and portable (no hardcoded usernames, drive letters, or private paths).
3. Register the new check in [`checks/INDEX.md`](checks/INDEX.md).

## License

[MIT](LICENSE)
