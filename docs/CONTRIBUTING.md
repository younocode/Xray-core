# Xray-core 新人参与指南 | Contributor's Guide

[English](#english) | [中文](#中文)

---

<a name="中文"></a>
# 中文版

## 一、项目概览

Xray-core 是一个模块化的网络代理平台，使用 Go 1.25+ 开发，实现了 VLESS、XTLS、REALITY、VMess、Trojan、Shadowsocks 等多种协议。

**关键信息：**
- **主要语言：** Go 1.25+
- **架构：** 分层模块化设计
- **测试：** 跨平台（Windows、Linux、macOS）
- **构建目标：** 20+ 种架构/操作系统组合
- **提交风格：** Conventional Commits（如 `fix(dns): description`）
- **许可证：** [Mozilla Public License Version 2.0](../LICENSE)
- **行为准则：** [Contributor Covenant Code of Conduct](../CODE_OF_CONDUCT.md)

**社区渠道：**
- [GitHub Discussions](https://github.com/XTLS/Xray-core/discussions) - 提问和讨论
- [Telegram: Project X](https://t.me/projectXray) - 主群
- [Telegram: Project X Channel](https://t.me/projectXtls) - 公告频道
- [Telegram: Project VLESS](https://t.me/projectVless) - Русский
- [Telegram: Project XHTTP](https://t.me/projectXhttp) - Persian

**AI 辅助：**
- [![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/XTLS/Xray-core) - 使用 AI 查询项目文档

---

## 二、项目结构速览

```
Xray-core/
├── main/              # CLI 入口（main.go, run.go）
├── core/              # 核心运行时（Instance、依赖注入、配置）
├── features/          # 功能抽象接口（DNS、路由、策略等）
├── app/               # features 的具体实现
├── proxy/             # 协议实现（vless、vmess、trojan、shadowsocks等）
├── transport/         # 传输层（TCP、UDP、WebSocket、gRPC、KCP、REALITY等）
├── common/            # 工具包（buf、net、crypto、errors、protocol等）
├── infra/             # 配置加载和解析（conf、vprotogen）
├── testing/           # 测试基础设施（scenarios、mocks、coverage）
└── .github/           # CI/CD（workflows: test.yml、release.yml）
```

**核心文件路径：**
| 文件 | 说明 |
|------|------|
| `main/main.go`, `main/run.go` | 程序入口 |
| `core/xray.go` | 服务器实例、依赖注入 |
| `core/config.go` | 配置加载 |
| `infra/conf/serial/loader.go` | 配置反序列化 |

---

## 三、参与路径（从易到难）

### Level 1: 文档贡献（零门槛）

**适合人群：** 所有新人，无需代码能力

**典型任务：**
1. 更新 README.md 添加新的 GUI 客户端/面板
2. 修复文档中的错别字或过期链接
3. 翻译文档（中英文互译）
4. 添加配置示例和使用教程

**提交示例：**
```bash
git commit -m "docs(readme): add new macOS client XrayUI"
```

---

### Level 2: 简单 Bug 修复

**适合人群：** 有 Go 基础，想接触实际代码

**前置准备：**
1. 安装 Go 1.25+
2. 下载 geodata 文件（`geoip.dat`、`geosite.dat`）到 `resources/` 目录
3. 运行 `go test ./...` 确保环境正常

**典型任务：**
- 修复单个函数/文件的逻辑错误
- 添加缺失的错误检查
- 修复已报告的简单 bug

---

### Level 3: 功能增强

**适合人群：** 熟悉 Go、了解网络基础、能阅读 protobuf

**典型任务：**
- 为现有协议添加新配置选项
- 优化性能（如缓存、并发）
- 增强日志输出
- 改进错误处理

**关键步骤：**
1. 探索现有实现 - 先阅读相关包的代码
2. 修改 proto 文件（如需要）
3. 重新生成 protobuf
4. 更新配置解析器（`infra/conf/`）
5. 实现功能逻辑
6. 添加测试
7. 更新示例配置

---

### Level 4: 新协议/传输实现

**适合人群：** 深入理解网络协议、TLS、加密学

**开发模板（以添加新协议 `MyProtocol` 为例）：**

```
proxy/myprotocol/
├── myprotocol.proto         # 配置定义
├── myprotocol.pb.go         # 生成的 protobuf 代码
├── account.go               # 用户账号管理
├── validator.go             # 账号验证器
├── inbound/
│   ├── inbound.go          # 服务端处理器
│   └── inbound_test.go
├── outbound/
│   ├── outbound.go         # 客户端处理器
│   └── outbound_test.go
└── encoding/
    ├── encoding.go         # 消息编解码
    └── encoding_test.go
```

---

## 四、开发环境搭建

### 1. 基础环境

```bash
# 1. 安装 Go 1.25+
go version  # 确认版本

# 2. 克隆仓库
git clone https://github.com/XTLS/Xray-core.git
cd Xray-core

# 3. 下载依赖
go mod download

# 4. 下载 geodata（测试必需）
mkdir -p resources
cd resources
wget https://github.com/v2fly/geoip/releases/latest/download/geoip.dat
wget https://github.com/v2fly/domain-list-community/releases/latest/download/dlc.dat -O geosite.dat
cd ..
```

### 2. 构建和运行

```bash
# 开发构建（快速）
CGO_ENABLED=0 go build -o xray -trimpath -buildvcs=false \
  -ldflags="-s -w -buildid=" -v ./main

# 直接运行（无需编译）
go run ./main -config config.json

# 测试配置文件
go run ./main -test -config config.json

# 打印合并后的配置
go run ./main -dump -config config.json
```

### 3. 测试

```bash
# 运行全部测试
go test ./...

# 运行特定包的测试
go test -v ./proxy/vless/...

# 运行单个测试函数
go test -v -run TestConfigLoader ./infra/conf/serial/

# 检测竞态条件
go test -race ./...

# 生成覆盖率报告
./testing/coverage/coverall

# 静态检查
go vet ./...
```

### 4. Protobuf 工作流

```bash
# 安装 protoc 编译器
# macOS: brew install protobuf
# Linux: apt-get install protobuf-compiler

# 安装 Go 插件
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# 修改 .proto 文件后重新生成
protoc --go_out=. --go_opt=paths=source_relative path/to/your.proto

# 注意：不要手动编辑 .pb.go 文件！
```

---

## 五、代码规范

### 1. Git 提交规范（Conventional Commits）

**格式：** `<type>(<scope>): <description>`

**类型（type）：**
| 类型 | 说明 |
|------|------|
| `fix` | Bug 修复 |
| `feat` | 新功能 |
| `perf` | 性能优化 |
| `refactor` | 重构（不改变外部行为） |
| `docs` | 文档更新 |
| `chore` | 构建/工具变更 |
| `test` | 测试相关 |

**示例：**
```bash
git commit -m "fix(dns): resolve cache inheritance issue

Fixes #5351"

git commit -m "feat(transport): add HTTP/3 support

Implements QUIC-based HTTP/3 transport for improved performance.

Closes #1234"
```

### 2. Go 代码风格

- 使用 `gofmt`（标准 Go 格式化，tab 缩进）
- 包名：小写、单数
- 导出符号：PascalCase
- 未导出符号：camelCase
- 配置键和 JSON/TOML 标签必须保持稳定（向后兼容）
- 添加新配置字段时提供默认值和验证

### 3. 安全要求

- 防范常见漏洞：命令注入、XSS、SQL 注入
- 只在系统边界验证（用户输入、外部 API）
- 信任内部代码和框架保证

---

## 六、贡献流程

### 1. 开发前

1. **搜索现有 Issue/PR** - 确认没有重复工作
2. **阅读文档** - 确保理解现有功能
3. **创建 Issue（可选）** - 对于大功能，先讨论设计
4. **Fork 仓库** - 在自己的 fork 上开发

### 2. 开发中

```bash
# 1. 创建功能分支
git checkout -b feat/my-feature

# 2. 开发和测试
go test ./...
go vet ./...

# 3. 提交（遵循 Conventional Commits）
git add .
git commit -m "feat(scope): description"

# 4. 推送到 fork
git push origin feat/my-feature
```

### 3. 创建 Pull Request

**PR 标题：** 使用 Conventional Commits 格式

**PR 描述模板：**
```markdown
## Summary
Brief description of the changes.

Fixes #1234 (if applicable)

## Changes
- Added XXX feature
- Modified YYY logic
- Optimized ZZZ performance

## Testing
- [x] `go test ./...` passed
- [x] `go vet ./...` passed
- [x] Added unit tests
- [x] Tested locally

## Backward Compatibility
- [x] Old configs still work
- [x] No API breaking changes
```

### 4. Issue 提交规范

请按照 `.github/ISSUE_TEMPLATE/` 中的模板提交 Issue：
- 提供完整、可复现的配置（不要截断）
- 包含完整的调试日志（服务端和客户端）
- 描述复现步骤
- 确认问题在最新 Release 版本中存在
- **提问请使用 GitHub Discussions，不要在 Issues 中提问**

---

## 七、常见陷阱

### 错误做法

| 错误 | 正确做法 |
|------|----------|
| 手动编辑 `.pb.go` 文件 | 修改 `.proto` 后用 `protoc` 重新生成 |
| 破坏配置兼容性 | 为新字段提供默认值，旧配置依然有效 |
| 提交前不测试 | 每次提交前运行 `go test ./...` |
| 不阅读现有代码直接提 PR | 先理解现有实现 |
| 在 Issues 中提问 | 使用 GitHub Discussions 提问 |

### 注意事项

- **geodata 文件** - 测试前必须下载到 `resources/`
- **平台差异** - 测试在多个平台（Windows/Linux/macOS）上运行
- **性能敏感** - 许多提交涉及优化，需要考虑性能影响
- **协议知识** - 大部分贡献需要理解 VLESS、XTLS、REALITY 等协议

---

## 八、学习路径建议

### 第 1 周：熟悉项目

1. **阅读关键文件**
   - [README.md](../README.md) - 项目概述
   - [CLAUDE.md](../CLAUDE.md) - 开发指南
   - `main/run.go` - 理解启动流程
   - `core/xray.go` - 理解核心架构

2. **运行项目**
   ```bash
   go run ./main -config config.json
   ```

3. **运行测试**
   ```bash
   go test ./proxy/vless/...
   go test ./app/dns/...
   ```

### 第 2 周：贡献文档

1. 找一个文档任务（或自己发现可改进的文档）
2. 提交第一个 PR（修复拼写错误、添加链接等）

### 第 3-4 周：简单代码修改

1. 选择一个简单 bug 或 `good first issue`
2. 按照流程完成修复：理解问题 → 定位代码 → 修改 → 测试 → 提交 PR

### 第 5-8 周：深入某个子系统

选择感兴趣的领域深入研究：
- DNS：`app/dns/`
- 路由：`app/router/`
- 传输：`transport/internet/`
- 协议：`proxy/vless/`、`proxy/trojan/`

---

## 九、资源链接

### 官方资源
- **项目主页：** https://github.com/XTLS/Xray-core
- **官方文档：** https://xtls.github.io
- **配置示例：** https://github.com/XTLS/Xray-examples
- **REALITY 文档：** https://github.com/XTLS/REALITY

### 技术文档
- **Go 官方文档：** https://go.dev/doc/
- **Protobuf 指南：** https://protobuf.dev/

---

## 十、快速上手清单

```bash
# 1. 环境搭建
git clone https://github.com/XTLS/Xray-core.git
cd Xray-core
go mod download

# 2. 下载 geodata
mkdir -p resources && cd resources
wget https://github.com/v2fly/geoip/releases/latest/download/geoip.dat
wget https://github.com/v2fly/domain-list-community/releases/latest/download/dlc.dat -O geosite.dat
cd ..

# 3. 构建和测试
go build -o xray ./main
go test ./...

# 4. 运行
./xray -config config.json
```

**关键记忆点：**
- 使用 Conventional Commits 格式
- 提交前运行 `go test ./...`
- 不要手动编辑 `.pb.go` 文件
- 保持配置向后兼容
- 先阅读代码，再提 PR

---

<a name="english"></a>
# English Version

## Overview

Xray-core is a modular network proxy platform written in Go 1.25+, implementing protocols like VLESS, XTLS, REALITY, VMess, Trojan, and Shadowsocks.

**Key Information:**
- **Language:** Go 1.25+
- **Architecture:** Layered modular design
- **Testing:** Cross-platform (Windows, Linux, macOS)
- **Build Targets:** 20+ architecture/OS combinations
- **Commit Style:** Conventional Commits (e.g., `fix(dns): description`)
- **License:** [Mozilla Public License Version 2.0](../LICENSE)
- **Code of Conduct:** [Contributor Covenant](../CODE_OF_CONDUCT.md)

**Community Channels:**
- [GitHub Discussions](https://github.com/XTLS/Xray-core/discussions) - Q&A and discussions
- [Telegram: Project X](https://t.me/projectXray) - Main group
- [Telegram: Project X Channel](https://t.me/projectXtls) - Announcements

**AI Assistance:**
- [![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/XTLS/Xray-core)

---

## Quick Start

```bash
# 1. Clone and setup
git clone https://github.com/XTLS/Xray-core.git
cd Xray-core
go mod download

# 2. Download geodata (required for tests)
mkdir -p resources && cd resources
wget https://github.com/v2fly/geoip/releases/latest/download/geoip.dat
wget https://github.com/v2fly/domain-list-community/releases/latest/download/dlc.dat -O geosite.dat
cd ..

# 3. Build and test
go build -o xray ./main
go test ./...

# 4. Run
./xray -config config.json
```

---

## Contribution Levels

### Level 1: Documentation (No code required)
- Update README.md with new clients/panels
- Fix typos or broken links
- Add configuration examples

### Level 2: Simple Bug Fixes
- Fix single function/file logic errors
- Add missing error checks

### Level 3: Feature Enhancements
- Add new config options to existing protocols
- Performance optimizations
- Improve logging/error handling

### Level 4: New Protocol/Transport Implementation
- Implement new proxy protocols
- Add new transport methods
- Integrate third-party libraries

---

## Development Workflow

### Build Commands

```bash
# Development build
CGO_ENABLED=0 go build -o xray -trimpath -buildvcs=false \
  -ldflags="-s -w -buildid=" -v ./main

# Run directly
go run ./main -config config.json
```

### Testing

```bash
# Run all tests
go test ./...

# Run specific package tests
go test -v ./proxy/vless/...

# Race detection
go test -race ./...

# Static analysis
go vet ./...
```

### Commit Convention

Use Conventional Commits format: `<type>(<scope>): <description>`

Types: `fix`, `feat`, `perf`, `refactor`, `docs`, `chore`, `test`

Example:
```bash
git commit -m "fix(dns): resolve cache inheritance issue

Fixes #5351"
```

---

## Project Structure

```
Xray-core/
├── main/              # CLI entrypoint
├── core/              # Runtime orchestration, DI
├── features/          # Abstract interfaces
├── app/               # Feature implementations
├── proxy/             # Protocol implementations
├── transport/         # Transport layer
├── common/            # Shared utilities
├── infra/             # Config loading/parsing
├── testing/           # Test infrastructure
└── .github/           # CI/CD workflows
```

---

## Key Rules

1. **Never manually edit `.pb.go` files** - Regenerate from `.proto`
2. **Maintain backward compatibility** - Provide defaults for new config fields
3. **Test before committing** - Run `go test ./...`
4. **Read existing code first** - Understand before modifying
5. **Use Discussions for questions** - Issues are for bugs/features only

---

## Resources

- **Official Docs:** https://xtls.github.io
- **Examples:** https://github.com/XTLS/Xray-examples
- **REALITY:** https://github.com/XTLS/REALITY
- **Go Docs:** https://go.dev/doc/
- **Protobuf Guide:** https://protobuf.dev/
