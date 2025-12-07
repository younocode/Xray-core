# Xray-core 全链路架构深度解析

本文档基于 VLESS + TCP 的最简配置，结合源码全面讲解 Xray-core 的工作原理、架构设计和数据流转过程。

---

## 目录

1. [配置文件示例](#1-配置文件示例)
2. [整体架构概览](#2-整体架构概览)
3. [启动流程详解](#3-启动流程详解)
4. [配置加载机制](#4-配置加载机制)
5. [核心运行时 Instance](#5-核心运行时-instance)
6. [Inbound/Outbound 管理](#6-inboundoutbound-管理)
7. [路由与调度系统](#7-路由与调度系统)
8. [VLESS 协议实现](#8-vless-协议实现)
9. [传输层抽象](#9-传输层抽象)
10. [完整数据流追踪](#10-完整数据流追踪)
11. [设计思考与总结](#11-设计思考与总结)

---

## 1. 配置文件示例

### 1.1 服务端配置 (server-simple.json)

```json
{
  "log": {
    "loglevel": "info"
  },
  "inbounds": [
    {
      "tag": "vless-in",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "a3482e88-686a-4a58-8126-99c9df64b060",
            "level": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp"
      }
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

### 1.2 客户端配置 (client-simple.json)

```json
{
  "log": {
    "loglevel": "info"
  },
  "inbounds": [
    {
      "tag": "socks-in",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "udp": true
      }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "your-server.com",
            "port": 443,
            "users": [
              {
                "id": "a3482e88-686a-4a58-8126-99c9df64b060",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp"
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    }
  ],
  "routing": {
    "rules": [
      {
        "type": "field",
        "domain": ["geosite:cn"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": ["geoip:cn", "geoip:private"],
        "outboundTag": "direct"
      }
    ]
  }
}
```

### 1.3 数据流向图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              客户端                                      │
│  ┌──────────┐    ┌────────────┐    ┌────────────┐    ┌──────────────┐  │
│  │ 浏览器   │───>│ SOCKS5     │───>│  Router    │───>│ VLESS        │  │
│  │ 应用程序 │    │ Inbound    │    │ (路由判断)  │    │ Outbound     │  │
│  └──────────┘    │ :10808     │    └────────────┘    │ (加密封装)   │  │
│                  └────────────┘                      └──────┬───────┘  │
└─────────────────────────────────────────────────────────────┼──────────┘
                                                              │
                                                              │ TCP 连接
                                                              │ VLESS 协议
                                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              服务端                                      │
│  ┌──────────────┐    ┌────────────┐    ┌────────────┐    ┌──────────┐  │
│  │ VLESS        │───>│ Dispatcher │───>│ Freedom    │───>│ 目标网站 │  │
│  │ Inbound      │    │ (请求分发)  │    │ Outbound   │    │          │  │
│  │ :443         │    └────────────┘    │ (直连)     │    └──────────┘  │
│  │ (解密解析)   │                      └────────────┘                  │
│  └──────────────┘                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 整体架构概览

### 2.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         入口层 (main/)                               │
│  main.go ──> run.go ──> commands/                                   │
│  处理命令行参数、启动流程                                              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         核心层 (core/)                               │
│  Instance: 运行时容器，管理所有 Feature 的生命周期                      │
│  依赖注入: RequireFeatures() 实现特性间的依赖解析                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   功能层 (app/) │   │  代理层 (proxy/)│   │ 传输层(transport)│
│                 │   │                 │   │                 │
│ • Dispatcher    │   │ • VLESS         │   │ • TCP/UDP       │
│ • Router        │   │ • VMess         │   │ • WebSocket     │
│ • DNS           │   │ • Trojan        │   │ • gRPC          │
│ • Proxyman      │   │ • Shadowsocks   │   │ • REALITY       │
│ • Policy        │   │ • Freedom       │   │ • TLS           │
│ • Stats         │   │ • Blackhole     │   │ • KCP           │
└─────────────────┘   └─────────────────┘   └─────────────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       公共层 (common/)                               │
│  buf/: 缓冲区管理  net/: 网络工具  errors/: 错误处理                   │
│  protocol/: 协议定义  crypto/: 加密工具  serial/: 序列化               │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心模块职责

| 模块 | 目录 | 职责 |
|------|------|------|
| **Instance** | `core/` | 运行时容器，管理所有 Feature 生命周期和依赖注入 |
| **Proxyman** | `app/proxyman/` | 管理 Inbound/Outbound Handler 的创建和启动 |
| **Dispatcher** | `app/dispatcher/` | 请求调度中心，创建数据管道，协调路由 |
| **Router** | `app/router/` | 路由规则匹配，决定请求发往哪个 Outbound |
| **Proxy** | `proxy/` | 各协议实现（VLESS、VMess、Trojan 等） |
| **Transport** | `transport/` | 传输层抽象（TCP、WebSocket、gRPC 等） |

---

## 3. 启动流程详解

### 3.1 启动入口

**文件**: `main/main.go:10-24`

```go
func main() {
    os.Args = getArgsV4Compatible()  // 兼容 v4 参数格式

    base.RootCommand.Long = "Xray is a platform for building proxies."
    base.RootCommand.Commands = append(
        []*base.Command{
            cmdRun,      // 核心：运行服务
            cmdVersion,  // 版本信息
        },
        base.RootCommand.Commands...,
    )
    base.Execute()  // 执行命令
}
```

### 3.2 Run 命令执行流程

**文件**: `main/run.go`

```go
func executeRun(cmd *base.Command, args []string) {
    // 1. 打印版本信息
    printVersion()

    // 2. 启动 Xray 服务器
    server, err := startXray()
    if err != nil {
        fmt.Println("Failed to start:", err)
        os.Exit(23)  // 配置错误
    }

    // 3. 配置测试模式
    if *test {
        fmt.Println("Configuration OK.")
        os.Exit(0)
    }

    // 4. 启动服务
    if err := server.Start(); err != nil {
        fmt.Println("Failed to start:", err)
        os.Exit(-1)
    }
    defer server.Close()

    // 5. 手动 GC 清理配置加载产生的临时对象
    runtime.GC()
    debug.FreeOSMemory()

    // 6. 等待中断信号
    osSignals := make(chan os.Signal, 1)
    signal.Notify(osSignals, os.Interrupt, syscall.SIGTERM)
    <-osSignals
}
```

### 3.3 启动流程图

```
main.go::main()
       │
       ▼
executeRun()
       │
       ├──> getConfigFilePath()        # 获取配置文件路径
       │    支持: config.json/jsonc/toml/yaml/yml
       │    支持: 多文件合并
       │
       ├──> core.LoadConfig()          # 加载并解析配置
       │         │
       │         ├──> 格式检测 (auto/json/toml/yaml)
       │         ├──> 读取文件内容
       │         ├──> JSON 解码 + 错误定位
       │         └──> 转换为 protobuf Config
       │
       ├──> core.New(config)           # 创建 Instance
       │         │
       │         ├──> 加载 App 特性 (DNS, Router, Policy...)
       │         ├──> 添加默认特性 (如果未配置)
       │         ├──> 初始化传输层系统拨号器
       │         ├──> addInboundHandlers()
       │         └──> addOutboundHandlers()
       │
       ├──> server.Start()             # 启动所有特性
       │         │
       │         ├──> Inbound Manager.Start()
       │         │    └──> 各 Inbound Worker 开始监听
       │         ├──> Outbound Manager.Start()
       │         ├──> Dispatcher.Start()
       │         ├──> Router.Start()
       │         └──> ... 其他特性
       │
       └──> 等待 SIGTERM/SIGINT 信号
```

---

## 4. 配置加载机制

### 4.1 配置加载链

**文件**: `core/config.go:60-110`

```go
func LoadConfig(formatName string, input interface{}) (*Config, error) {
    switch v := input.(type) {
    case cmdarg.Arg:  // 文件列表
        files := make([]*ConfigSource, len(v))

        for i, file := range v {
            var f string
            if formatName == "auto" {
                f = getFormat(file)  // 根据扩展名推断格式
            } else {
                f = formatName
            }
            files[i] = &ConfigSource{Name: file, Format: f}
        }

        // 使用 ConfigBuilder 构建最终配置
        return ConfigBuilderForFiles(files)
    }
}
```

### 4.2 JSON 配置解析

**文件**: `infra/conf/serial/loader.go:25-60`

```go
func DecodeJSONConfig(reader io.Reader) (*conf.Config, error) {
    jsonConfig := &conf.Config{}

    // 使用 TeeReader 同时读取和缓存，以便错误定位
    jsonContent := bytes.NewBuffer(make([]byte, 0, 10240))
    jsonReader := io.TeeReader(&json_reader.Reader{Reader: reader}, jsonContent)

    decoder := json.NewDecoder(jsonReader)
    if err := decoder.Decode(jsonConfig); err != nil {
        // 精确定位语法错误的行号和列号
        var pos *offset
        cause := errors.Cause(err)
        switch tErr := cause.(type) {
        case *json.SyntaxError:
            pos = findOffset(jsonContent.Bytes(), int(tErr.Offset))
        case *json.UnmarshalTypeError:
            pos = findOffset(jsonContent.Bytes(), int(tErr.Offset))
        }
        if pos != nil {
            return nil, errors.New("failed to read config at line ",
                pos.line, " char ", pos.char).Base(err)
        }
        return nil, errors.New("failed to read config").Base(err)
    }

    return jsonConfig, nil
}
```

### 4.3 协议配置注册表

**文件**: `infra/conf/xray.go:20-80`

```go
var (
    // Inbound 协议加载器
    inboundConfigLoader = NewJSONConfigLoader(ConfigCreatorCache{
        "dokodemo-door": func() interface{} { return new(DokodemoConfig) },
        "http":          func() interface{} { return new(HTTPServerConfig) },
        "socks":         func() interface{} { return new(SocksServerConfig) },
        "vless":         func() interface{} { return new(VLessInboundConfig) },
        "vmess":         func() interface{} { return new(VMessInboundConfig) },
        "trojan":        func() interface{} { return new(TrojanServerConfig) },
        // ... 更多协议
    }, "protocol", "settings")

    // Outbound 协议加载器
    outboundConfigLoader = NewJSONConfigLoader(ConfigCreatorCache{
        "blackhole":   func() interface{} { return new(BlackholeConfig) },
        "freedom":     func() interface{} { return new(FreedomConfig) },
        "vless":       func() interface{} { return new(VLessOutboundConfig) },
        "vmess":       func() interface{} { return new(VMessOutboundConfig) },
        // ... 更多协议
    }, "protocol", "settings")
)
```

### 4.4 配置转换流程

```
JSON 文件                  Go 结构体                   Protobuf Config
┌──────────────┐          ┌─────────────────┐         ┌─────────────────┐
│ {            │          │ conf.Config {   │         │ core.Config {   │
│   "inbounds":│──解码──>│   InboundConfigs│──Build()─>│   Inbound []    │
│   "outbounds"│          │   OutboundConf  │         │   Outbound []   │
│   "routing": │          │   RouterConfig  │         │   App []        │
│ }            │          │ }               │         │ }               │
└──────────────┘          └─────────────────┘         └─────────────────┘
```

---

## 5. 核心运行时 Instance

### 5.1 Instance 结构

**文件**: `core/xray.go:25-40`

```go
type Instance struct {
    statusLock                 sync.Mutex
    features                   []features.Feature     // 所有注册的特性
    pendingResolutions         []resolution           // 待解决的依赖
    pendingOptionalResolutions []resolution           // 可选依赖
    running                    bool
    resolveLock                sync.Mutex
    ctx                        context.Context
}
```

### 5.2 Feature 接口

**文件**: `features/feature.go`

```go
// Feature 是所有功能模块的基础接口
type Feature interface {
    common.HasType      // Type() interface{} - 返回类型标识
    common.Runnable     // Start() error; Close() error - 生命周期
}
```

### 5.3 依赖注入机制

**核心思想**: Feature 通过 `RequireFeatures()` 声明依赖，Instance 在所有依赖就位时自动调用回调。

**文件**: `core/xray.go:130-180`

```go
// RequireFeatures 注册依赖回调
func (s *Instance) RequireFeatures(callback interface{}, optional bool) error {
    callbackType := reflect.TypeOf(callback)

    // 通过反射获取回调函数的参数类型（即依赖的 Feature 类型）
    var featureTypes []reflect.Type
    for i := 0; i < callbackType.NumIn(); i++ {
        featureTypes = append(featureTypes, reflect.PtrTo(callbackType.In(i)))
    }

    r := resolution{deps: featureTypes, callback: callback}

    // 检查是否所有依赖都已就位
    s.resolveLock.Lock()
    foundAll := true
    for _, d := range r.deps {
        if getFeature(s.features, d) == nil {
            foundAll = false
            break
        }
    }

    if foundAll {
        s.resolveLock.Unlock()
        return r.callbackResolution(s.features)  // 立即执行
    }

    // 否则加入等待队列
    s.pendingResolutions = append(s.pendingResolutions, r)
    s.resolveLock.Unlock()
    return nil
}
```

**使用示例**:

```go
// Dispatcher 依赖 Router 和 Outbound Manager
core.RequireFeatures(ctx, func(router routing.Router, ohm outbound.Manager) error {
    d.router = router
    d.ohm = ohm
    return nil
})
```

### 5.4 Instance 初始化流程

**文件**: `core/xray.go:45-120`

```go
func initInstanceWithConfig(config *Config, server *Instance) (bool, error) {
    // 1. 加载配置中的 App 特性
    for _, appSettings := range config.App {
        settings, _ := appSettings.GetInstance()
        obj, _ := CreateObject(server, settings)
        if feature, ok := obj.(features.Feature); ok {
            server.AddFeature(feature)
        }
    }

    // 2. 添加必要的默认特性（如果未配置）
    essentialFeatures := []struct {
        Type     interface{}
        Instance features.Feature
    }{
        {dns.ClientType(), localdns.New()},
        {policy.ManagerType(), policy.DefaultManager{}},
        {routing.RouterType(), routing.DefaultRouter{}},
        {stats.ManagerType(), stats.NoopManager{}},
    }

    for _, f := range essentialFeatures {
        if server.GetFeature(f.Type) == nil {
            server.AddFeature(f.Instance)
        }
    }

    // 3. 初始化传输层系统拨号器
    internet.InitSystemDialer(
        server.GetFeature(dns.ClientType()).(dns.Client),
        func() outbound.Manager { ... }(),
    )

    // 4. 添加 Inbound 和 Outbound 处理器
    addInboundHandlers(server, config.Inbound)
    addOutboundHandlers(server, config.Outbound)

    return false, nil
}
```

---

## 6. Inbound/Outbound 管理

### 6.1 架构图

```
┌───────────────────────────────────────────────────────────────────┐
│                     Proxyman (app/proxyman/)                      │
│                                                                   │
│  ┌─────────────────────────┐   ┌─────────────────────────┐       │
│  │   Inbound Manager       │   │   Outbound Manager      │       │
│  │                         │   │                         │       │
│  │  ┌───────────────────┐  │   │  ┌───────────────────┐  │       │
│  │  │ taggedHandlers    │  │   │  │ taggedHandler     │  │       │
│  │  │ map[string]Handler│  │   │  │ map[string]Handler│  │       │
│  │  └───────────────────┘  │   │  └───────────────────┘  │       │
│  │                         │   │                         │       │
│  │  ┌───────────────────┐  │   │  ┌───────────────────┐  │       │
│  │  │ untaggedHandlers  │  │   │  │ defaultHandler    │  │       │
│  │  │ []Handler         │  │   │  │ (第一个 Outbound)  │  │       │
│  │  └───────────────────┘  │   │  └───────────────────┘  │       │
│  └─────────────────────────┘   └─────────────────────────┘       │
│                                                                   │
│  每个 Handler 内部:                                               │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Handler                                                      │ │
│  │  ├─ tag: string                                             │ │
│  │  ├─ proxy: proxy.Inbound/Outbound (协议实现)                 │ │
│  │  ├─ streamSettings: 传输层配置                               │ │
│  │  └─ worker: tcpWorker/udpWorker (Inbound 监听)              │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

### 6.2 Inbound Manager

**文件**: `app/proxyman/inbound/inbound.go:20-80`

```go
type Manager struct {
    access           sync.RWMutex
    untaggedHandlers []inbound.Handler      // 无标签处理器
    taggedHandlers   map[string]inbound.Handler  // 有标签处理器
    running          bool
}

func (m *Manager) AddHandler(ctx context.Context, handler inbound.Handler) error {
    m.access.Lock()
    defer m.access.Unlock()

    tag := handler.Tag()
    if len(tag) > 0 {
        if _, found := m.taggedHandlers[tag]; found {
            return errors.New("existing tag: " + tag)
        }
        m.taggedHandlers[tag] = handler
    } else {
        m.untaggedHandlers = append(m.untaggedHandlers, handler)
    }

    if m.running {
        return handler.Start()  // 热添加时立即启动
    }
    return nil
}

func (m *Manager) Start() error {
    m.access.Lock()
    defer m.access.Unlock()

    m.running = true

    // 启动所有处理器
    for _, handler := range m.taggedHandlers {
        if err := handler.Start(); err != nil {
            return err
        }
    }
    for _, handler := range m.untaggedHandlers {
        if err := handler.Start(); err != nil {
            return err
        }
    }
    return nil
}
```

### 6.3 Inbound Worker（TCP 监听）

**文件**: `app/proxyman/inbound/worker.go:30-120`

```go
type tcpWorker struct {
    address         net.Address
    port            net.Port
    proxy           proxy.Inbound          // 协议实现 (如 VLESS)
    stream          *internet.MemoryStreamConfig
    tag             string
    dispatcher      routing.Dispatcher
    sniffingConfig  *proxyman.SniffingConfig
    hub             internet.Listener       // TCP 监听器
}

func (w *tcpWorker) Start() error {
    // 创建 TCP 监听器
    hub, err := internet.ListenTCP(w.ctx, w.address, w.port, w.stream, w.callback)
    if err != nil {
        return err
    }
    w.hub = hub
    return nil
}

// callback 处理每个新连接
func (w *tcpWorker) callback(conn stat.Connection) {
    ctx, cancel := context.WithCancel(w.ctx)
    sid := session.NewID()
    ctx = c.ContextWithID(ctx, sid)

    // 设置入站会话信息
    ctx = session.ContextWithInbound(ctx, &session.Inbound{
        Source:  net.DestinationFromAddr(conn.RemoteAddr()),
        Gateway: net.DestinationFromAddr(conn.LocalAddr()),
        Tag:     w.tag,
    })

    // 调用协议的 Process 方法处理连接
    if err := w.proxy.Process(ctx, conn.RemoteAddr().Network(), conn, w.dispatcher); err != nil {
        errors.LogInfoInner(ctx, err, "connection ends")
    }
    cancel()
}
```

### 6.4 Outbound Manager

**文件**: `app/proxyman/outbound/outbound.go:20-80`

```go
type Manager struct {
    access          sync.RWMutex
    defaultHandler  outbound.Handler           // 默认处理器
    taggedHandler   map[string]outbound.Handler
    untaggedHandlers []outbound.Handler
    running         bool
}

func (m *Manager) AddHandler(ctx context.Context, handler outbound.Handler) error {
    m.access.Lock()
    defer m.access.Unlock()

    // 第一个添加的成为默认处理器
    if m.defaultHandler == nil {
        m.defaultHandler = handler
    }

    tag := handler.Tag()
    if len(tag) > 0 {
        m.taggedHandler[tag] = handler
    } else {
        m.untaggedHandlers = append(m.untaggedHandlers, handler)
    }

    if m.running {
        return handler.Start()
    }
    return nil
}

func (m *Manager) GetHandler(tag string) outbound.Handler {
    m.access.RLock()
    defer m.access.RUnlock()

    if handler, found := m.taggedHandler[tag]; found {
        return handler
    }
    return nil
}

func (m *Manager) GetDefaultHandler() outbound.Handler {
    m.access.RLock()
    defer m.access.RUnlock()
    return m.defaultHandler
}
```

---

## 7. 路由与调度系统

### 7.1 Router-Dispatcher 协作图

```
请求到达
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Dispatcher                                  │
│                                                                 │
│  1. 创建 Link 对 (Pipe)     ┌─────────────────────────────────┐ │
│     ┌───────────┐           │         Router                  │ │
│     │ uplink    │           │                                 │ │
│     │ downlink  │──2.查询──>│  Domain Rules  ──────┐         │ │
│     └───────────┘           │  IP Rules      ──────┼─匹配──>  │ │
│                             │  Protocol Rules ─────┘    │    │ │
│  4. 转发请求给 Handler      │                           │    │ │
│                             │  3. 返回 OutboundTag <────┘    │ │
│                             └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Outbound Handler (根据 tag 查找)
    │
    ▼
远程服务器
```

### 7.2 Dispatcher 实现

**文件**: `app/dispatcher/default.go:50-150`

```go
type DefaultDispatcher struct {
    ohm    outbound.Manager    // 出站管理器
    router routing.Router      // 路由器
    policy policy.Manager      // 策略管理器
    stats  stats.Manager       // 统计管理器
}

func (d *DefaultDispatcher) Dispatch(ctx context.Context, destination net.Destination) (*transport.Link, error) {
    if !destination.IsValid() {
        panic("Dispatcher: Invalid destination.")
    }

    // 获取或创建出站栈
    outbounds := session.OutboundsFromContext(ctx)
    if len(outbounds) == 0 {
        outbounds = []*session.Outbound{{}}
        ctx = session.ContextWithOutbounds(ctx, outbounds)
    }

    ob := outbounds[len(outbounds)-1]
    ob.OriginalTarget = destination
    ob.Target = destination

    // 创建数据管道对
    inbound, outbound := d.getLink(ctx)

    content := session.ContentFromContext(ctx)
    sniffingRequest := content.SniffingRequest

    if !sniffingRequest.Enabled {
        // 不嗅探，直接路由分发
        go d.routedDispatch(ctx, outbound, destination)
    } else {
        // 启用协议嗅探
        go func() {
            cReader := &cachedReader{reader: outbound.Reader.(*pipe.Reader)}
            outbound.Reader = cReader

            result, err := sniffer(ctx, cReader, sniffingRequest.MetadataOnly, destination.Network)
            if err == nil {
                content.Protocol = result.Protocol()
                if d.shouldOverride(ctx, result, sniffingRequest, destination) {
                    domain := result.Domain()
                    destination.Address = net.ParseAddress(domain)
                }
            }
            d.routedDispatch(ctx, outbound, destination)
        }()
    }

    return inbound, nil
}
```

### 7.3 Link 管道创建

**文件**: `app/dispatcher/default.go:200-250`

```go
func (d *DefaultDispatcher) getLink(ctx context.Context) (*transport.Link, *transport.Link) {
    opt := pipe.OptionsFromContext(ctx)

    // 创建两对管道
    uplinkReader, uplinkWriter := pipe.New(opt...)      // 客户端 -> 服务器
    downlinkReader, downlinkWriter := pipe.New(opt...)  // 服务器 -> 客户端

    // Inbound Link: 返回给调用者
    inboundLink := &transport.Link{
        Reader: downlinkReader,   // 从下行管道读（服务器的响应）
        Writer: uplinkWriter,     // 写入上行管道（客户端的请求）
    }

    // Outbound Link: 传给 Outbound Handler
    outboundLink := &transport.Link{
        Reader: uplinkReader,     // 从上行管道读（客户端的请求）
        Writer: downlinkWriter,   // 写入下行管道（服务器的响应）
    }

    return inboundLink, outboundLink
}
```

**管道数据流向**:

```
                     上行管道 (uplink)
客户端请求 ────> uplinkWriter ────────────> uplinkReader ────> 服务器请求

                     下行管道 (downlink)
客户端响应 <──── downlinkReader <─────────── downlinkWriter <── 服务器响应

Inbound Link:                    Outbound Link:
  Reader = downlinkReader          Reader = uplinkReader
  Writer = uplinkWriter            Writer = downlinkWriter
```

### 7.4 路由分发

**文件**: `app/dispatcher/default.go:160-200`

```go
func (d *DefaultDispatcher) routedDispatch(ctx context.Context, link *transport.Link, destination net.Destination) {
    // 1. 路由查询
    ob := session.OutboundsFromContext(ctx)
    route, _ := d.router.PickRoute(routing.AsRoutingContext(ctx))

    var handler outbound.Handler
    if route != nil {
        tag := route.GetOutboundTag()
        if h := d.ohm.GetHandler(tag); h != nil {
            handler = h
            ob[len(ob)-1].ReverseMode = tag == "reverse"
        }
    }

    if handler == nil {
        handler = d.ohm.GetDefaultHandler()  // 使用默认处理器
    }

    if handler == nil {
        errors.LogInfo(ctx, "no available handler")
        common.Interrupt(link.Writer)
        return
    }

    // 2. 调用 Outbound Handler 处理
    handler.Dispatch(ctx, link)
}
```

### 7.5 Router 规则匹配

**文件**: `app/router/router.go:100-150`

```go
type Router struct {
    domainStrategy  Config_DomainStrategy
    rules           []*Rule
    balancers       map[string]*Balancer
    dns             dns.Client
}

func (r *Router) PickRoute(ctx routing.Context) (routing.Route, error) {
    // 遍历所有规则进行匹配
    for _, rule := range r.rules {
        if rule.Apply(ctx) {
            return &Route{
                outboundTag: rule.Tag,
                ruleTag:     rule.RuleTag,
            }, nil
        }
    }
    return nil, common.ErrNoClue
}
```

---

## 8. VLESS 协议实现

### 8.1 VLESS 协议格式

```
VLESS 请求格式:
┌──────┬────────────────┬─────────┬──────────┬──────────────┬─────────────┐
│ Ver  │      UUID      │ Addons  │ Command  │  Destination │   Payload   │
│ 1B   │      16B       │  ?B     │   1B     │     ?B       │   Stream    │
└──────┴────────────────┴─────────┴──────────┴──────────────┴─────────────┘

Ver:     版本号 (0x00)
UUID:    用户身份验证 (16 字节)
Addons:  扩展信息 (长度+内容, 用于 XTLS Vision 等)
Command: 命令类型
         0x01 = TCP
         0x02 = UDP
         0x03 = Mux (多路复用)
Dest:    目标地址 (ATYP + Address + Port)
         ATYP: 0x01=IPv4, 0x02=Domain, 0x03=IPv6
Payload: 原始数据流

VLESS 响应格式:
┌──────┬─────────┬─────────────┐
│ Ver  │ Addons  │   Payload   │
│ 1B   │  ?B     │   Stream    │
└──────┴─────────┴─────────────┘
```

### 8.2 VLESS Inbound Handler

**文件**: `proxy/vless/inbound/inbound.go:50-150`

```go
type Handler struct {
    inboundHandlerManager  feature_inbound.Manager
    policyManager          policy.Manager
    validator              vless.Validator     // UUID 验证器
    decryption             *encryption.ServerInstance
    outboundHandlerManager outbound.Manager
    fallbacks              map[string]map[string]map[string]*Fallback
}

func (h *Handler) Process(ctx context.Context, network net.Network,
    connection stat.Connection, dispatcher routing.Dispatcher) error {

    // 1. 读取并解析请求头
    first := buf.FromBytes(firstBytes)
    requestAddons, request, isfb, err := encoding.DecodeRequestHeader(
        isfb, first, reader, h.validator)
    if err != nil {
        // 可能是回落请求
        return h.fallback(ctx, err, sessionPolicy, connection, isfb, first)
    }

    // 2. 验证用户
    if request.User == nil {
        return errors.New("invalid user")
    }

    // 3. 提取目标地址
    dest := request.Destination()

    // 4. 调用 Dispatcher 分发请求
    link, err := dispatcher.Dispatch(ctx, dest)
    if err != nil {
        return err
    }

    // 5. 写入响应头
    responseAddons := &encoding.Addons{}
    if err := encoding.EncodeResponseHeader(conn, request, responseAddons); err != nil {
        return err
    }

    // 6. 双向数据转发
    // 客户端 -> 服务器
    requestDone := func() error {
        return buf.Copy(clientReader, link.Writer)
    }

    // 服务器 -> 客户端
    responseDone := func() error {
        return buf.Copy(link.Reader, clientWriter)
    }

    // 并行执行
    task.Run(ctx, requestDone, responseDone)

    return nil
}
```

### 8.3 VLESS Outbound Handler

**文件**: `proxy/vless/outbound/outbound.go:50-150`

```go
type Handler struct {
    server        *protocol.ServerSpec
    policyManager policy.Manager
    cone          bool
    encryption    *encryption.ClientInstance
}

func (h *Handler) Process(ctx context.Context, link *transport.Link,
    dialer internet.Dialer) error {

    outbounds := session.OutboundsFromContext(ctx)
    ob := outbounds[len(outbounds)-1]
    target := ob.Target

    // 1. 建立到服务器的连接
    var conn stat.Connection
    if h.testpre > 0 {
        conn = h.getPreConn()  // 使用预连接
    }
    if conn == nil {
        rawConn, err := dialer.Dial(ctx, h.server.Destination())
        if err != nil {
            return err
        }
        conn = rawConn
    }
    defer conn.Close()

    // 2. 获取用户信息
    user := h.server.User
    account := user.Account.(*vless.MemoryAccount)

    // 3. 构造 VLESS 请求
    request := &protocol.RequestHeader{
        Version: encoding.Version,
        User:    user,
        Command: protocol.RequestCommandTCP,
        Address: target.Address,
        Port:    target.Port,
    }

    requestAddons := &encoding.Addons{Flow: account.Flow}

    // 4. 发送请求头
    if err := encoding.EncodeRequestHeader(conn, request, requestAddons); err != nil {
        return err
    }

    // 5. 双向数据转发
    // 本地 -> 远程服务器
    requestDone := func() error {
        return buf.Copy(link.Reader, conn)
    }

    // 远程服务器 -> 本地
    responseDone := func() error {
        // 先读取响应头
        responseAddons, err := encoding.DecodeResponseHeader(conn, request)
        if err != nil {
            return err
        }
        return buf.Copy(conn, link.Writer)
    }

    task.Run(ctx, requestDone, responseDone)

    return nil
}
```

### 8.4 VLESS 编解码

**文件**: `proxy/vless/encoding/encoding.go:30-100`

```go
const Version = byte(0)

// EncodeRequestHeader 编码请求头
func EncodeRequestHeader(writer io.Writer, request *protocol.RequestHeader,
    requestAddons *Addons) error {

    buffer := buf.StackNew()
    defer buffer.Release()

    // 1. 版本号 (1 字节)
    buffer.WriteByte(request.Version)

    // 2. UUID (16 字节)
    buffer.Write(request.User.Account.(*vless.MemoryAccount).ID.Bytes())

    // 3. 扩展信息
    EncodeHeaderAddons(&buffer, requestAddons)

    // 4. 命令 (1 字节)
    buffer.WriteByte(byte(request.Command))

    // 5. 目标地址 (如果不是 Mux)
    if request.Command != protocol.RequestCommandMux {
        addrParser.WriteAddressPort(&buffer, request.Address, request.Port)
    }

    _, err := writer.Write(buffer.Bytes())
    return err
}

// DecodeRequestHeader 解码请求头
func DecodeRequestHeader(isfb bool, first *buf.Buffer, reader io.Reader,
    validator vless.Validator) ([]byte, *protocol.RequestHeader, *Addons, bool, error) {

    request := new(protocol.RequestHeader)

    // 1. 读取版本
    request.Version = first.Byte(0)

    // 2. 读取并验证 UUID
    var id [16]byte
    copy(id[:], first.BytesRange(1, 17))
    request.User = validator.Get(id)
    if request.User == nil {
        return nil, nil, nil, false, errors.New("invalid user")
    }

    // 3. 解析扩展信息
    addons := new(Addons)
    DecodeHeaderAddons(first, reader, addons)

    // 4. 读取命令
    request.Command = protocol.RequestCommand(cmdByte)

    // 5. 读取目标地址
    request.Address, request.Port, _ = addrParser.ReadAddressPort(first, reader)

    return nil, request, addons, false, nil
}
```

---

## 9. 传输层抽象

### 9.1 传输层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Transport 抽象层                                 │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │   Dialer    │  │  Listener   │  │    Link     │                 │
│  │ (出站连接)  │  │ (入站监听)  │  │ (数据管道)  │                 │
│  └─────────────┘  └─────────────┘  └─────────────┘                 │
│                                                                     │
│  实现:                                                              │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐                │
│  │ TCP  │ UDP  │  WS  │ gRPC │ KCP  │ HTTP │ TLS  │                │
│  │      │      │      │      │      │ Upgr │      │                │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘                │
│                                                                     │
│  特殊传输:                                                          │
│  ┌────────────────────┬────────────────────┐                       │
│  │      REALITY       │     SplitHTTP      │                       │
│  │  (TLS 指纹伪装)     │   (HTTP 分片)      │                       │
│  └────────────────────┴────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 Link 和 Pipe

**文件**: `transport/transport.go`

```go
// Link 代表数据流的双向通道
type Link struct {
    Reader buf.Reader   // 读取数据
    Writer buf.Writer   // 写入数据
}
```

**文件**: `transport/pipe/pipe.go:20-80`

```go
type pipe struct {
    readSignal  *signal.Notifier
    writeSignal *signal.Notifier
    done        *done.Instance
    errChan     chan error
    option      pipeOption
    data        buf.MultiBuffer
    access      sync.Mutex
}

// New 创建一对连接的读写管道
func New(opts ...Option) (*Reader, *Writer) {
    p := &pipe{
        readSignal:  signal.NewNotifier(),
        writeSignal: signal.NewNotifier(),
        done:        done.New(),
        errChan:     make(chan error, 1),
        option:      pipeOption{limit: -1},
    }

    for _, opt := range opts {
        opt(&(p.option))
    }

    return &Reader{pipe: p}, &Writer{pipe: p}
}

// Reader 从管道读取数据
type Reader struct {
    pipe *pipe
}

func (r *Reader) ReadMultiBuffer() (buf.MultiBuffer, error) {
    r.pipe.access.Lock()
    for r.pipe.data.IsEmpty() {
        r.pipe.access.Unlock()

        select {
        case <-r.pipe.readSignal.Wait():
            r.pipe.access.Lock()
        case err := <-r.pipe.errChan:
            return nil, err
        case <-r.pipe.done.Wait():
            return nil, io.EOF
        }
    }

    mb := r.pipe.data
    r.pipe.data = nil
    r.pipe.access.Unlock()

    r.pipe.writeSignal.Signal()
    return mb, nil
}
```

### 9.3 TCP 传输实现

**文件**: `transport/internet/tcp/dialer.go`

```go
func Dial(ctx context.Context, dest net.Destination,
    streamSettings *internet.MemoryStreamConfig) (stat.Connection, error) {

    // 获取 TCP 配置
    tcpSettings := streamSettings.ProtocolSettings.(*Config)

    // 创建系统拨号器
    conn, err := internet.DialSystem(ctx, dest, streamSettings.SocketSettings)
    if err != nil {
        return nil, err
    }

    // 如果配置了 TLS
    if config := tls.ConfigFromStreamSettings(streamSettings); config != nil {
        conn = tls.Client(conn, config.GetTLSConfig())
    }

    return stat.Connection(conn), nil
}
```

**文件**: `transport/internet/tcp/hub.go`

```go
func ListenTCP(ctx context.Context, address net.Address, port net.Port,
    streamSettings *internet.MemoryStreamConfig, handler internet.ConnHandler) (internet.Listener, error) {

    listener, err := internet.ListenSystem(ctx, &net.TCPAddr{
        IP:   address.IP(),
        Port: int(port),
    }, streamSettings.SocketSettings)
    if err != nil {
        return nil, err
    }

    l := &Listener{
        listener: listener,
        config:   streamSettings.ProtocolSettings.(*Config),
    }

    // 异步接受连接
    go l.keepAccepting(ctx, handler)

    return l, nil
}

func (l *Listener) keepAccepting(ctx context.Context, handler internet.ConnHandler) {
    for {
        conn, err := l.listener.Accept()
        if err != nil {
            return
        }

        // 如果配置了 TLS
        if tlsConfig := tls.ConfigFromStreamSettings(l.streamSettings); tlsConfig != nil {
            conn = tls.Server(conn, tlsConfig.GetTLSConfig())
        }

        // 调用连接处理回调
        handler(stat.Connection(conn))
    }
}
```

---

## 10. 完整数据流追踪

### 10.1 客户端请求流程

以浏览器访问 `https://example.com` 为例:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 第 1 步: 浏览器 -> SOCKS5 Inbound                                        │
│                                                                         │
│ 浏览器发起 SOCKS5 连接到 127.0.0.1:10808                                 │
│   ↓                                                                     │
│ tcpWorker.callback() 被调用                                             │
│   ↓                                                                     │
│ 创建 Session Context:                                                   │
│   - Inbound.Source = 127.0.0.1:xxxxx                                   │
│   - Inbound.Tag = "socks-in"                                           │
│   ↓                                                                     │
│ socks.Handler.Process() 解析 SOCKS5 请求:                               │
│   - 目标地址: example.com:443                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第 2 步: SOCKS5 Handler -> Dispatcher                                    │
│                                                                         │
│ socks.Handler 调用:                                                     │
│   link, err := dispatcher.Dispatch(ctx, net.Destination{               │
│       Address: "example.com",                                          │
│       Port: 443,                                                       │
│       Network: net.Network_TCP,                                        │
│   })                                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第 3 步: Dispatcher 创建 Link 并路由                                     │
│                                                                         │
│ Dispatcher.Dispatch():                                                  │
│   1. 创建 Pipe 对:                                                      │
│      uplinkReader, uplinkWriter = pipe.New()                           │
│      downlinkReader, downlinkWriter = pipe.New()                       │
│                                                                         │
│   2. 查询路由:                                                          │
│      route = router.PickRoute(ctx)                                     │
│      → 匹配规则: domain:example.com 不在 geosite:cn                     │
│      → 返回: outboundTag = "proxy"                                     │
│                                                                         │
│   3. 获取 Outbound Handler:                                            │
│      handler = ohm.GetHandler("proxy")  // VLESS Outbound              │
│                                                                         │
│   4. 调用 Handler.Dispatch(outboundLink)                               │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第 4 步: VLESS Outbound 处理                                             │
│                                                                         │
│ vless.Handler.Process():                                                │
│   1. 建立 TCP 连接到服务器:                                              │
│      conn = dialer.Dial(ctx, server.Destination())                     │
│      → TCP 连接到 your-server.com:443                                   │
│                                                                         │
│   2. 构造 VLESS 请求头:                                                  │
│      request = {                                                       │
│          Version: 0,                                                   │
│          User: { UUID: "a3482e88-..." },                              │
│          Command: TCP,                                                 │
│          Address: "example.com",                                       │
│          Port: 443,                                                    │
│      }                                                                 │
│                                                                         │
│   3. 发送 VLESS 请求头:                                                  │
│      encoding.EncodeRequestHeader(conn, request, addons)               │
│                                                                         │
│   4. 启动双向数据转发:                                                   │
│      go buf.Copy(link.Reader, conn)   // 本地 -> 服务器                 │
│      go buf.Copy(conn, link.Writer)   // 服务器 -> 本地                 │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     │ VLESS 协议
                                     │ TCP 连接
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第 5 步: 服务端 VLESS Inbound 接收                                       │
│                                                                         │
│ vless.Handler.Process():                                                │
│   1. 解析 VLESS 请求头:                                                  │
│      request = encoding.DecodeRequestHeader(conn, validator)           │
│      → 验证 UUID                                                        │
│      → 提取目标: example.com:443                                        │
│                                                                         │
│   2. 调用 Dispatcher:                                                   │
│      link = dispatcher.Dispatch(ctx, net.Destination{                  │
│          Address: "example.com",                                       │
│          Port: 443,                                                    │
│      })                                                                │
│                                                                         │
│   3. 路由到 Freedom Outbound (直连)                                     │
│                                                                         │
│   4. 发送 VLESS 响应头:                                                  │
│      encoding.EncodeResponseHeader(conn, request, addons)              │
│                                                                         │
│   5. 双向数据转发:                                                       │
│      go buf.Copy(clientReader, link.Writer)                            │
│      go buf.Copy(link.Reader, clientWriter)                            │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第 6 步: Freedom Outbound 直连目标                                       │
│                                                                         │
│ freedom.Handler.Process():                                              │
│   1. 建立 TCP 连接:                                                      │
│      conn = dialer.Dial(ctx, destination)                              │
│      → TCP 连接到 example.com:443                                       │
│                                                                         │
│   2. 双向数据转发:                                                       │
│      go buf.Copy(link.Reader, conn)   // VLESS -> example.com          │
│      go buf.Copy(conn, link.Writer)   // example.com -> VLESS          │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第 7 步: 目标网站响应                                                    │
│                                                                         │
│ example.com 返回 HTTPS 响应                                             │
│   ↓                                                                     │
│ 数据沿原路返回:                                                         │
│   example.com → Freedom → VLESS Inbound → TCP → VLESS Outbound         │
│   → SOCKS5 Inbound → 浏览器                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                   客户端                                         │
│                                                                                 │
│  ┌──────────┐     ┌───────────────────────────────────────────────────────┐    │
│  │ 浏览器   │     │              Xray Client                              │    │
│  │          │     │  ┌─────────────┐  ┌───────────┐  ┌─────────────────┐ │    │
│  │          │────>│  │ SOCKS5      │──│Dispatcher │──│ VLESS Outbound  │ │    │
│  │          │<────│  │ Inbound     │  │ + Router  │  │                 │ │    │
│  │          │     │  │ :10808      │  └───────────┘  │ encode/decode   │ │    │
│  └──────────┘     │  └─────────────┘                 └────────┬────────┘ │    │
│                   └───────────────────────────────────────────┼──────────┘    │
└───────────────────────────────────────────────────────────────┼────────────────┘
                                                                │
                                                                │ VLESS over TCP
                                                                │ (可选 TLS/REALITY)
                                                                │
┌───────────────────────────────────────────────────────────────┼────────────────┐
│                                   服务端                       │                 │
│                                                                │                 │
│  ┌───────────────────────────────────────────────────────────────────────┐    │
│  │                            Xray Server                                │    │
│  │  ┌─────────────────┐  ┌───────────┐  ┌─────────────────────────────┐ │    │
│  │  │ VLESS Inbound   │──│Dispatcher │──│ Freedom Outbound            │ │    │
│  │  │ :443            │  │           │  │ (直连)                       │ │    │
│  │  │ decode/encode   │  └───────────┘  │                             │ │    │
│  │  └────────┬────────┘                 └─────────────┬───────────────┘ │    │
│  └───────────┼────────────────────────────────────────┼─────────────────┘    │
│              │                                        │                       │
└──────────────┼────────────────────────────────────────┼───────────────────────┘
               │                                        │
               │ VLESS Protocol                         │ Direct TCP
               │                                        │
               │                                        ▼
               │                              ┌──────────────────┐
               │                              │   目标服务器     │
               │                              │  example.com:443 │
               │                              └──────────────────┘
               │
   ┌───────────┴───────────────────────────────────────────────┐
   │                                                           │
   │  VLESS 协议数据包:                                        │
   │  ┌──────┬────────┬────────┬─────┬────────────┬──────────┐│
   │  │ Ver  │  UUID  │ Addons │ Cmd │ Dest Addr  │ Payload  ││
   │  │ 0x00 │ 16bytes│  ?B    │ 0x01│example.com │ TLS Data ││
   │  └──────┴────────┴────────┴─────┴────────────┴──────────┘│
   │                                                           │
   └───────────────────────────────────────────────────────────┘
```

---

## 11. 设计思考与总结

### 11.1 核心设计模式

#### 依赖注入 (Dependency Injection)

```go
// 声明式依赖
core.RequireFeatures(ctx, func(router routing.Router, ohm outbound.Manager) error {
    d.router = router
    d.ohm = ohm
    return nil
})
```

**设计思考**:
- 解耦模块间的直接依赖
- 支持运行时动态添加特性
- 便于测试和模块替换

#### 接口隔离 (Interface Segregation)

```go
// 最小化的接口定义
type Inbound interface {
    Network() []net.Network
    Process(ctx, network, conn, dispatcher) error
}

type Outbound interface {
    Process(ctx, link, dialer) error
}
```

**设计思考**:
- 协议实现只需关注自身逻辑
- 传输层、路由层等由框架处理
- 新协议实现成本低

#### 管道模式 (Pipe Pattern)

```go
// 通过 Pipe 解耦读写双方
uplinkReader, uplinkWriter := pipe.New(opt...)
downlinkReader, downlinkWriter := pipe.New(opt...)
```

**设计思考**:
- Inbound 和 Outbound 无需直接通信
- 支持异步读写和流量控制
- 便于添加统计、限速等中间件

### 11.2 架构优势

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            Xray 架构优势                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 高度模块化                                                          │
│     ├─ 协议层、传输层、路由层分离                                        │
│     ├─ 新协议/传输只需实现简单接口                                       │
│     └─ 配置驱动的组合方式                                                │
│                                                                         │
│  2. 灵活的路由系统                                                       │
│     ├─ 支持域名、IP、协议等多维度匹配                                     │
│     ├─ 支持 GeoIP/GeoSite 大规模规则                                    │
│     └─ 链式路由和负载均衡                                                │
│                                                                         │
│  3. 可扩展的传输层                                                       │
│     ├─ TCP/UDP/WebSocket/gRPC/KCP 等                                   │
│     ├─ TLS/REALITY 等安全传输                                           │
│     └─ 传输与协议正交组合                                                │
│                                                                         │
│  4. 性能优化                                                             │
│     ├─ Zero-copy 缓冲区管理                                             │
│     ├─ 连接复用 (Mux)                                                   │
│     ├─ XTLS 直接 splice                                                 │
│     └─ 预连接池 (Pre-connect)                                           │
│                                                                         │
│  5. 运行时动态配置                                                       │
│     ├─ 热添加/移除 Inbound/Outbound                                     │
│     ├─ 动态路由规则更新                                                  │
│     └─ API 管理接口                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.3 关键代码位置索引

| 功能模块 | 主要文件 | 核心类型/函数 |
|---------|---------|-------------|
| 启动入口 | `main/main.go` | `main()` |
| 运行命令 | `main/run.go` | `executeRun()`, `startXray()` |
| 核心实例 | `core/xray.go` | `Instance`, `New()`, `Start()` |
| 配置加载 | `core/config.go` | `LoadConfig()` |
| JSON 解析 | `infra/conf/serial/loader.go` | `DecodeJSONConfig()` |
| Inbound 管理 | `app/proxyman/inbound/inbound.go` | `Manager` |
| Outbound 管理 | `app/proxyman/outbound/outbound.go` | `Manager` |
| 连接 Worker | `app/proxyman/inbound/worker.go` | `tcpWorker` |
| 请求调度 | `app/dispatcher/default.go` | `DefaultDispatcher` |
| 路由器 | `app/router/router.go` | `Router`, `PickRoute()` |
| VLESS Inbound | `proxy/vless/inbound/inbound.go` | `Handler` |
| VLESS Outbound | `proxy/vless/outbound/outbound.go` | `Handler` |
| VLESS 编解码 | `proxy/vless/encoding/encoding.go` | `Encode/DecodeRequestHeader()` |
| 数据管道 | `transport/pipe/pipe.go` | `pipe`, `Reader`, `Writer` |
| TCP 传输 | `transport/internet/tcp/` | `Dial()`, `ListenTCP()` |

### 11.4 调试技巧

```bash
# 启用详细日志
{
  "log": {
    "loglevel": "debug",
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log"
  }
}

# 测试配置有效性
./xray -test -config config.json

# 查看版本和编译信息
./xray version
```

---

## 附录: 关键接口定义

### Feature 接口

```go
// features/feature.go
type Feature interface {
    common.HasType      // Type() interface{}
    common.Runnable     // Start() error; Close() error
}
```

### Inbound/Outbound 接口

```go
// proxy/proxy.go
type Inbound interface {
    Network() []net.Network
    Process(context.Context, net.Network, stat.Connection, routing.Dispatcher) error
}

type Outbound interface {
    Process(context.Context, *transport.Link, internet.Dialer) error
}
```

### Dispatcher 接口

```go
// features/routing/dispatcher.go
type Dispatcher interface {
    features.Feature
    Dispatch(ctx context.Context, dest net.Destination) (*transport.Link, error)
    DispatchLink(ctx context.Context, dest net.Destination, link *transport.Link) error
}
```

### Router 接口

```go
// features/routing/router.go
type Router interface {
    features.Feature
    PickRoute(ctx Context) (Route, error)
    AddRule(config *serial.TypedMessage, shouldAppend bool) error
    RemoveRule(tag string) error
}
```

---

*文档生成时间: 2024*
*基于 Xray-core 源码分析*
