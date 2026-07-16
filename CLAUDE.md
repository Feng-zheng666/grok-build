# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Grok Build（`grok`）是 SpaceXAI 的终端 AI 编程助手 — 一个用 Rust 编写的全屏 TUI 应用，能理解代码库、编辑文件、执行 shell 命令、搜索网页、管理长时间任务。支持交互式 TUI、headless 脚本/CI 模式、以及通过 ACP（Agent Client Protocol）嵌入编辑器。

代码从 SpaceXAI 内部 monorepo 定期同步至此公开仓库。**不接受外部 PR**。

## 构建与开发命令

```sh
# 构建并运行 TUI
cargo run -p xai-grok-pager-bin

# Release 构建
cargo build -p xai-grok-pager-bin --release          # 本地开发用，incremental
cargo build -p xai-grok-pager-bin --profile release-dist  # 正式发布用，thin LTO + 1 codegen unit

# 快速检查（按 crate 指定，全 workspace 构建很慢）
cargo check -p xai-grok-pager-bin
cargo check -p <crate>

# 全 workspace 验证（工具链版本升级后必须执行）
cargo check --all-targets --workspace
cargo clippy --all-targets --workspace

# 测试（按 crate）
cargo test -p xai-grok-config
cargo test -p xai-grok-shell
cargo test -p xai-grok-tools

# Lint
cargo clippy -p <crate>

# 格式化
cargo fmt --all
```

**构建前提**：Rust 工具链由 `rust-toolchain.toml` 锁定（当前 1.92.0），`rustup` 自动安装。需要 `protoc`（自动解析 `bin/protoc` dotslash 启动器，或 PATH 中的 `protoc`）。macOS/Linux 为主要构建平台，Windows 仅尽力支持。

### Build Profiles

| Profile | 用途 | 关键配置 |
|---------|------|---------|
| `dev` | 本地开发 | `panic = "abort"`, `codegen-units = 128`, `opt-level = 0` |
| `release` | 本地 release 测试 | `incremental = true`, `panic = "abort"` |
| `release-dist` | 正式发布 | `lto = "thin"`, `codegen-units = 1`, `debug = 1` |
| `x-prod` | 生产环境 | 继承 `release`, `lto = "thin"`, `strip = false`, `debug = "line-tables-only"` |
| `release-dist-jemalloc` | jemalloc 发布 | 继承 `release-dist`，配置完全相同（别名） |

### 工具链版本升级流程

来自 `rust-toolchain.toml` 注释：每次只升一个小版本，等新版发布至少两周后再升。升级后必须执行：

```sh
cargo check --all-targets --workspace
cargo clippy --all-targets --workspace
```

## 架构概览

### Crate 目录约定

| 路径 | 内容 |
|------|------|
| `crates/codegen/` | CLI/TUI 核心 crate 闭包（~60 个 crate） |
| `crates/common/` | 小型共享基础库（工具协议、熔断器、压缩等） |
| `crates/build/` | 构建辅助（proto 代码生成） |
| `prod/mc/` | 微服务相关 crate |
| `third_party/` | 供应商源码（Mermaid 图表栈：dagre_rust, graphlib_rust, mermaid-to-svg） |

### Crate 分层

```
xai-grok-pager-bin          ← 组合根，构建二进制文件 xai-grok-pager（发布为 grok）
├── xai-grok-pager           ← TUI 层：scrollback、prompt、modal、渲染（ratatui + crossterm）
│   └── xai-grok-pager-render ← 渲染原语
├── xai-grok-shell           ← agent 运行时核心：leader/stdio/headless 入口、认证、MCP 编排、工具调度
│   ├── agent::app           ← 入口：run_headless / run_leader / run_stdio_agent
│   ├── agent::mvp_agent     ← 主 agent 循环（MvpAgent）
│   ├── agent::subagent      ← 子 agent 协调器
│   ├── auth                 ← OIDC 认证、令牌刷新
│   ├── leader               ← 长生命期 leader 守护进程（session 复用）
│   ├── extensions           ← 内置 slash 命令、hook、文件系统、git 等扩展
│   ├── session              ← session 管理
│   ├── relay                ← 模型 API 中继
│   └── managed_config       ← 受管配置
├── xai-grok-agent           ← Agent 定义解析、系统提示组装（minijinja 模板）
├── xai-grok-tools           ← 工具实现：terminal/bash、文件编辑、搜索、LSP、浏览器等
│   └── xai-grok-tools-api   ← 工具 API 类型
├── xai-grok-workspace       ← 宿主机文件系统、VCS (libgit2)、命令执行、checkpoint、文件监控
│   ├── xai-grok-workspace-types  ← workspace 共享类型
│   └── xai-grok-workspace-client ← workspace 客户端
├── xai-grok-mcp             ← MCP 服务器集成（rmcp 2.1）
├── xai-grok-config          ← 配置层（TOML 解析、设置管理）
│   └── xai-grok-config-types ← 配置类型定义
├── xai-grok-markdown        ← Markdown 渲染
│   └── xai-grok-markdown-core ← Markdown 核心
├── xai-grok-sandbox         ← 沙箱执行
├── xai-grok-auth            ← 认证 HTTP 中间件
├── xai-grok-memory          ← 持久化记忆存储
├── xai-grok-hooks           ← Hook 系统
├── xai-grok-plugin-marketplace ← 插件市场
├── xai-grok-subagent-resolution ← 子 agent 解析
├── xai-grok-models          ← 模型定义
├── xai-grok-telemetry       ← 遥测
├── xai-grok-update          ← 自动更新
├── xai-grok-voice           ← 语音输入
├── xai-grok-http            ← HTTP 客户端
├── xai-grok-env             ← 环境变量管理
├── xai-grok-paths           ← 路径管理
├── xai-grok-secrets         ← 密钥管理
├── xai-grok-shell-base      ← Shell 基础共享类型
├── xai-grok-shell-session-support ← Session 支持
├── xai-grok-test-support    ← 共享测试工具（mock、fixture 等）
├── xai-grok-sampler         ← 采样
├── xai-grok-sampling-types  ← 采样类型
├── xai-acp-lib              ← ACP 协议库
├── xai-grok-mermaid         ← Mermaid 图表渲染
├── xai-grok-announcements   ← 公告
├── xai-grok-shared          ← 共享工具
├── xai-grok-compaction      ← 上下文压缩（crates/common/）
├── xai-circuit-breaker      ← 熔断器（crates/common/）
├── xai-tool-protocol        ← 工具协议定义（crates/common/）
├── xai-tool-runtime         ← 工具运行时（crates/common/）
├── xai-tool-types           ← 工具类型（crates/common/）
├── xai-computer-hub-*       ← 计算机集线器（远程执行，crates/common/）
└── third_party/             ← 供应商源码（Mermaid 图表栈）
```

### 核心数据流

1. **TUI 启动** → `xai-grok-pager-bin/src/main.rs` → 解析 CLI 参数 → 启动 `xai-grok-pager` TUI 或分发至 headless/leader/stdio 模式
2. **Agent 循环** → `xai-grok-shell::agent::mvp_agent::MvpAgent` → 接收用户 prompt → 发送至模型 API → 解析工具调用 → 调度到 `xai-grok-tools` → 返回结果给模型 → 渲染响应
3. **工具执行** → `xai-grok-tools` 通过 `xai-grok-workspace` 访问文件系统/VCS，通过 `xai-grok-sandbox` 隔离执行
4. **Leader 模式** → `xai-grok-shell::leader` 长生命期守护进程，复用认证和 session 状态，TUI 通过 Unix socket/WebSocket 连接

### Shell Agent 运行时内部模块

`xai-grok-shell` 是核心 crate，内部模块组织：

- `agent::app` — 四个入口：`run_headless`、`run_leader`、`run_stdio_agent`、`run_desktop_agent`
- `agent::mvp_agent` — MvpAgent 主循环，处理 prompt → 模型 → 工具调用 → 响应
- `agent::subagent` — 子 agent 协调（spawn、delegate、collect）
- `agent::handlers` — 事件处理器
- `auth` — OIDC 认证流程、token 管理
- `leader` — 守护进程模式，session 复用
- `extensions` — 内置 slash 命令、hook 集成
- `session` — session 生命周期管理
- `relay` — 模型 API 请求中继
- `terminal` — 终端 I/O 管理
- `tools` — 工具代理层
- `managed_config` — 远程受管配置
- `inspect` — 调试和诊断工具

## 关键约定

### 根 `Cargo.toml`
**自动生成，只读**。始终编辑各 crate 自己的 `Cargo.toml`。workspace 成员、依赖版本、lint 配置、profiles 均由此文件集中管理。

### 路径规范化
禁止使用 `std::fs::canonicalize`（Windows 返回 verbatim `\\?\` 路径），用 `dunce::canonicalize` 替代。在 `xai-grok-tools` 的异步上下文中，使用 `crate::util::fs` 中的辅助函数。此规则由 `clippy.toml` 中的 `disallowed-methods` 强制执行。

### clippy.toml
- 仓库根目录 `clippy.toml` 配置所有 crate 的 lint 规则
- **clippy 使用最近的 clippy.toml，不会合并** — 所以 `crates/codegen/` 有其自己的副本，修改规则时必须两边同步
- `disallowed-methods` 禁止 `std::fs::canonicalize`、`std::path::Path::canonicalize`、`tokio::fs::canonicalize`
- `large-error-threshold = 256`（tonic 生成的代码需要较大错误类型）

### .cargo/config.toml
平台特定的 rustflags：
- **macOS**：`-C link-arg=-undefined dynamic_lookup`、`-C link-args=-ObjC`
- **Linux musl**：binary hardening — Full RELRO + non-executable stack（`-Wl,-z,relro,-z,now,-z,noexecstack`）
- **Windows**：`-C target-feature=+crt-static`
- **aarch64 Linux**：`-C target-cpu=neoverse-v2`（gnu）/ `-C target-cpu=generic`（musl）
- jemalloc page size 环境变量：Apple Silicon = 16KB，aarch64 Linux = 64KB；aarch64 musl 禁用 background threads

### 异步运行时
tokio multi-thread，jemalloc 为 Unix 默认分配器。

### 格式化
`rustfmt.toml`：`use_field_init_shorthand = true`

### 提交信息
`type: 简短描述`（feat/fix/docs/refactor/test/chore）
