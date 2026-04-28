# ace-tool-rs

[English](README.md) | 简体中文

一个高性能的 MCP（模型上下文协议）服务器，用于代码库索引和语义搜索，使用 Rust 编写。

## 概述

ace-tool-rs 是一个 Rust 实现的代码库上下文引擎，使 AI 助手能够使用自然语言查询来搜索和理解代码库。它提供：

- **实时代码库索引** - 自动索引项目文件并保持索引更新
- **语义搜索** - 使用自然语言描述查找相关代码
- **多语言支持** - 支持 50+ 种编程语言和文件类型
- **增量更新** - 使用 mtime 缓存跳过未更改文件，仅上传新增/修改内容
- **并行处理** - 多线程文件扫描和处理，加快索引速度
- **智能排除** - 遵循 `.gitignore`、`.aceignore` 和常见忽略规则

## 特性

- **MCP 协议支持** - 基于 stdio 的完整 JSON-RPC 2.0 实现
- **自适应上传策略** - AIMD（加性增加，乘性减少）算法根据运行时指标动态调整并发度和超时时间
- **多编码支持** - 支持 UTF-8、GBK、GB18030、Windows-1252 编码
- **并发上传** - 滑动窗口并行批量上传，提升大项目索引效率
- **Mtime 缓存** - 跟踪文件修改时间，避免重复处理
- **健壮错误处理** - 指数退避重试和限流处理

## 安装

### 快速开始（推荐）

```bash
npx ace-tool-rs --base-url <API_URL> --token <AUTH_TOKEN>
```

会自动下载对应平台二进制并运行。

**支持平台：**
- Windows (x64)
- macOS (x64, ARM64)
- Linux (x64, ARM64)

### 从源码构建

```bash
git clone https://github.com/missdeer/ace-tool-rs.git
cd ace-tool-rs
cargo build --release
```

二进制位于 `target/release/ace-tool-rs`。

### 环境要求

- Rust 1.70+
- 索引服务 API 端点
- 认证令牌

## 使用方法

### 命令行

```bash
ace-tool-rs --base-url <API_URL> --token <AUTH_TOKEN>
```

### 参数

| 参数 | 描述 |
|------|------|
| `--base-url` | 索引服务 API 基础地址 |
| `--token` | API 认证令牌 |
| `--transport` | 传输帧格式：`auto`（默认）、`lsp`、`line` |
| `--upload-timeout` | 覆盖上传超时（秒），禁用自适应超时 |
| `--upload-concurrency` | 覆盖上传并发度，禁用自适应并发 |
| `--no-adaptive` | 禁用自适应策略，使用静态启发式值 |
| `--index-only` | 仅索引当前目录并退出（不启动 MCP 服务器） |
| `--max-lines-per-blob` | 每个 blob 的最大行数（默认 800） |
| `--retrieval-timeout` | 搜索超时（秒，默认 60） |

### 环境变量

| 变量 | 描述 |
|------|------|
| `RUST_LOG` | 日志级别，如 `info`、`debug`、`warn` |

### 示例

```bash
RUST_LOG=debug ace-tool-rs --base-url https://api.example.com --token your-token-here
```

### 传输帧格式

默认自动检测行分隔 JSON 与 LSP `Content-Length`。如需强制：

```bash
ace-tool-rs --base-url https://api.example.com --token your-token-here --transport lsp
```

## MCP 集成

### Codex CLI 配置

添加到 Codex 配置文件（通常 `~/.codex/config.toml`）：

```toml
[mcp_servers.ace-tool]
command = "npx"
args = ["ace-tool-rs", "--base-url", "https://api.example.com", "--token", "your-token-here", "--transport", "lsp"]
env = { RUST_LOG = "info" }
startup_timeout_ms = 60000
```

### Claude Desktop 配置

添加到 Claude Desktop 配置文件：

- macOS：`~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows：`%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "ace-tool": {
      "command": "npx",
      "args": [
        "ace-tool-rs",
        "--base-url", "https://api.example.com",
        "--token", "your-token-here"
      ]
    }
  }
}
```

### Claude Code

```bash
claude mcp add-json ace-tool --scope user '{"type":"stdio","command":"npx","args":["ace-tool-rs","--base-url","https://api.example.com/","--token","your-token-here"],"env":{}}'
```

在 `~/.claude/settings.json` 里加权限：

```json
{
  "permissions": {
    "allow": [
      "mcp__ace-tool__search_context"
    ]
  }
}
```

## 可用工具

### `search_context`

用自然语言查询搜索代码库。

**参数：**

| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `project_root_path` | string | 是 | 项目根目录绝对路径 |
| `query` | string | 是 | 需要检索的代码描述 |

**查询示例：**

- “处理用户认证的函数在哪里？”
- “登录功能有哪些测试？”
- “数据库是如何连接到应用的？”
- “找到消息队列消费者初始化流程”

## 支持的文件类型

### 编程语言

`.py`、`.js`、`.ts`、`.jsx`、`.tsx`、`.java`、`.go`、`.rs`、`.cpp`、`.c`、`.h`、`.cs`、`.rb`、`.php`、`.swift`、`.kt`、`.scala`、`.lua`、`.dart`、`.r`、`.jl`、`.ex`、`.hs`、`.zig` 等。

### 配置和数据

`.json`、`.yaml`、`.yml`、`.toml`、`.xml`、`.ini`、`.conf`、`.md`、`.txt`

### Web 技术

`.html`、`.css`、`.scss`、`.sass`、`.vue`、`.svelte`、`.astro`

### 特殊文件

`Makefile`、`Dockerfile`、`Jenkinsfile`、`.gitignore`、`.env.example`、`requirements.txt` 等。

## 默认排除项

- **依赖目录**：`node_modules`、`vendor`、`.venv`、`venv`
- **构建产物**：`target`、`dist`、`build`、`out`、`.next`
- **版本控制**：`.git`、`.svn`、`.hg`
- **缓存目录**：`__pycache__`、`.cache`、`.pytest_cache`
- **二进制文件**：`*.exe`、`*.dll`、`*.so`、`*.pyc`
- **媒体文件**：`*.png`、`*.jpg`、`*.mp4`、`*.pdf`
- **锁文件**：`package-lock.json`、`yarn.lock`、`Cargo.lock`

### 自定义排除

可在项目根目录创建 `.aceignore`，语法与 `.gitignore` 相同。

```gitignore
my-private-folder/
temp-data/
*.local
*.secret
```

`.gitignore` 与 `.aceignore` 会合并，冲突时 `.aceignore` 优先。

## 架构

```text
ace-tool-rs/
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── config.rs
│   ├── index/
│   │   ├── mod.rs
│   │   └── manager.rs
│   ├── mcp/
│   │   ├── mod.rs
│   │   ├── server.rs
│   │   └── types.rs
│   ├── strategy/
│   │   ├── mod.rs
│   │   ├── adaptive.rs
│   │   └── metrics.rs
│   ├── tools/
│   │   ├── mod.rs
│   │   └── search_context.rs
│   └── utils/
│       ├── mod.rs
│       └── project_detector.rs
└── tests/
    ├── config_test.rs
    ├── index_test.rs
    ├── mcp_server_test.rs
    ├── mcp_test.rs
    ├── path_normalizer_test.rs
    ├── tools_test.rs
    └── utils_test.rs
```

## 自适应上传策略

该工具使用 AIMD（加性增加，乘性减少）算法动态优化上传性能。

1. **预热阶段**：从并发 1 开始，在 5-10 个请求后评估并发提升。
2. **加性增加**：成功率 > 95% 且延迟健康时，并发 +1。
3. **乘性减少**：成功率 < 70%、被限流或高延迟时，并发减半、超时 +50%。

### 安全边界

| 参数 | 最小值 | 最大值 |
|------|--------|--------|
| 并发度 | 1 | 8 |
| 超时 | 15s | 180s |

### CLI 覆盖

```bash
ace-tool-rs --base-url ... --token ... --upload-concurrency 4
ace-tool-rs --base-url ... --token ... --upload-timeout 60
ace-tool-rs --base-url ... --token ... --no-adaptive
```

## 开发

### 运行测试

```bash
cargo test
cargo test -- --nocapture
cargo test test_config_new
```

### 构建

```bash
cargo build
cargo build --release
cargo check
cargo clippy
```

## 限制

- 仅处理根目录 `.gitignore` 与 `.aceignore`
- 需要网络访问索引 API
- 单文件最大 128KB
- 单批次最大 1MB

## 许可证

双许可证：

- **非商业/个人使用**：GPLv3（见 [LICENSE](LICENSE)）
- **商业/工作场景**：需商业许可证（见 [LICENSE-COMMERCIAL](LICENSE-COMMERCIAL)）

商业授权联系：missdeer@gmail.com

## 作者

[missdeer](https://github.com/missdeer)
