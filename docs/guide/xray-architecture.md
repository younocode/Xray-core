# Xray 架构详解

本文档深入介绍 Xray-core 的架构设计、核心组件和工作原理。

## 目录

- [架构概览](#架构概览)
- [核心组件](#核心组件)
- [数据流处理](#数据流处理)
- [配置结构](#配置结构)

---

## 架构概览

### 整体架构图

```mermaid
graph TB
    subgraph 入站 Inbounds
        I1[SOCKS5]
        I2[HTTP]
        I3[VLESS]
        I4[VMess]
    end

    subgraph 核心 Core
        R[路由器<br/>Router]
        D[调度器<br/>Dispatcher]
        DNS[DNS]
    end

    subgraph 出站 Outbounds
        O1[VLESS]
        O2[VMess]
        O3[Trojan]
        O4[Freedom<br/>直连]
        O5[Blackhole<br/>阻断]
    end

    I1 --> D
    I2 --> D
    I3 --> D
    I4 --> D

    D --> R
    R --> DNS
    R --> O1
    R --> O2
    R --> O3
    R --> O4
    R --> O5

    style R fill:#ffd700
    style D fill:#87ceeb
    style DNS fill:#90ee90
```

### 设计理念

Xray 采用**模块化设计**，核心特点：

1. **入站协议独立**：支持多种入站协议同时运行
2. **出站协议独立**：每个出站可使用不同协议和配置
3. **路由系统**：基于规则的灵活流量分发
4. **传输层分离**：协议层和传输层解耦

---

## 核心组件

### 1. 入站处理（Inbound）

入站负责接收客户端连接，解析协议并提取目标信息。

```mermaid
sequenceDiagram
    participant App as 应用程序
    participant In as 入站处理器
    participant Dis as 调度器

    App->>In: TCP/UDP 连接
    In->>In: 协议握手
    In->>In: 解析目标地址
    In->>Dis: 转发数据 + 元数据<br/>(目标、协议、用户)
```

**支持的入站协议**：
- **SOCKS**：SOCKS5 代理协议
- **HTTP**：HTTP/HTTPS 代理
- **VLESS**：轻量级协议
- **VMess**：加密传输协议
- **Trojan**：伪装协议
- **Dokodemo-door**：透明代理
- **Shadowsocks**：Shadowsocks 协议

### 2. 路由器（Router）

路由器根据规则决定数据流向哪个出站。

```mermaid
graph TD
    A[数据包] --> B{路由匹配}
    B -->|匹配规则1<br/>广告域名| C[Blackhole<br/>阻断]
    B -->|匹配规则2<br/>国内IP| D[Direct<br/>直连]
    B -->|匹配规则3<br/>国外域名| E[Proxy<br/>代理]
    B -->|默认规则| E

    style B fill:#ffd700
```

**路由规则类型**：
- **域名匹配**：domain、geosite
- **IP 匹配**：ip、geoip
- **端口匹配**：port、portRange
- **协议匹配**：protocol (如 bittorrent)
- **网络类型**：network (tcp/udp)
- **入站标签**：inboundTag
- **用户邮箱**：user

### 3. 调度器（Dispatcher）

调度器协调入站和出站，处理数据转发。

```mermaid
sequenceDiagram
    participant In as 入站
    participant Dis as 调度器
    participant Router as 路由器
    participant Out as 出站

    In->>Dis: 新连接 + 元数据
    Dis->>Router: 查询路由规则
    Router->>Dis: 返回出站标签
    Dis->>Out: 建立出站连接
    Out->>Dis: 连接成功

    loop 数据传输
        In->>Dis: 上行数据
        Dis->>Out: 转发上行
        Out->>Dis: 下行数据
        Dis->>In: 转发下行
    end
```

### 4. DNS 解析器

内置 DNS 解析器，支持分流和防污染。

```mermaid
graph TB
    A[DNS 查询] --> B{域名匹配}
    B -->|国内域名| C[国内 DNS<br/>223.5.5.5]
    B -->|国外域名| D[国外 DNS<br/>1.1.1.1]

    C --> E{IP 匹配}
    D --> E

    E -->|期望的 IP| F[返回结果]
    E -->|不匹配| G[丢弃，使用其他 DNS]

    style B fill:#ffd700
    style E fill:#87ceeb
```

**DNS 策略**：
- **AsIs**：使用系统 DNS
- **UseIP**：优先使用 IP，减少 DNS 查询
- **IPIfNonMatch**：路由无法匹配域名时才解析 IP
- **IPOnDemand**：按需解析

### 5. 出站处理（Outbound）

出站负责连接目标服务器或下一跳代理。

**出站类型**：
- **代理协议**：VLESS、VMess、Trojan、Shadowsocks、Wireguard
- **Freedom**：直连（可指定出口 IP）
- **Blackhole**：黑洞（丢弃流量）
- **DNS**：DNS 查询代理
- **Loopback**：环回到入站

---

## 数据流处理

### 完整数据流

```mermaid
graph LR
    A[浏览器] -->|HTTP 请求| B[SOCKS5 入站]
    B -->|解析目标| C[调度器]
    C -->|查询路由| D[路由器]
    D -->|选择出站| E[VLESS 出站]
    E -->|TLS + WS| F[远程服务器]
    F -->|转发| G[目标网站]

    G -.->|响应| F
    F -.->|TLS + WS| E
    E -.->|调度| C
    C -.->|SOCKS5| B
    B -.->|HTTP 响应| A

    style C fill:#ffd700
    style D fill:#87ceeb
```

### 传输层封装

Xray 的协议层和传输层分离，可以灵活组合。

```mermaid
graph TB
    subgraph 协议层
        P1[VLESS]
        P2[VMess]
        P3[Trojan]
    end

    subgraph 传输层
        T1[TCP]
        T2[mKCP]
        T3[WebSocket]
        T4[HTTP/2<br/>gRPC]
        T5[QUIC]
        T6[HTTPUpgrade]
    end

    subgraph 安全层
        S1[TLS]
        S2[REALITY]
        S3[None]
    end

    P1 --> T1
    P1 --> T3
    P2 --> T3
    P2 --> T4

    T1 --> S1
    T1 --> S2
    T3 --> S1
    T4 --> S1

    style P1 fill:#e1f5ff
    style S2 fill:#90ee90
```

**组合示例**：
- VLESS + TCP + TLS
- VLESS + TCP + REALITY
- VLESS + WebSocket + TLS
- VLESS + gRPC + TLS
- VMess + WebSocket + TLS

---

## 配置结构

### JSON 配置文件结构

```mermaid
graph TD
    A[config.json] --> B[log<br/>日志配置]
    A --> C[dns<br/>DNS配置]
    A --> D[inbounds<br/>入站配置数组]
    A --> E[outbounds<br/>出站配置数组]
    A --> F[routing<br/>路由配置]
    A --> G[policy<br/>策略配置]
    A --> H[transport<br/>传输配置]
    A --> I[stats<br/>统计配置]

    D --> D1[port 端口]
    D --> D2[protocol 协议]
    D --> D3[settings 协议设置]
    D --> D4[streamSettings 传输设置]

    E --> E1[tag 标签]
    E --> E2[protocol 协议]
    E --> E3[settings 协议设置]
    E --> E4[streamSettings 传输设置]

    F --> F1[domainStrategy]
    F --> F2[rules 规则数组]
    F --> F3[balancers 负载均衡]

    style A fill:#ffd700
    style F fill:#87ceeb
```

### 最小配置示例

**客户端**：
```json
{
  "inbounds": [{
    "port": 1080,
    "protocol": "socks"
  }],
  "outbounds": [{
    "protocol": "vless",
    "settings": {
      "vnext": [{
        "address": "server.com",
        "port": 443,
        "users": [{"id": "uuid"}]
      }]
    }
  }]
}
```

**服务端**：
```json
{
  "inbounds": [{
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{"id": "uuid"}]
    }
  }],
  "outbounds": [{
    "protocol": "freedom"
  }]
}
```

---

## 高级特性

### 1. Fallback 机制

当无法识别流量时，回退到其他服务（如网站）。

```mermaid
graph LR
    A[入站流量] --> B{识别协议}
    B -->|VLESS 协议| C[代理处理]
    B -->|非协议流量| D[Fallback]
    D --> E[伪装网站]

    style D fill:#ffd700
```

### 2. 链式代理

通过多个代理服务器转发流量。

```mermaid
graph LR
    A[客户端] --> B[代理1]
    B --> C[代理2]
    C --> D[代理3]
    D --> E[目标]

    style B fill:#e1f5ff
    style C fill:#fff4e1
    style D fill:#ffe1f5
```

### 3. 负载均衡

分发流量到多个出站，提高可用性和性能。

```mermaid
graph TB
    A[路由器] --> B[Balancer<br/>负载均衡器]
    B -->|leastPing| C[出站1<br/>延迟: 100ms]
    B -->|leastPing| D[出站2<br/>延迟: 50ms]
    B -->|leastPing| E[出站3<br/>延迟: 200ms]

    style D fill:#90ee90
    note[选择延迟最低的出站]
```

**负载均衡策略**：
- **random**：随机选择
- **leastPing**：选择延迟最低的（需要 Observatory）
- **leastLoad**：选择负载最低的

### 4. Observatory（观测器）

定期探测出站服务器的可用性和延迟。

```mermaid
sequenceDiagram
    participant O as Observatory
    participant S1 as 服务器1
    participant S2 as 服务器2
    participant S3 as 服务器3

    loop 每 1 分钟
        O->>S1: HTTPS 探测
        S1->>O: 200 OK (100ms)
        O->>S2: HTTPS 探测
        S2->>O: 200 OK (50ms)
        O->>S3: HTTPS 探测
        Note over S3: 超时
    end

    Note over O: 更新延迟数据<br/>服务器2最快
```

---

## 性能优化

### 1. 零拷贝（Zero Copy）

使用内存池和 buffer 复用减少内存分配。

### 2. 多路复用（Multiplexing）

单个连接承载多个数据流，减少握手开销。

```mermaid
graph LR
    A1[请求1] --> M[Mux<br/>多路复用]
    A2[请求2] --> M
    A3[请求3] --> M
    M --> B[单个TCP连接]

    style M fill:#ffd700
```

### 3. 连接复用

复用 TCP 连接，避免频繁建立连接。

---

## 安全特性

### 1. UUID 认证

每个用户使用唯一的 UUID 标识，防止未授权访问。

### 2. 时间验证

VMess 协议包含时间戳验证，防止重放攻击。

### 3. 流量混淆

通过 TLS、WebSocket 等传输层伪装流量特征。

### 4. 动态端口

可以配置动态修改端口，增加检测难度。

---

## 总结

### Xray 核心优势

| 特性 | 说明 |
|------|------|
| 🚀 高性能 | 零拷贝、连接复用、多路复用 |
| 🔒 安全性 | UUID 认证、时间验证、TLS 加密 |
| 🎭 伪装性 | REALITY、Fallback、多种传输层 |
| 🛣️ 灵活路由 | 强大的规则系统、负载均衡 |
| 🔧 可扩展 | 模块化设计、协议传输分离 |

### 架构对比

```mermaid
graph LR
    subgraph 传统代理
        A1[客户端] --> B1[单一协议]
        B1 --> C1[服务器]
    end

    subgraph Xray
        A2[客户端] --> B2[多入站]
        B2 --> C2[路由器]
        C2 --> D2[负载均衡]
        D2 --> E2[多出站]
        E2 --> F2[服务器集群]
    end

    style C2 fill:#ffd700
    style D2 fill:#87ceeb
```

---

## 下一步

- 📖 了解 [协议对比](protocols-comparison.md)
- 🔒 深入 [REALITY 技术](reality-guide.md)
- ⚡ 学习 [XTLS Vision](xtls-vision-guide.md)
- 🛣️ 配置 [路由规则](routing-guide.md)
