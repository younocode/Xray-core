# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Xray-core is a network proxy platform implementing protocols like VLESS, XTLS, REALITY, VMess, Trojan, Shadowsocks, and others. The codebase is written in Go and provides a modular architecture for building various proxy configurations.

## Build Commands

### Standard Build
```bash
# Basic build (creates xray binary in repo root)
CGO_ENABLED=0 go build -o xray -trimpath -buildvcs=false -ldflags="-s -w -buildid=" -v ./main

# Windows PowerShell
$env:CGO_ENABLED=0
go build -o xray.exe -trimpath -buildvcs=false -ldflags="-s -w -buildid=" -v ./main
```

### Reproducible Build (for releases)
```bash
# Set git commit id (7 bytes) for versioning
CGO_ENABLED=0 go build -o xray -trimpath -buildvcs=false -gcflags="all=-l=4" -ldflags="-X github.com/xtls/xray-core/core.build=REPLACE -s -w -buildid=" -v ./main

# For 32-bit MIPS/MIPSLE targets
CGO_ENABLED=0 go build -o xray -trimpath -buildvcs=false -gcflags="-l=4" -ldflags="-X github.com/xtls/xray-core/core.build=REPLACE -s -w -buildid=" -v ./main
```

### Quick Development Run
```bash
# Run directly from source with a config file
go run ./main -config config.json

# Or use xray subcommands
go run ./main run -config config.json
```

## Testing Commands

```bash
# Run all tests
go test ./...

# Run a single test (use -run with regex pattern)
go test -v -run TestConfigLoader ./infra/conf/serial/

# Run tests in a specific package
go test -v ./proxy/vless/...

# Run tests with race detector (for concurrency changes)
go test -race ./...

# Run tests with timeout (CI uses this)
go test -timeout 1h -v ./...

# Run coverage tests (generates coverage.txt in out/xray/cov/)
./testing/coverage/coverall

# Run coverage for a specific package (uses -tags "json coverage")
go test -tags "json coverage" -coverprofile=coverage.out ./proxy/vless/...

# Run static checks before submitting PR
go vet ./...
```

### Test Data Requirements
Some tests require geodata files. Download them to `resources/` directory:
- `geoip.dat` - GeoIP database
- `geosite.dat` - Domain database

The CI workflow caches these files automatically.

## Architecture

### Core Layers

- **main/**: CLI entrypoint with subcommands in `main/commands/`. The binary target for `xray`.
- **core/**: Runtime orchestration, config schema, and protobuf definitions (`config.proto` → `config.pb.go`). The `Instance` type manages the server lifecycle and feature dependency injection.
- **features/**: Abstract interfaces for core functionalities (DNS, inbound/outbound, routing, stats, policy). Implementations are in `app/`.
- **app/**: Concrete implementations of features (dns, dispatcher, proxyman, router, stats, policy, log, metrics, observatory).
- **proxy/**: Protocol implementations:
  - `vless/`: VLESS protocol (including REALITY support)
  - `vmess/`: VMess protocol
  - `trojan/`: Trojan protocol
  - `shadowsocks/`, `shadowsocks_2022/`: Shadowsocks implementations
  - `socks/`, `http/`: SOCKS and HTTP proxy protocols
  - `freedom/`: Direct outbound connection
  - `wireguard/`: WireGuard VPN integration
  - `blackhole/`: Blackhole/reject handler
  - `dokodemo/`: Port forwarding/transparent proxy
  - `dns/`: DNS outbound
  - `loopback/`: Loopback connections

### Transport Layer

- **transport/internet/**: Transport protocol implementations:
  - `tcp/`, `udp/`: Basic TCP/UDP transports
  - `websocket/`: WebSocket transport
  - `grpc/`: gRPC transport
  - `kcp/`: KCP (UDP-based) transport
  - `httpupgrade/`: HTTP upgrade transport
  - `splithttp/`: Split HTTP transport
  - `reality/`: REALITY protocol (TLS fingerprint masking)
  - `tls/`: TLS wrapper
  - `headers/`: HTTP header manipulation
- **transport/pipe/**: Internal data pipe for connection handling
- **common/**: Shared utilities:
  - `buf/`: Buffer management and pooling
  - `net/`: Network address/connection utilities
  - `crypto/`: Cryptographic functions
  - `errors/`: Error handling
  - `mux/`: Multiplexing support
  - `protocol/`: Protocol-level utilities

### Configuration

- **infra/conf/**: Configuration loading and parsing
  - Supports JSON, YAML, and TOML formats
  - `infra/conf/serial/loader.go`: Config deserialization
  - Protocol-specific config files: `vless.go`, `vmess.go`, `trojan.go`, etc.
  - Transport config: `transport_internet.go`, `grpc.go`, `websocket.go`
- **infra/vprotogen/**: Protobuf generation tools
- **infra/vformat/**: Config format utilities

### Dependency Injection

The core uses reflection-based dependency injection. Features register themselves and declare dependencies, which the `Instance` type resolves at runtime. Check `core/xray.go` for the resolution logic.

## Configuration Files

Config files are in JSON/YAML/TOML format with this structure:
- `log`: Logging configuration
- `dns`: DNS server settings
- `inbounds`: Incoming connection handlers (protocols, listen addresses)
- `outbounds`: Outgoing connection handlers (proxy chains)
- `routing`: Traffic routing rules
- `policy`: Connection policies (timeouts, buffer sizes)
- `transport`: Transport-level settings (TLS, WebSocket, etc.)

**Auto-detection**: If no `-config` flag is provided, Xray searches for `config.json`, `config.jsonc`, `config.toml`, `config.yaml`, or `config.yml` in the working directory.

Example configs are in the repository root (config.json, client-advanced-tcp.json, etc.).

## Protobuf Code Generation

- **DO NOT** manually edit `*.pb.go` files
- Regenerate from `*.proto` files when schema changes
- Use protoc with the appropriate plugins for Go

## Code Style

- Go 1.25 codebase
- Use `gofmt` (tabs, standard Go formatting)
- Package names: lowercase, singular
- Exported symbols: PascalCase
- Unexported symbols: camelCase
- Config keys and JSON/TOML tags must remain stable for backward compatibility
- When adding new config fields, provide defaults and validation

## Testing Guidelines

- Place tests in `*_test.go` files alongside code
- Use table-driven tests where appropriate
- Mock definitions in `testing/mocks/`
- Integration/scenario tests in `testing/scenarios/`
- Include regression tests for parsing, protocol edge cases, and concurrency
- Coverage script: `testing/coverage/coverall` outputs to `out/xray/cov/coverage.txt`

## Protocol-Specific Notes

### VLESS
- VLESS is the primary protocol with XTLS and REALITY support
- REALITY provides TLS fingerprint masking to bypass censorship
- Implementation in `proxy/vless/`

### XTLS
- Direct TLS splice for improved performance
- Vision mode for enhanced security
- Used primarily with VLESS

### REALITY
- Transport-level implementation in `transport/internet/reality/`
- Requires specific server configuration (steal from target domain)
- See examples: https://github.com/XTLS/REALITY

## Common Development Patterns

### Adding a New Protocol
1. Create package in `proxy/[protocol-name]/`
2. Implement `proxy.Inbound` and/or `proxy.Outbound` interfaces
3. Add protobuf definitions for config
4. Create config parser in `infra/conf/[protocol-name].go`
5. Register protocol in `main/distro/all/`

### Adding a New Transport
1. Create package in `transport/internet/[transport-name]/`
2. Implement `Dial` and `Listen` functions
3. Add protobuf config definitions
4. Create config parser in `infra/conf/`
5. Register in transport initialization

### Modifying Config Schema
1. Update `*.proto` file
2. Regenerate `*.pb.go`
3. Update config parser in `infra/conf/`
4. Ensure backward compatibility or document breaking changes
5. Update example configs if needed

## Git Workflow

- Use Conventional Commit style: `fix(dns): description`, `feat(transport): description`
- Reference related issues/PRs in commit messages
- Run `go test ./...` and `go vet ./...` before pushing
- For breaking changes, document migration steps in PR description
