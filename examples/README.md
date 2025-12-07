# Xray 配置文件示例

本目录包含从简单到复杂的 Xray 配置文件示例，涵盖各种协议、传输方式和使用场景。

## 📚 快速导航

- [基础级别](#基础级别) - 入门配置
- [中级配置](#中级配置) - 加密和传输层
- [高级配置](#高级配置) - REALITY、XTLS Vision
- [专家级配置](#专家级配置) - 复杂场景
- [配置文件对照表](#配置文件对照表)

---

## 🌟 推荐配置

| 使用场景 | 推荐配置 | 特点 |
|---------|---------|------|
| 🏆 **首选方案** | `08-vless-reality-*` | 无需证书，最强抗审查 |
| ⚡ **性能优先** | `09-vless-xtls-vision-*` | 最高性能，需要证书 |
| 🌐 **CDN 中转** | `04-vless-ws-tls-*` | 隐藏 IP，稳定性好 |
| 🎮 **游戏/流媒体** | `09-vless-xtls-vision-*` | 低延迟，高带宽 |

---

## 基础级别

适合新手入门，理解基本概念。

### 01-socks-http-local.json

**本地代理入站配置**

```
功能：提供 SOCKS5 (1080) 和 HTTP (8080) 本地代理入站
用途：客户端入站，接收应用程序连接
难度：⭐
```

**特点**：
- ✅ 最简单的配置
- ✅ 仅本地监听（127.0.0.1）
- ✅ 直连模式，不需要服务器

**使用场景**：
- 本地开发测试
- 作为其他配置的入站模板

---

### 02-vless-tcp-basic-*

**VLESS over TCP 基础配置**

```
协议：VLESS
传输：TCP（原始）
加密：无
难度：⭐⭐
```

**文件**：
- `02-vless-tcp-basic-server.json` - 服务器端
- `02-vless-tcp-basic-client.json` - 客户端

**特点**：
- ✅ 最基础的 VLESS 配置
- ⚠️ 无加密，仅演示用途
- ✅ 理解协议基本结构

**⚠️ 警告**：不适合生产环境（无加密）

---

### 03-vmess-tcp-basic-*

**VMess over TCP 基础配置**

```
协议：VMess
传输：TCP
加密：VMess 内置加密
难度：⭐⭐
```

**文件**：
- `03-vmess-tcp-basic-server.json`
- `03-vmess-tcp-basic-client.json`

**特点**：
- ✅ VMess 协议自带加密
- ✅ 兼容性好
- ⚠️ 性能略低于 VLESS

**适用**：需要内置加密的场景

---

## 中级配置

添加传输层加密和路由功能。

### 04-vless-ws-tls-*

**VLESS + WebSocket + TLS**

```
协议：VLESS
传输：WebSocket
加密：TLS
难度：⭐⭐⭐
```

**文件**：
- `04-vless-ws-tls-server.json`
- `04-vless-ws-tls-client.json`

**特点**：
- ✅ 伪装成 HTTPS 流量
- ✅ 可通过 CDN（Cloudflare）
- ✅ 防火墙友好
- ⚠️ 需要域名和 TLS 证书

**使用场景**：
- CDN 中转（隐藏服务器 IP）
- 穿透企业防火墙
- 稳定性优先

**相关文档**：[使用场景 - CDN 中转](../docs/guide/use-cases.md#10-cdn-中转)

---

### 05-vmess-ws-tls-*

**VMess + WebSocket + TLS**

```
协议：VMess
传输：WebSocket
加密：TLS
难度：⭐⭐⭐
```

**特点**：
- ✅ VMess 双重加密（协议层 + TLS）
- ✅ 成熟稳定
- ⚠️ 性能开销较大

---

### 06-trojan-tls-*

**Trojan 协议**

```
协议：Trojan
传输：TCP
加密：TLS
特性：Fallback 伪装
难度：⭐⭐⭐
```

**特点**：
- ✅ 完美伪装成 HTTPS 网站
- ✅ 错误连接显示真实网站
- ✅ 密码认证，简单易用
- ⚠️ 需要域名和证书

**使用场景**：
- 需要网站伪装
- 密码管理比 UUID 更方便

---

### 07-vless-routing-client.json

**带路由规则的客户端**

```
功能：国内直连 + 国外代理 + 广告拦截
特性：DNS 分流
难度：⭐⭐⭐
```

**特点**：
- ✅ 自动分流（国内/国外）
- ✅ 广告拦截（geosite:category-ads-all）
- ✅ BT 直连
- ✅ DNS 防污染

**路由规则**：
```
广告域名 → 拦截
国内 IP/域名 → 直连
国外域名 → 代理
默认 → 代理
```

**相关文档**：[路由配置指南](../docs/guide/routing-guide.md)

---

## 高级配置

核心技术，重点推荐。

### 08-vless-reality-* ⭐ 重点

**VLESS + REALITY（最强抗审查）**

```
协议：VLESS
传输：TCP
加密：REALITY（伪装 TLS）
难度：⭐⭐
```

**文件**：
- `08-vless-reality-server.json`
- `08-vless-reality-client.json`

**核心特性**：
- 🏆 **无需域名和证书**
- 🏆 **伪装成真实网站**（如 microsoft.com）
- 🏆 **几乎无法被检测**
- 🏆 **抗主动探测**

**工作原理**：
```
客户端 → 伪装成访问 microsoft.com → REALITY 服务器
                    ↓
            验证密钥通过 → 代理服务
                    ↓
            验证失败 → 返回真实的 microsoft.com 网站
```

**使用场景**：
- 🔒 强审查地区（首选）
- 💰 无域名预算
- 🚀 快速部署

**详细指南**：[REALITY 完整指南](../docs/guide/reality-guide.md)

---

### 09-vless-xtls-vision-* ⭐ 重点

**VLESS + XTLS Vision（最高性能）**

```
协议：VLESS
传输：TCP
加密：TLS
特性：XTLS Vision
难度：⭐⭐⭐
```

**文件**：
- `09-vless-xtls-vision-server.json`
- `09-vless-xtls-vision-client.json`

**核心特性**：
- ⚡ **性能提升 2-3 倍**
- ⚡ **CPU 使用率降低 50%+**
- ⚡ **延迟降低 30-50%**
- ⚠️ 需要域名和证书

**性能对比**：
```
传统 TLS：     800 Mbps, CPU 35%
XTLS Vision： 950 Mbps, CPU 15% ✅
```

**使用场景**：
- 🎮 游戏加速（低延迟）
- 📺 流媒体 4K（高带宽）
- 💻 大文件传输
- ⚡ 性能优先

**详细指南**：[XTLS Vision 完整指南](../docs/guide/xtls-vision-guide.md)

---

### 10-vless-grpc-tls-*

**VLESS + gRPC + TLS**

```
协议：VLESS
传输：gRPC（基于 HTTP/2）
加密：TLS
难度：⭐⭐⭐
```

**特点**：
- ✅ 基于 HTTP/2
- ✅ 多路复用
- ✅ 可通过 CDN
- ⚠️ 配置复杂

**使用场景**：CDN 中转（替代 WebSocket）

---

### 11-vless-httpupgrade-*

**VLESS + HTTP Upgrade**

```
传输：HTTPUpgrade
特点：类似 WebSocket，兼容性更好
难度：⭐⭐⭐
```

---

### 12-vless-splithttp-*

**VLESS + Split HTTP**

```
传输：Split HTTP
特点：分段传输，适应严格环境
难度：⭐⭐⭐
```

---

## 专家级配置

复杂场景，高级功能。

### 13-multi-inbound-server.json

**多入站配置（多协议共存）**

```
入站：VLESS + VMess + Trojan + SOCKS
特性：Fallback 分流
难度：⭐⭐⭐⭐
```

**特点**：
- ✅ 单服务器支持多种协议
- ✅ 端口复用（都用 443）
- ✅ Fallback 到不同服务
- ✅ 统计和策略管理

**架构**：
```
443 端口
  ├── VLESS (主入站)
  │     ├── 合法连接 → 代理服务
  │     └── Fallback
  │           ├── /vmess-ws → VMess
  │           ├── /trojan-ws → Trojan
  │           └── 其他 → 伪装网站
  └── 内部端口
        ├── 8002: VMess WebSocket
        └── 8003: Trojan WebSocket
```

**使用场景**：
- 企业多用户管理
- 协议兼容性需求
- 端口受限环境

---

### 14-multi-outbound-client.json

**多出站 + 负载均衡**

```
出站：3个代理服务器
策略：leastPing（最低延迟）
特性：自动故障转移
难度：⭐⭐⭐⭐
```

**特点**：
- ✅ 负载均衡（选择最快服务器）
- ✅ 自动故障转移
- ✅ Observatory 健康检查
- ✅ 高可用部署

**工作原理**：
```
Observatory 每 1 分钟探测:
  服务器1: 50ms   ← 选择这个 ✅
  服务器2: 100ms
  服务器3: 超时（自动排除）
```

**使用场景**：
- 高可用需求
- 多服务器备份
- 企业网关

**相关文档**：[使用场景 - 高可用部署](../docs/guide/use-cases.md#11-高可用部署)

---

### 15-chain-proxy-client.json

**链式代理（多跳）**

```
架构：客户端 → 代理1 → 代理2 → 代理3 → 目标
用途：最高隐匿性
难度：⭐⭐⭐⭐
```

**特点**：
- ✅ 多重跳转，隐匿性极强
- ✅ 混合不同协议
- ⚠️ 延迟累积
- ⚠️ 配置复杂

**使用场景**：
- 隐私保护
- 匿名通信
- 规避追踪

---

### 16-fallback-server.json

**Fallback 伪装服务器**

```
功能：代理服务 + 网站伪装
特性：多协议 Fallback
难度：⭐⭐⭐⭐
```

**特点**：
- ✅ 对外看起来是普通网站
- ✅ 支持多种协议
- ✅ 防主动探测
- ✅ 端口复用

**架构**：
```
HTTPS (443)
  ├── 正确的 VLESS 连接 → 代理
  ├── /vless-ws 路径 → VLESS WebSocket
  ├── /vmess-ws 路径 → VMess WebSocket
  └── 其他所有连接 → Nginx 伪装网站 (端口 8080)
```

---

## 配置文件对照表

### 按协议分类

| 协议 | 配置文件 | 传输方式 | 难度 |
|------|---------|---------|------|
| **VLESS** | 02, 04, 07, 08*, 09*, 10, 11, 12, 13, 14, 15, 16 | 多种 | ⭐-⭐⭐⭐⭐ |
| **VMess** | 03, 05, 13 | TCP, WebSocket | ⭐⭐-⭐⭐⭐ |
| **Trojan** | 06, 13 | TCP + TLS | ⭐⭐⭐ |
| **SOCKS5** | 01, 13 | 本地 | ⭐ |
| **HTTP** | 01 | 本地 | ⭐ |

### 按传输方式分类

| 传输方式 | 配置文件 | 特点 |
|---------|---------|------|
| **TCP** | 02, 03, 06, 08*, 09* | 直连，低延迟 |
| **WebSocket** | 04, 05, 13, 16 | CDN 友好，伪装性好 |
| **gRPC** | 10 | HTTP/2，多路复用 |
| **HTTPUpgrade** | 11 | WebSocket 替代 |
| **SplitHTTP** | 12 | 严格环境 |
| **REALITY** | 08* | 伪装 TLS，无需证书 |

### 按使用场景分类

| 场景 | 推荐配置 |
|------|---------|
| 科学上网（抗审查） | **08-vless-reality-*** ⭐ |
| 流媒体/游戏（性能） | **09-vless-xtls-vision-*** ⭐ |
| CDN 中转 | 04-vless-ws-tls-* |
| 网站伪装 | 06-trojan-tls-*, 16-fallback-server |
| 高可用部署 | 14-multi-outbound-client |
| 企业多用户 | 13-multi-inbound-server |
| 隐私保护 | 15-chain-proxy-client |

---

## 🚀 快速开始

### 1. 选择配置

根据您的需求选择合适的配置文件：

```bash
# 初学者：从基础配置开始
cp examples/02-vless-tcp-basic-server.json config.json

# 推荐：REALITY（无需证书）
cp examples/08-vless-reality-server.json config.json

# 性能：XTLS Vision（需要证书）
cp examples/09-vless-xtls-vision-server.json config.json
```

### 2. 修改配置

**生成 UUID**：
```bash
xray uuid
# 或
uuidgen | tr '[:upper:]' '[:lower:]'
```

**REALITY 专用：生成密钥**：
```bash
xray x25519
```

**修改配置文件**：
- 替换 `your-server-ip.com` 为实际地址
- 替换 UUID 为生成的值
- 修改端口（如需要）

### 3. 运行 Xray

```bash
# 测试配置
xray -test -config=config.json

# 运行
xray -config=config.json

# 或使用 systemd
systemctl start xray
```

---

## 📖 相关文档

### 基础知识
- [网络基础](../docs/guide/network-basics.md) - TCP/IP、DNS、TLS 等
- [Xray 架构](../docs/guide/xray-architecture.md) - 核心组件和工作原理

### 协议和技术
- [协议对比](../docs/guide/protocols-comparison.md) - VLESS、VMess、Trojan 对比
- [REALITY 指南](../docs/guide/reality-guide.md) ⭐ - 最强抗审查技术
- [XTLS Vision 指南](../docs/guide/xtls-vision-guide.md) ⭐ - 最高性能技术

### 配置和应用
- [路由配置](../docs/guide/routing-guide.md) - 分流规则、负载均衡
- [使用场景](../docs/guide/use-cases.md) - 各种实际应用案例

---

## ⚠️ 注意事项

### 安全提醒

1. **不要共享配置**：UUID 和密钥是敏感信息
2. **定期更换密钥**：建议每 3-6 个月更换
3. **使用强密码**：Trojan 等协议的密码要足够复杂
4. **启用防火墙**：只开放必要端口

### 证书管理

需要证书的配置（04, 05, 06, 09, 10, 11, 12, 13, 16）：

```bash
# 使用 acme.sh 申请免费证书
curl https://get.acme.sh | sh
~/.acme.sh/acme.sh --issue -d your-domain.com --standalone

# 安装证书
~/.acme.sh/acme.sh --installcert -d your-domain.com \
  --key-file /path/to/privkey.pem \
  --fullchain-file /path/to/fullchain.pem
```

### 性能优化

```bash
# 启用 BBR（Linux）
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p

# 增加文件描述符限制
ulimit -n 65535
```

---

## 🆘 故障排查

### 连接失败

1. 检查防火墙
2. 验证 UUID 是否一致
3. 确认端口正确
4. 查看日志：`journalctl -u xray -f`

### 性能问题

1. 测试网络带宽
2. 尝试不同传输方式
3. 启用 BBR
4. 考虑使用 XTLS Vision

### 证书问题

1. 检查证书路径
2. 验证证书有效期
3. 确认域名解析

---

## 💡 最佳实践

1. **从简单开始**：先测试基础配置，再尝试高级功能
2. **阅读文档**：每个配置都有对应的详细文档
3. **测试验证**：修改后务必测试
4. **备份配置**：保存可用的配置文件
5. **监控日志**：定期检查运行状态

---

## 📞 获取帮助

- GitHub Issues: https://github.com/XTLS/Xray-core/issues
- Telegram: https://t.me/projectXray
- 官方文档: https://xtls.github.io/

---

**最后更新**：2025-12-07
