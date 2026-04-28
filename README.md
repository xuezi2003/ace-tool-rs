# ace-tool-rs

English | [简体中文](README-zh-CN.md)

A high-performance MCP (Model Context Protocol) server for codebase indexing and semantic search, written in Rust.

## Overview

ace-tool-rs is a Rust implementation of a codebase context engine that enables AI assistants to search and understand codebases using natural language queries. It provides:

- **Real-time codebase indexing** - Automatically indexes project files and keeps the index up-to-date
- **Semantic search** - Find relevant code using natural language descriptions
- **Multi-language support** - Works with 50+ programming languages and file types
- **Incremental updates** - Uses mtime caching to skip unchanged files and only uploads new/modified content
- **Parallel processing** - Multi-threaded file scanning and processing for faster indexing
- **Smart exclusions** - Respects `.gitignore`, `.aceignore` and common ignore patterns

## Features

- **MCP Protocol Support** - Full JSON-RPC 2.0 implementation over stdio transport
- **Adaptive Upload Strategy** - AIMD (Additive Increase, Multiplicative Decrease) algorithm dynamically adjusts concurrency and timeout based on runtime metrics
- **Multi-encoding Support** - Handles UTF-8, GBK, GB18030, and Windows-1252 encoded files
- **Concurrent Uploads** - Parallel batch uploads with sliding window for faster indexing of large projects
- **Mtime Caching** - Tracks file modification times to avoid re-processing unchanged files
- **Robust Error Handling** - Retry logic with exponential backoff and rate limiting support

## Installation

### Quick Start (Recommended)

```bash
npx ace-tool-rs --base-url <API_URL> --token <AUTH_TOKEN>
```

This downloads the proper binary for your platform and runs it.

**Supported platforms:**
- Windows (x64)
- macOS (x64, ARM64)
- Linux (x64, ARM64)

### From Source

```bash
git clone https://github.com/missdeer/ace-tool-rs.git
cd ace-tool-rs
cargo build --release
```

The binary is at `target/release/ace-tool-rs`.

### Requirements

- Rust 1.70+
- An indexing API endpoint
- Authentication token

## Usage

### Command Line

```bash
ace-tool-rs --base-url <API_URL> --token <AUTH_TOKEN>
```

### Arguments

| Argument | Description |
|----------|-------------|
| `--base-url` | API base URL for the indexing service |
| `--token` | Authentication token for API access |
| `--transport` | Transport framing: `auto` (default), `lsp`, `line` |
| `--upload-timeout` | Override upload timeout in seconds (disables adaptive timeout) |
| `--upload-concurrency` | Override upload concurrency (disables adaptive concurrency) |
| `--no-adaptive` | Disable adaptive strategy, use static heuristic values |
| `--index-only` | Index current directory and exit (no MCP server) |
| `--max-lines-per-blob` | Maximum lines per blob chunk (default: 800) |
| `--retrieval-timeout` | Search retrieval timeout in seconds (default: 60) |

### Environment Variables

| Variable | Description |
|----------|-------------|
| `RUST_LOG` | Log level, e.g. `info`, `debug`, `warn` |

### Example

```bash
RUST_LOG=debug ace-tool-rs --base-url https://api.example.com --token your-token-here
```

### Transport Framing

By default, the server auto-detects line-delimited JSON vs. LSP `Content-Length` framing.

```bash
ace-tool-rs --base-url https://api.example.com --token your-token-here --transport lsp
```

## MCP Integration

### Codex CLI Configuration

Add to Codex config (typically `~/.codex/config.toml`):

```toml
[mcp_servers.ace-tool]
command = "npx"
args = ["ace-tool-rs", "--base-url", "https://api.example.com", "--token", "your-token-here", "--transport", "lsp"]
env = { RUST_LOG = "info" }
startup_timeout_ms = 60000
```

### Claude Desktop Configuration

Add to Claude Desktop config:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

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

Grant tool permission in `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "mcp__ace-tool__search_context"
    ]
  }
}
```

## Available Tools

### `search_context`

Search the codebase using natural language queries.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `project_root_path` | string | Yes | Absolute path to the project root directory |
| `query` | string | Yes | Natural language description of the code you're looking for |

**Example queries:**

- "Where is the function that handles user authentication?"
- "What tests are there for the login functionality?"
- "How is the database connected to the application?"
- "Find the initialization flow of message queue consumers"

## Supported File Types

### Programming Languages

`.py`, `.js`, `.ts`, `.jsx`, `.tsx`, `.java`, `.go`, `.rs`, `.cpp`, `.c`, `.h`, `.cs`, `.rb`, `.php`, `.swift`, `.kt`, `.scala`, `.lua`, `.dart`, `.r`, `.jl`, `.ex`, `.hs`, `.zig`, and many more.

### Configuration & Data

`.json`, `.yaml`, `.yml`, `.toml`, `.xml`, `.ini`, `.conf`, `.md`, `.txt`

### Web Technologies

`.html`, `.css`, `.scss`, `.sass`, `.vue`, `.svelte`, `.astro`

### Special Files

`Makefile`, `Dockerfile`, `Jenkinsfile`, `.gitignore`, `.env.example`, `requirements.txt`, and more.

## Default Exclusions

- **Dependencies**: `node_modules`, `vendor`, `.venv`, `venv`
- **Build artifacts**: `target`, `dist`, `build`, `out`, `.next`
- **Version control**: `.git`, `.svn`, `.hg`
- **Cache directories**: `__pycache__`, `.cache`, `.pytest_cache`
- **Binary files**: `*.exe`, `*.dll`, `*.so`, `*.pyc`
- **Media files**: `*.png`, `*.jpg`, `*.mp4`, `*.pdf`
- **Lock files**: `package-lock.json`, `yarn.lock`, `Cargo.lock`

### Custom Exclusions

Create `.aceignore` in project root. Syntax is the same as `.gitignore`.

```gitignore
my-private-folder/
temp-data/
*.local
*.secret
```

`.gitignore` and `.aceignore` patterns are merged. `.aceignore` wins on conflicts.

## Architecture

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

## Adaptive Upload Strategy

The tool uses AIMD (Additive Increase, Multiplicative Decrease) inspired by TCP congestion control.

1. **Warmup**: starts with concurrency=1, evaluates success over 5-10 requests, then jumps toward target.
2. **Additive increase**: if success rate > 95% and latency is healthy, concurrency +1.
3. **Multiplicative decrease**: if success rate < 70%, rate limited, or latency spikes, concurrency halves and timeout +50%.

### Safety Bounds

| Parameter | Minimum | Maximum |
|-----------|---------|---------|
| Concurrency | 1 | 8 |
| Timeout | 15s | 180s |

### CLI Overrides

```bash
ace-tool-rs --base-url ... --token ... --upload-concurrency 4
ace-tool-rs --base-url ... --token ... --upload-timeout 60
ace-tool-rs --base-url ... --token ... --no-adaptive
```

## Development

### Running Tests

```bash
cargo test
cargo test -- --nocapture
cargo test test_config_new
```

### Building

```bash
cargo build
cargo build --release
cargo check
cargo clippy
```

## Limitations

- Only root `.gitignore` and `.aceignore` are processed
- Requires network access to indexing API
- Maximum file size: 128KB per file
- Maximum batch size: 1MB per upload batch

## License

Dual license:

- **Non-commercial / personal use**: GPLv3 ([LICENSE](LICENSE))
- **Commercial / workplace use**: commercial license required ([LICENSE-COMMERCIAL](LICENSE-COMMERCIAL))

Commercial licensing: missdeer@gmail.com

## Author

[missdeer](https://github.com/missdeer)
