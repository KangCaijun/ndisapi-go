# Proxifyre 功能增强方案

## 📋 增强目标

在现有 proxifyre 项目基础上增加以下功能：

1. **支持根据 PID 代理特定进程**：能够精确指定进程 ID，代理该进程的流量
2. **支持全局流量代理**：代理所有进程的流量（可配置排除列表）

---

## 🔍 现有架构分析

### 当前工作机制

1. **进程识别**：通过 `ProcessLookup.FindProcessInfo()` 获取进程信息（包含 PID、进程名、路径）
2. **代理匹配**：通过进程路径名包含关系匹配（`strings.Contains(process.PathName, name)`）
3. **存储结构**：`nameToProxy map[string]int` 存储进程名到代理ID的映射

### 现有数据流

```
数据包到达 → NDIS拦截 → 查找进程信息 → 进程名匹配 → 重定向到透明代理 → SOCKS5转发
```

---

## 📝 详细修改方案

### 一、配置文件结构修改

#### 现有配置 (`config.json`)

```json
{
  "proxies": [
    {
      "appNames": ["firefox"],
      "endpoint": "socks5://127.0.0.1:8080"
    }
  ]
}
```

#### 新配置结构

```json
{
  "proxies": [
    {
      "appNames": ["firefox", "chrome"],
      "appPids": [1234, 5678],
      "global": false,
      "endpoint": "socks5://127.0.0.1:8080"
    },
    {
      "global": true,
      "excludeAppNames": ["proxifyre", "svchost"],
      "endpoint": "socks5://127.0.0.1:9090"
    }
  ]
}
```

#### 新增字段说明

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `appPids` | `[]uint32` | 进程 PID 数组，支持精确匹配特定进程实例 | `[]` |
| `global` | `bool` | 是否为全局代理，为 true 时代理所有流量 | `false` |
| `excludeAppNames` | `[]string` | 全局代理时排除的进程名列表 | `[]` |

#### 配置示例场景

**场景1：仅代理特定 PID**
```json
{
  "proxies": [
    {
      "appPids": [1234],
      "endpoint": "socks5://127.0.0.1:8080"
    }
  ]
}
```

**场景2：全局代理 + 排除系统进程**
```json
{
  "proxies": [
    {
      "global": true,
      "excludeAppNames": ["proxifyre", "svchost", "System"],
      "endpoint": "socks5://127.0.0.1:8080"
    }
  ]
}
```

**场景3：混合模式**
```json
{
  "proxies": [
    {
      "appNames": ["firefox"],
      "appPids": [1234, 5678],
      "endpoint": "socks5://127.0.0.1:8080"
    },
    {
      "appNames": ["chrome"],
      "endpoint": "socks5://127.0.0.1:9090"
    }
  ]
}
```

---

### 二、数据结构修改

#### 1. SocksLocalRouter 结构体扩展

**文件**：`socks_local_router.go`

**修改位置**：`SocksLocalRouter` 结构体定义

```go
type SocksLocalRouter struct {
    sync.Mutex
    *A.NdisApi
    
    tcpConnections map[string]TCPPortMapping
    udpEndpoints   map[string]UDPPortMapping
    
    tcpMutex sync.RWMutex
    udpMutex sync.RWMutex
    
    ctx    context.Context
    cancel context.CancelFunc
    wg     sync.WaitGroup
    
    proxyServers []*TransparentProxy
    
    // ========== 现有字段 ==========
    nameToProxy  map[string]int  // 进程名 -> 代理ID
    
    // ========== 新增字段 ==========
    pidToProxy    map[uint32]int  // PID -> 代理ID
    globalProxyID int             // 全局代理ID，-1 表示没有全局代理
    excludeNames  []string        // 全局代理时排除的进程名列表
    
    // ========== 其他字段 ==========
    ifNotifyHandle windows.Handle
    ifIndex        int
    adapters       *A.TcpAdapterList
    defaultAdapter *N.NetworkAdapterInfo
    processLookup  *N.ProcessLookup
    filter         *D.QueuedPacketFilter
    staticFilter   *D.StaticFilters
    isActive       bool
}
```

#### 2. 初始化修改

**文件**：`socks_local_router.go`

**修改位置**：`NewSocksLocalRouter` 函数

```go
func NewSocksLocalRouter(api *A.NdisApi, debug bool) (*SocksLocalRouter, error) {
    // ... 现有代码 ...
    
    socksLocalRouter := &SocksLocalRouter{
        NdisApi: api,
        
        ctx:    ctx,
        cancel: cancel,
        
        // 现有初始化
        nameToProxy:    make(map[string]int),
        tcpConnections: make(map[string]TCPPortMapping),
        udpEndpoints:   make(map[string]UDPPortMapping),
        processLookup:  N.NewProcessLookup(),
        adapters:       adapters,
        
        // ========== 新增初始化 ==========
        pidToProxy:    make(map[uint32]int),
        globalProxyID: -1,              // 默认无全局代理
        excludeNames:  make([]string, 0),
        
        isActive: false,
    }
    
    // ... 其余代码 ...
}
```

---

### 三、新增方法

#### 1. PID 关联方法

**文件**：`socks_local_router.go`

**添加位置**：在 `AssociateProcessNameToProxy` 方法之后

```go
// AssociateProcessPidToProxy associates a process PID with a proxy ID.
// This allows precise targeting of specific process instances by their PID.
//
// Parameters:
//   - pid: The process ID to associate
//   - proxyID: The proxy server index
//
// Returns:
//   - error if the proxy index is out of range
func (s *SocksLocalRouter) AssociateProcessPidToProxy(pid uint32, proxyID int) error {
    s.Lock()
    defer s.Unlock()

    if proxyID >= len(s.proxyServers) {
        return fmt.Errorf("AssociateProcessPidToProxy: proxy index is out of range")
    }
    
    s.pidToProxy[pid] = proxyID
    log.Printf("Associated PID %d with proxy ID %d", pid, proxyID)
    
    return nil
}

// RemoveProcessPidAssociation removes the association between a PID and a proxy.
// Useful when a process terminates and PID might be reused.
func (s *SocksLocalRouter) RemoveProcessPidAssociation(pid uint32) {
    s.Lock()
    defer s.Unlock()
    
    delete(s.pidToProxy, pid)
    log.Printf("Removed PID %d association", pid)
}
```

#### 2. 全局代理设置方法

**文件**：`socks_local_router.go`

**添加位置**：在 `AssociateProcessPidToProxy` 方法之后

```go
// SetGlobalProxy sets a proxy as the global proxy for all traffic.
// When enabled, all processes will be proxied except those in the exclude list.
//
// Parameters:
//   - proxyID: The proxy server index to use as global proxy
//   - excludeNames: List of process name patterns to exclude from global proxying
//
// Returns:
//   - error if the proxy index is out of range
//
// Note: This is a powerful feature that affects all system traffic.
// Make sure to exclude critical system processes and the proxy application itself.
func (s *SocksLocalRouter) SetGlobalProxy(proxyID int, excludeNames []string) error {
    s.Lock()
    defer s.Unlock()

    if proxyID >= len(s.proxyServers) {
        return fmt.Errorf("SetGlobalProxy: proxy index is out of range")
    }
    
    s.globalProxyID = proxyID
    s.excludeNames = excludeNames
    
    log.Printf("Set global proxy with ID %d, excluding %d processes", proxyID, len(excludeNames))
    for _, name := range excludeNames {
        log.Printf("  - Excluding: %s", name)
    }
    
    return nil
}

// DisableGlobalProxy disables the global proxy setting.
func (s *SocksLocalRouter) DisableGlobalProxy() {
    s.Lock()
    defer s.Unlock()
    
    s.globalProxyID = -1
    s.excludeNames = make([]string, 0)
    
    log.Println("Global proxy disabled")
}

// IsGlobalProxyEnabled checks if global proxy is currently enabled.
func (s *SocksLocalRouter) IsGlobalProxyEnabled() bool {
    s.Lock()
    defer s.Unlock()
    
    return s.globalProxyID >= 0
}
```

---

### 四、修改现有方法

#### 1. 修改 GetProxyPortTCP

**文件**：`socks_local_router.go`

**修改位置**：`GetProxyPortTCP` 方法（约497行）

**原有实现**：
```go
func (s *SocksLocalRouter) GetProxyPortTCP(process *N.ProcessInfo) uint16 {
    s.Lock()
    defer s.Unlock()

    for name, proxyID := range s.nameToProxy {
        if strings.Contains(process.PathName, name) {
            if proxyID < len(s.proxyServers) && s.proxyServers[proxyID] != nil {
                return s.proxyServers[proxyID].GetLocalTcpProxyPort()
            }
        }
    }

    return 0
}
```

**新实现**（三层优先级匹配）：
```go
// GetProxyPortTCP retrieves the TCP proxy port for a given process.
// Matching priority (highest to lowest):
//   1. Exact PID match
//   2. Process name pattern match
//   3. Global proxy (if enabled and not in exclude list)
//
// Returns:
//   - uint16: The proxy port to use, or 0 if no proxy should be used
func (s *SocksLocalRouter) GetProxyPortTCP(process *N.ProcessInfo) uint16 {
    s.Lock()
    defer s.Unlock()

    // ========== 优先级1：精确 PID 匹配 ==========
    if proxyID, exists := s.pidToProxy[process.ProcessId]; exists {
        if proxyID < len(s.proxyServers) && s.proxyServers[proxyID] != nil {
            log.Printf("[TCP] Matched by PID: %d -> Proxy ID %d", process.ProcessId, proxyID)
            return s.proxyServers[proxyID].GetLocalTcpProxyPort()
        }
    }

    // ========== 优先级2：进程名匹配 ==========
    for name, proxyID := range s.nameToProxy {
        if strings.Contains(process.PathName, name) {
            if proxyID < len(s.proxyServers) && s.proxyServers[proxyID] != nil {
                log.Printf("[TCP] Matched by name: %s -> Proxy ID %d", name, proxyID)
                return s.proxyServers[proxyID].GetLocalTcpProxyPort()
            }
        }
    }

    // ========== 优先级3：全局代理（需要检查排除列表）==========
    if s.globalProxyID >= 0 {
        // 检查是否在排除列表中
        for _, excludeName := range s.excludeNames {
            if strings.Contains(process.PathName, excludeName) {
                // 在排除列表中，不代理
                return 0
            }
        }
        
        // 不在排除列表中，使用全局代理
        if s.globalProxyID < len(s.proxyServers) && s.proxyServers[s.globalProxyID] != nil {
            log.Printf("[TCP] Matched by global proxy: %s (PID: %d) -> Proxy ID %d", 
                filepath.Base(process.PathName), process.ProcessId, s.globalProxyID)
            return s.proxyServers[s.globalProxyID].GetLocalTcpProxyPort()
        }
    }

    // 不匹配任何规则，不代理
    return 0
}
```

#### 2. 修改 GetProxyPortUDP

**文件**：`socks_local_router.go`

**修改位置**：`GetProxyPortUDP` 方法（约513行）

**新实现**（与 TCP 逻辑一致）：
```go
// GetProxyPortUDP retrieves the UDP proxy port for a given process.
// Matching priority (highest to lowest):
//   1. Exact PID match
//   2. Process name pattern match
//   3. Global proxy (if enabled and not in exclude list)
//
// Returns:
//   - uint16: The proxy port to use, or 0 if no proxy should be used
func (s *SocksLocalRouter) GetProxyPortUDP(process *N.ProcessInfo) uint16 {
    s.Lock()
    defer s.Unlock()

    // ========== 优先级1：精确 PID 匹配 ==========
    if proxyID, exists := s.pidToProxy[process.ProcessId]; exists {
        if proxyID < len(s.proxyServers) && s.proxyServers[proxyID] != nil {
            log.Printf("[UDP] Matched by PID: %d -> Proxy ID %d", process.ProcessId, proxyID)
            return s.proxyServers[proxyID].GetLocalUdpProxyPort()
        }
    }

    // ========== 优先级2：进程名匹配 ==========
    for name, proxyID := range s.nameToProxy {
        if strings.Contains(process.PathName, name) {
            if proxyID < len(s.proxyServers) && s.proxyServers[proxyID] != nil {
                log.Printf("[UDP] Matched by name: %s -> Proxy ID %d", name, proxyID)
                return s.proxyServers[proxyID].GetLocalUdpProxyPort()
            }
        }
    }

    // ========== 优先级3：全局代理（需要检查排除列表）==========
    if s.globalProxyID >= 0 {
        // 检查是否在排除列表中
        for _, excludeName := range s.excludeNames {
            if strings.Contains(process.PathName, excludeName) {
                // 在排除列表中，不代理
                return 0
            }
        }
        
        // 不在排除列表中，使用全局代理
        if s.globalProxyID < len(s.proxyServers) && s.proxyServers[s.globalProxyID] != nil {
            log.Printf("[UDP] Matched by global proxy: %s (PID: %d) -> Proxy ID %d", 
                filepath.Base(process.PathName), process.ProcessId, s.globalProxyID)
            return s.proxyServers[s.globalProxyID].GetLocalUdpProxyPort()
        }
    }

    // 不匹配任何规则，不代理
    return 0
}
```

---

### 五、主程序配置加载修改

#### 修改 main.go

**文件**：`main.go`

**修改位置**：`setupProxyRouter` 函数（约49-94行）

**原有实现**：
```go
func setupProxyRouter(api *A.NdisApi) (*SocksLocalRouter, error) {
    router, err := NewSocksLocalRouter(api, true)
    if err != nil {
        return nil, fmt.Errorf("Failed to create SOCKS5 Local Router instance: %v", err)
    }

    // Load configuration from JSON file
    configFilePath := "config.json"
    configFile, err := os.Open(configFilePath)
    if err != nil {
        return nil, fmt.Errorf("Failed to open config file: %v", err)
    }
    defer configFile.Close()

    var serviceSettings struct {
        Proxies []struct {
            AppNames []string `json:"appNames"`
            Endpoint string   `json:"endpoint"`
        } `json:"proxies"`
    }

    if err := json.NewDecoder(configFile).Decode(&serviceSettings); err != nil {
        log.Fatalf("Failed to decode config file: %v", err)
    }

    // Add SOCKS5 proxies
    for _, appSettings := range serviceSettings.Proxies {
        proxyID, err := router.AddSocks5Proxy(&appSettings.Endpoint)
        if err != nil {
            return nil, fmt.Errorf("Failed to add Socks5 proxy for endpoint %s: %v", appSettings.Endpoint, err)
        }

        for _, appName := range appSettings.AppNames {
            if err := router.AssociateProcessNameToProxy(appName, proxyID); err != nil {
                return nil, fmt.Errorf("Failed to associate %s with proxy ID %d: %v", appName, proxyID, err)
            }
        }
    }

    if err := router.Start(); err != nil {
        return nil, fmt.Errorf("Error starting filter: %s", err.Error())
    }
    log.Println("SOCKS5 local router has been started.")

    return router, nil
}
```

**新实现**：
```go
func setupProxyRouter(api *A.NdisApi) (*SocksLocalRouter, error) {
    router, err := NewSocksLocalRouter(api, true)
    if err != nil {
        return nil, fmt.Errorf("Failed to create SOCKS5 Local Router instance: %v", err)
    }

    // Load configuration from JSON file
    configFilePath := "config.json"
    configFile, err := os.Open(configFilePath)
    if err != nil {
        return nil, fmt.Errorf("Failed to open config file: %v", err)
    }
    defer configFile.Close()

    // ========== 扩展配置结构 ==========
    var serviceSettings struct {
        Proxies []struct {
            AppNames        []string `json:"appNames"`
            AppPids         []uint32 `json:"appPids"`         // 新增：PID 列表
            Global          bool     `json:"global"`          // 新增：全局代理标志
            ExcludeAppNames []string `json:"excludeAppNames"` // 新增：排除列表
            Endpoint        string   `json:"endpoint"`
        } `json:"proxies"`
    }

    if err := json.NewDecoder(configFile).Decode(&serviceSettings); err != nil {
        log.Fatalf("Failed to decode config file: %v", err)
    }

    // ========== 处理每个代理配置 ==========
    for idx, appSettings := range serviceSettings.Proxies {
        log.Printf("Processing proxy configuration #%d: %s", idx+1, appSettings.Endpoint)
        
        // 添加 SOCKS5 代理
        proxyID, err := router.AddSocks5Proxy(&appSettings.Endpoint)
        if err != nil {
            return nil, fmt.Errorf("Failed to add Socks5 proxy for endpoint %s: %v", appSettings.Endpoint, err)
        }

        // ========== 处理进程名关联（原有功能）==========
        for _, appName := range appSettings.AppNames {
            if err := router.AssociateProcessNameToProxy(appName, proxyID); err != nil {
                return nil, fmt.Errorf("Failed to associate app name %s: %v", appName, err)
            }
            log.Printf("  ✓ Associated app name: %s", appName)
        }

        // ========== 新增：处理 PID 关联 ==========
        for _, pid := range appSettings.AppPids {
            if err := router.AssociateProcessPidToProxy(pid, proxyID); err != nil {
                return nil, fmt.Errorf("Failed to associate PID %d: %v", pid, err)
            }
            log.Printf("  ✓ Associated PID: %d", pid)
        }

        // ========== 新增：处理全局代理 ==========
        if appSettings.Global {
            if err := router.SetGlobalProxy(proxyID, appSettings.ExcludeAppNames); err != nil {
                return nil, fmt.Errorf("Failed to set global proxy: %v", err)
            }
            log.Printf("  ✓ Set as GLOBAL proxy (excluding %d processes)", len(appSettings.ExcludeAppNames))
        }
    }

    // 启动路由器
    if err := router.Start(); err != nil {
        return nil, fmt.Errorf("Error starting filter: %s", err.Error())
    }
    log.Println("SOCKS5 local router has been started.")

    return router, nil
}
```

---

### 六、需要导入的额外包

**文件**：`socks_local_router.go`

确保已导入 `filepath` 包（可能已经导入）：

```go
import (
    // ... 现有导入 ...
    "path/filepath"
    // ... 其他导入 ...
)
```

---

## 🎯 匹配优先级总结

当数据包到达时，按以下优先级查找代理：

```
1. PID 精确匹配（最高优先级）
   ↓ 未匹配
2. 进程名模式匹配
   ↓ 未匹配
3. 全局代理（需检查排除列表）
   ↓ 不在排除列表
4. 使用全局代理
   ↓ 在排除列表或无全局代理
5. 不代理（直连）
```

**流程图**：

```
                    数据包到达
                        ↓
                 获取进程信息 (PID, PathName)
                        ↓
          ┌─────────────┴─────────────┐
          │ PID 在 pidToProxy 中？    │
          └─────────────┬─────────────┘
                 是 ↓        ↓ 否
            使用对应代理    继续匹配
                        ↓
          ┌─────────────┴─────────────┐
          │ PathName 匹配 nameToProxy？│
          └─────────────┬─────────────┘
                 是 ↓        ↓ 否
            使用对应代理    继续匹配
                        ↓
          ┌─────────────┴─────────────┐
          │ globalProxyID >= 0？      │
          └─────────────┬─────────────┘
                 是 ↓        ↓ 否
          ┌─────────────┐    直接放行
          │在排除列表中？ │
          └─────────────┘
           否 ↓      ↓ 是
      使用全局代理  直接放行
```

---

## 💡 使用示例

### 示例1：代理特定 PID 的 Chrome

假设 Chrome 进程的 PID 是 12345：

```json
{
  "proxies": [
    {
      "appPids": [12345],
      "endpoint": "socks5://127.0.0.1:8080"
    }
  ]
}
```

**效果**：
- 只有 PID 为 12345 的进程流量会被代理
- 其他 Chrome 进程不受影响

### 示例2：全局代理模式

```json
{
  "proxies": [
    {
      "global": true,
      "excludeAppNames": [
        "proxifyre",
        "svchost",
        "System",
        "wininit",
        "services"
      ],
      "endpoint": "socks5://127.0.0.1:8080"
    }
  ]
}
```

**效果**：
- 所有进程的流量都会被代理
- 排除列表中的系统进程保持直连

### 示例3：混合模式（推荐）

```json
{
  "proxies": [
    {
      "appNames": ["firefox", "telegram"],
      "endpoint": "socks5://127.0.0.1:8080"
    },
    {
      "appPids": [12345, 67890],
      "endpoint": "socks5://127.0.0.1:9090"
    },
    {
      "global": false,
      "appNames": ["chrome"],
      "endpoint": "socks5://127.0.0.1:7070"
    }
  ]
}
```

**效果**：
- Firefox 和 Telegram 使用 8080 端口代理
- PID 为 12345 和 67890 的进程使用 9090 端口代理
- Chrome 使用 7070 端口代理
- 其他进程直连

---

## ⚠️ 注意事项和最佳实践

### 1. 全局代理的风险

- **排除关键系统进程**：必须排除 `svchost`、`System` 等关键进程
- **排除代理程序自身**：避免循环代理导致死锁
- **测试后再使用**：先用进程名模式测试，再启用全局代理

推荐排除列表：
```json
"excludeAppNames": [
  "proxifyre",      // 代理程序自身
  "svchost",        // Windows 服务宿主
  "System",         // 系统进程
  "services",       // 服务控制管理器
  "wininit",        // Windows 启动应用程序
  "csrss",          // 客户端/服务器运行时子系统
  "lsass",          // 本地安全认证服务器
  "smss"            // 会话管理器子系统
]
```

### 2. PID 使用注意事项

- **临时性**：进程终止后 PID 可能被重用
- **重启失效**：应用重启后 PID 会改变
- **适用场景**：适合临时调试、测试特定进程实例

### 3. 性能考虑

- 全局代理会增加所有流量的处理开销
- 排除列表检查是线性扫描，不要添加过多项
- 建议优先使用进程名匹配，PID 匹配作为补充

### 4. 配置验证

建议在 `setupProxyRouter` 中添加配置验证：

```go
// 验证配置合理性
if appSettings.Global {
    if len(appSettings.AppNames) > 0 || len(appSettings.AppPids) > 0 {
        log.Println("Warning: Global proxy ignores appNames and appPids")
    }
    if len(appSettings.ExcludeAppNames) == 0 {
        log.Println("Warning: Global proxy with empty exclude list is dangerous!")
    }
}
```

### 5. 日志记录

增强版会产生更多日志，建议添加日志级别控制：

- **DEBUG**：每个数据包的匹配过程
- **INFO**：代理规则加载、连接建立/关闭
- **WARNING**：配置问题、匹配异常
- **ERROR**：系统错误

---

## 🔧 扩展功能建议

### 1. PID 自动管理

监控进程生命周期，自动清理失效的 PID 映射：

```go
// 定期检查 PID 是否仍然有效
func (s *SocksLocalRouter) cleanupInvalidPids() {
    s.Lock()
    defer s.Unlock()
    
    for pid := range s.pidToProxy {
        if !isProcessAlive(pid) {
            delete(s.pidToProxy, pid)
            log.Printf("Cleaned up terminated process PID: %d", pid)
        }
    }
}

// 使用 Windows API 检查进程是否存在
func isProcessAlive(pid uint32) bool {
    handle, err := windows.OpenProcess(windows.PROCESS_QUERY_LIMITED_INFORMATION, false, pid)
    if err != nil {
        return false
    }
    windows.CloseHandle(handle)
    return true
}
```

### 2. 配置热加载

支持无需重启即可更新配置：

```go
import "github.com/fsnotify/fsnotify"

func (s *SocksLocalRouter) watchConfigFile(configPath string) {
    watcher, _ := fsnotify.NewWatcher()
    watcher.Add(configPath)
    
    for {
        select {
        case event := <-watcher.Events:
            if event.Op&fsnotify.Write == fsnotify.Write {
                s.reloadConfig(configPath)
            }
        }
    }
}
```

### 3. 优先级配置

为每个规则添加优先级字段：

```json
{
  "proxies": [
    {
      "priority": 100,
      "appPids": [1234],
      "endpoint": "socks5://127.0.0.1:8080"
    },
    {
      "priority": 50,
      "appNames": ["chrome"],
      "endpoint": "socks5://127.0.0.1:9090"
    }
  ]
}
```

### 4. 进程白名单/黑名单

增加更灵活的匹配模式：

```json
{
  "proxies": [
    {
      "mode": "whitelist",
      "appNames": ["firefox", "chrome"],
      "endpoint": "socks5://127.0.0.1:8080"
    },
    {
      "mode": "blacklist",
      "excludeAppNames": ["svchost"],
      "global": true,
      "endpoint": "socks5://127.0.0.1:9090"
    }
  ]
}
```

### 5. 统计信息

添加流量统计和监控：

```go
type ProxyStats struct {
    TotalConnections   int64
    ActiveConnections  int32
    BytesSent         int64
    BytesReceived     int64
    MatchedByPID      int64
    MatchedByName     int64
    MatchedByGlobal   int64
}
```

### 6. 正则表达式支持

支持更灵活的进程名匹配：

```json
{
  "proxies": [
    {
      "appNamePatterns": ["^C:\\\\Program Files\\\\.*\\.exe$"],
      "endpoint": "socks5://127.0.0.1:8080"
    }
  ]
}
```

---

## 📋 实现检查清单

实施修改时，按照以下顺序进行：

- [ ] 1. 备份现有代码
- [ ] 2. 修改 `SocksLocalRouter` 结构体（添加新字段）
- [ ] 3. 修改 `NewSocksLocalRouter` 初始化函数
- [ ] 4. 添加 `AssociateProcessPidToProxy` 方法
- [ ] 5. 添加 `SetGlobalProxy` 方法
- [ ] 6. 添加 `DisableGlobalProxy` 和 `IsGlobalProxyEnabled` 方法
- [ ] 7. 修改 `GetProxyPortTCP` 方法（三层匹配）
- [ ] 8. 修改 `GetProxyPortUDP` 方法（三层匹配）
- [ ] 9. 修改 `main.go` 中的配置结构体
- [ ] 10. 修改 `setupProxyRouter` 函数（处理新配置）
- [ ] 11. 创建测试配置文件
- [ ] 12. 测试 PID 匹配功能
- [ ] 13. 测试全局代理功能
- [ ] 14. 测试混合模式
- [ ] 15. 性能测试和优化
- [ ] 16. 更新 README.md 文档
- [ ] 17. 添加使用示例

---

## 🧪 测试方案

### 测试1：PID 匹配功能

1. 启动一个测试进程（如 `ping.exe`）
2. 记录其 PID
3. 配置文件中添加该 PID
4. 验证流量是否被代理

### 测试2：全局代理功能

1. 配置全局代理
2. 测试多个不同应用的流量
3. 验证排除列表是否生效
4. 检查系统稳定性

### 测试3：优先级测试

1. 为同一进程配置多个规则（PID + 进程名 + 全局）
2. 验证 PID 匹配优先级最高
3. 验证进程名匹配优先级次之
4. 验证全局代理优先级最低

### 测试4：性能测试

1. 基准测试（无代理）
2. PID 匹配性能
3. 全局代理性能
4. 大量排除规则的性能影响

---

## 📚 相关文档

实施后需要更新的文档：

1. **README.md**：添加新功能说明
2. **config.example.json**：提供配置示例
3. **CHANGELOG.md**：记录版本变更

---

## 📞 问题排查

### 问题1：全局代理导致系统网络异常

**原因**：关键系统进程被代理

**解决**：
- 检查排除列表是否包含系统进程
- 临时禁用全局代理
- 查看日志确认哪些进程被代理

### 问题2：PID 匹配不生效

**原因**：进程已终止，PID 被重用

**解决**：
- 验证 PID 是否正确
- 检查进程是否仍在运行
- 实现 PID 自动清理机制

### 问题3：配置文件解析失败

**原因**：JSON 格式错误

**解决**：
- 使用 JSON 验证工具检查格式
- 查看程序启动日志
- 使用配置示例作为模板

---

## 🎉 总结

本方案提供了完整的实现路径，涵盖：

- ✅ 数据结构设计
- ✅ 方法实现细节
- ✅ 配置文件格式
- ✅ 优先级匹配逻辑
- ✅ 错误处理和日志
- ✅ 测试和验证方案
- ✅ 扩展功能建议

实施后，proxifyre 将具备：
- **精确控制**：通过 PID 精确控制特定进程实例
- **全局能力**：支持全局代理模式
- **灵活配置**：支持多种匹配模式组合
- **安全可控**：排除列表保护关键系统进程

---

**文档版本**：1.0  
**创建日期**：2025-12-03  
**最后更新**：2025-12-03

