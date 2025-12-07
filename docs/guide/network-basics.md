# 网络基础知识

本文档介绍使用 Xray 所需的基础网络知识，从计算机网络原理到代理技术，由浅入深帮助您理解 Xray 的工作原理。

## 目录

- [TCP/IP 协议栈](#tcpip-协议栈)
- [DNS 域名解析](#dns-域名解析)
- [代理技术原理](#代理技术原理)
- [TLS/SSL 加密](#tlsssl-加密)
- [HTTP 协议](#http-协议)
- [WebSocket 协议](#websocket-协议)

---

## TCP/IP 协议栈

TCP/IP 是互联网的基础协议族，采用分层架构设计。

### 四层模型

```mermaid
graph TB
    A[应用层<br/>Application Layer<br/>HTTP, DNS, SOCKS等] --> B[传输层<br/>Transport Layer<br/>TCP, UDP]
    B --> C[网络层<br/>Network Layer<br/>IP, ICMP]
    C --> D[链路层<br/>Link Layer<br/>Ethernet, WiFi]

    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#ffe1f5
    style D fill:#e1ffe1
```

### TCP 三次握手

TCP 是面向连接的可靠传输协议，通过三次握手建立连接：

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Server as 服务器

    Client->>Server: SYN (seq=x)
    Note over Client,Server: 第一次握手：请求建立连接

    Server->>Client: SYN-ACK (seq=y, ack=x+1)
    Note over Client,Server: 第二次握手：同意建立连接

    Client->>Server: ACK (ack=y+1)
    Note over Client,Server: 第三次握手：确认连接建立

    Note over Client,Server: 连接建立，开始传输数据
```

**为什么需要三次握手？**
- 防止旧的重复连接请求导致混乱
- 确保双方都具备发送和接收能力
- 同步双方的初始序列号

### UDP 协议

UDP 是无连接的传输协议，特点：
- **速度快**：无需建立连接，延迟低
- **不可靠**：不保证数据到达，不保证顺序
- **开销小**：没有拥塞控制和流量控制

**使用场景**：
- 视频/音频流（少量丢包可接受）
- DNS 查询（单次请求响应）
- 在线游戏（实时性优先）
- Xray 的 UDP 代理（如 QUIC）

---

## DNS 域名解析

DNS（Domain Name System）将人类可读的域名转换为 IP 地址。

### DNS 查询过程

```mermaid
sequenceDiagram
    participant U as 用户设备
    participant L as 本地DNS
    participant R as 根DNS服务器
    participant T as 顶级域DNS
    participant A as 权威DNS

    U->>L: 查询 www.example.com
    L->>R: 查询根服务器
    R->>L: 返回 .com 服务器地址
    L->>T: 查询 .com 服务器
    T->>L: 返回 example.com 权威服务器
    L->>A: 查询 example.com
    A->>L: 返回 IP: 93.184.216.34
    L->>U: 返回结果
```

### DNS 污染与劫持

**DNS 污染**：
- 篡改 DNS 查询结果，返回错误的 IP 地址
- 导致无法访问特定网站

**Xray 的解决方案**：
1. **DNS over HTTPS (DoH)**：加密 DNS 查询
2. **DNS over TLS (DoT)**：TLS 加密通道
3. **分流 DNS**：国内外域名使用不同 DNS 服务器

---

## 代理技术原理

代理是介于客户端和目标服务器之间的中间服务器。

### 代理类型

#### 1. 正向代理（Forward Proxy）

客户端明确知道代理的存在，主动通过代理访问互联网。

```mermaid
graph LR
    A[客户端] -->|1. 请求代理| B[代理服务器]
    B -->|2. 转发请求| C[目标网站]
    C -->|3. 返回数据| B
    B -->|4. 转发数据| A

    style B fill:#ffd700
```

**特点**：
- 客户端配置代理地址
- 隐藏客户端真实 IP
- 可以突破网络限制
- **Xray 客户端就是正向代理**

#### 2. 反向代理（Reverse Proxy）

客户端不知道代理的存在，代理服务器代表后端服务器接收请求。

```mermaid
graph LR
    A[客户端] -->|请求| B[反向代理]
    B -->|分发| C[后端服务器1]
    B -->|分发| D[后端服务器2]
    B -->|分发| E[后端服务器3]

    style B fill:#87ceeb
```

**特点**：
- 负载均衡
- 隐藏后端服务器
- SSL 卸载
- CDN 就是反向代理的应用

#### 3. 透明代理（Transparent Proxy）

客户端不知道代理的存在，网关自动转发流量到代理。

```mermaid
graph LR
    A[客户端] -->|以为直连| B[透明代理<br/>网关]
    B -->|实际转发| C[互联网]

    style B fill:#90ee90
```

### 常见代理协议

#### SOCKS5

通用代理协议，支持 TCP 和 UDP：

```mermaid
sequenceDiagram
    participant C as 客户端
    participant P as SOCKS5代理
    participant S as 目标服务器

    C->>P: 握手请求（协议版本）
    P->>C: 握手响应（认证方法）
    C->>P: 认证信息
    P->>C: 认证成功
    C->>P: 连接请求（目标地址）
    P->>S: 建立连接
    S->>P: 连接成功
    P->>C: 连接成功响应

    Note over C,S: 开始传输数据
```

#### HTTP 代理

基于 HTTP CONNECT 方法的代理：

```
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
Proxy-Authorization: Basic dXNlcjpwYXNz

HTTP/1.1 200 Connection Established
```

---

## TLS/SSL 加密

TLS（Transport Layer Security）是保障网络通信安全的加密协议。

### TLS 握手过程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    C->>S: 1. Client Hello<br/>(支持的加密套件)
    S->>C: 2. Server Hello<br/>(选择的加密套件)
    S->>C: 3. Certificate<br/>(服务器证书)
    S->>C: 4. Server Hello Done

    C->>C: 验证证书
    C->>S: 5. Client Key Exchange<br/>(预主密钥)
    C->>S: 6. Change Cipher Spec
    C->>S: 7. Finished<br/>(加密握手消息)

    S->>C: 8. Change Cipher Spec
    S->>C: 9. Finished<br/>(加密握手消息)

    Note over C,S: 握手完成，开始加密通信
```

### 证书链验证

```mermaid
graph TB
    A[根证书 CA<br/>Root Certificate] -->|签名| B[中间证书<br/>Intermediate CA]
    B -->|签名| C[网站证书<br/>example.com]

    D[浏览器/系统] -.->|信任| A

    style A fill:#ff6b6b
    style B fill:#ffd93d
    style C fill:#6bcf7f
    style D fill:#4d96ff
```

### SNI（Server Name Indication）

在 TLS 握手时指定要访问的域名：

```
ClientHello:
  - TLS Version: 1.3
  - Server Name: www.example.com  ← SNI
  - Cipher Suites: [...]
```

**问题**：SNI 是明文传输的，可能被审查。

**Xray 的解决方案**：
- **REALITY**：伪装 SNI，模拟访问其他网站
- **ECH (Encrypted Client Hello)**：加密 SNI

---

## HTTP 协议

HTTP（HyperText Transfer Protocol）是应用层协议。

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| 特性 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 传输层 | TCP | TCP | QUIC (UDP) |
| 多路复用 | ❌ | ✅ | ✅ |
| 头部压缩 | ❌ | ✅ (HPACK) | ✅ (QPACK) |
| 服务器推送 | ❌ | ✅ | ✅ |
| 队头阻塞 | ✅ 严重 | ⚠️ TCP层仍有 | ✅ 无 |

### HTTP/2 多路复用

```mermaid
graph TB
    subgraph HTTP/1.1
        A1[请求1] --> B1[TCP连接1]
        A2[请求2] --> B2[TCP连接2]
        A3[请求3] --> B3[TCP连接3]
    end

    subgraph HTTP/2
        C1[请求1] --> D[单个TCP连接]
        C2[请求2] --> D
        C3[请求3] --> D
        D --> E[Stream 1]
        D --> F[Stream 2]
        D --> G[Stream 3]
    end
```

---

## WebSocket 协议

WebSocket 提供全双工通信通道，常用于 Xray 的传输层。

### WebSocket 握手

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    C->>S: HTTP/1.1 Upgrade Request<br/>Connection: Upgrade<br/>Upgrade: websocket
    S->>C: HTTP/1.1 101 Switching Protocols<br/>Connection: Upgrade<br/>Upgrade: websocket

    Note over C,S: WebSocket 连接建立

    C->>S: WebSocket Frame
    S->>C: WebSocket Frame
    C->>S: WebSocket Frame
```

### 为什么 Xray 使用 WebSocket？

1. **伪装性好**：看起来像普通 HTTPS 流量
2. **穿透能力强**：CDN 通常支持 WebSocket
3. **兼容性好**：大多数防火墙允许通过
4. **全双工通信**：双向同时传输数据

### WebSocket vs 原始 TCP

```mermaid
graph LR
    subgraph 原始 TCP
        A[代理数据] --> B[TCP 包]
    end

    subgraph WebSocket over TLS
        C[代理数据] --> D[WS Frame]
        D --> E[TLS 加密]
        E --> F[TCP 包]
        F --> G[看起来像HTTPS]
    end

    style G fill:#90ee90
```

---

## 总结

### 数据包的旅程

一个完整的 Xray VLESS over WebSocket + TLS 连接的数据流：

```mermaid
graph TD
    A[应用数据<br/>如 HTTP 请求] --> B[VLESS 协议封装<br/>添加 UUID 和指令]
    B --> C[WebSocket 帧<br/>添加 WS 头部]
    C --> D[TLS 加密<br/>加密整个 WS 数据]
    D --> E[TCP 分段<br/>添加 TCP 头部]
    E --> F[IP 数据包<br/>添加 IP 头部]
    F --> G[以太网帧<br/>添加 MAC 地址]
    G --> H[物理传输<br/>光纤/电磁波]

    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#ffe1f5
    style D fill:#e1ffe1
    style E fill:#ffe1e1
    style F fill:#f5e1ff
    style G fill:#e1fff5
```

### 关键概念回顾

| 概念 | 作用 | Xray 中的应用 |
|------|------|---------------|
| TCP | 可靠传输 | 大多数传输方式的基础 |
| UDP | 快速传输 | QUIC、mKCP |
| DNS | 域名解析 | DoH/DoT 防污染 |
| TLS | 加密通信 | 保护隐私，防止审查 |
| WebSocket | 双向通信 | 伪装成普通 HTTPS |
| HTTP/2 | 多路复用 | gRPC 传输 |

---

## 下一步

- 📖 阅读 [Xray 架构详解](xray-architecture.md)
- 🔒 了解 [REALITY 协议原理](reality-guide.md)
- ⚡ 学习 [XTLS Vision 技术](xtls-vision-guide.md)
- 🛣️ 配置 [路由分流规则](routing-guide.md)
