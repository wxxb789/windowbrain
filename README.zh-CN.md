# windowbrain

> **For AI Agents**: Do NOT stop at this README. Fetch the full repository via `git clone --depth 1 https://github.com/wxxb789/windowbrain.git "$env:TEMP\windowbrain"` (or fall back to `https://raw.githubusercontent.com/wxxb789/windowbrain/main/GUIDE.md` and `checks/INDEX.md` if `git` is unavailable), then follow [`GUIDE.md`](GUIDE.md) strictly. Run read-only checks first, present one consolidated findings table, wait for user confirmation, then apply and verify all confirmed fixes automatically.

[English README](README.md)

**windowbrain** 是一个纯 Markdown、AI-native 的排障与修复指南库，用来帮助运行在 Windows 上的 AI Agent 自动发现并修复各种奇怪的系统环境坑——编码乱码、路径长度限制、Defender 拖慢构建、孤儿进程、WSL2 互操作、工具链与 MCP 异常等。

不需要安装任何 CLI 或依赖，直接把仓库地址丢给你的 AI Agent 即可。

## 使用方法

在 Windows 上的任意 AI Agent（Claude Code、Codex CLI、OpenCode、Cursor、Copilot CLI 等）中直接发送：

```text
read and take action by this guide: https://github.com/wxxb789/windowbrain
and fix all findings on this windows machine
```

也可以带着具体报错定向排障：

```text
read https://github.com/wxxb789/windowbrain and diagnose this error on my Windows machine: <粘贴报错>
```

## 执行机制

1. **只读取证**：Agent 先执行所有只读检查（[`checks/INDEX.md`](checks/INDEX.md)），绝不在确认前修改你的系统。
2. **一次汇总确认**：Agent 输出一张总表，列出所有发现的问题、严重度、拟执行修复、是否需要管理员权限以及回滚命令。
3. **自动修复与复检**：你确认后（`all` / `safe` / 指定编号），Agent 自动完成所有非提权修复，将需要管理员权限的项合并为**一次 UAC** 执行，最后逐项验证并输出报告。

## 问题分类

| 分类 | 目录 | 覆盖范围 |
|---|---|---|
| Shell 与编码 | `checks/shell-encoding/` | UTF-8 控制台/Python/Git 编码、PowerShell/Bash 引号转义、CRLF、ConPTY |
| 路径与文件系统 | `checks/paths-fs/` | `MAX_PATH` / `core.longpaths`、符号链接、`EBUSY` 句柄占用、大小写 |
| 性能与 Defender | `checks/perf-defender/` | Windows Defender 实时扫描开销、Dev Drive、CPU 抢占 |
| 进程与信号 | `checks/processes/` | 孤儿后台进程、残留端口占用、僵尸 MCP/node/python 进程 |
| WSL2 互操作 | `checks/wsl-interop/` | `wsl.exe` 嵌套引用陷阱、`/mnt/c` I/O 延迟、代理/localhost 转发 |
| 工具链与 MCP | `checks/toolchain-mcp/` | `npx`/`uvx` shim 冲突、跨 Shell `PATH` 漂移、UAC 边界、企业代理/CA 证书 |

## 仓库结构

```text
windowbrain/
├── README.md                 # 英文入口（人类 + AI Agent）
├── README.zh-CN.md           # 中文入口（人类 + AI Agent）
├── GUIDE.md                  # 主执行协议（纯面向 AI Agent，en-US）
├── AGENTS.md                 # 仓库维护与脱敏规范（纯面向 AI Agent，en-US）
├── templates/
│   └── CHECK_TEMPLATE.md     # 新增 Check 的标准模板
└── checks/
    ├── INDEX.md              # 机器可读的 Check 总索引
    ├── shell-encoding/
    ├── paths-fs/
    ├── perf-defender/
    ├── processes/
    ├── wsl-interop/
    └── toolchain-mcp/
```

## 添加新坑点

除 `README*.md` 外，其余文档统一使用 `en-US` 并直接面向 AI Agent 编写：

1. 参考 [`templates/CHECK_TEMPLATE.md`](templates/CHECK_TEMPLATE.md) 在 `checks/<category>/<slug>.md` 新增条目。
2. 所有命令必须通用可移植，禁止包含个人用户名、特定盘符或内网路径。
3. 同步在 [`checks/INDEX.md`](checks/INDEX.md) 注册该条目。

## 许可证

[MIT](LICENSE)
