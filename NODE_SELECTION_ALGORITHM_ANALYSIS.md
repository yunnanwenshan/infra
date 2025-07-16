# E2B Infrastructure 节点选择算法详细解读

## 算法概述

E2B Infrastructure的节点选择算法位于 `packages/api/internal/orchestrator/create_instance.go` 文件中，是一个多层优先级的智能调度算法，确保沙盒创建的高可用性和负载均衡。

## 核心常量定义

```go
const (
    maxNodeRetries       = 3                    // 最大重试次数
    leastBusyNodeTimeout = 60 * time.Second     // 获取最忙节点的超时时间
    maxStartingInstancesPerNode = 3             // 每个节点最大并发启动实例数
)
```

## 算法完整流程分析

### 第一阶段：恢复场景的节点优先选择

**文件位置**: `packages/api/internal/orchestrator/create_instance.go:135-142`

```go
// 133: 定义节点变量
var node *Node

// 135-142: 恢复操作的节点优先选择逻辑
if isResume && clientID != nil {
    // 136: 记录遥测事件，用于监控和调试
    telemetry.ReportEvent(childCtx, "Placing sandbox on the node where the snapshot was taken")
    
    // 138: 尝试获取原始节点（快照所在节点）
    node, _ = o.nodes.Get(*clientID)
    
    // 139-141: 检查原始节点是否可用
    if node != nil && node.Status() != api.NodeStatusReady {
        node = nil  // 原始节点不可用，重置为nil
    }
}
```

**逐行解读**:
- **L135**: `isResume` 判断是否为恢复操作，`clientID` 是原始节点ID
- **L136**: 记录遥测事件，便于运维监控恢复操作的节点选择
- **L138**: 从节点池中获取指定的原始节点，使用 `_` 忽略可能的错误
- **L139**: 双重检查：节点存在且状态为Ready
- **L140**: 如果原始节点不可用，将node设为nil，后续会选择其他节点

### 第二阶段：重试循环和节点选择主逻辑

**文件位置**: `packages/api/internal/orchestrator/create_instance.go:144-201`

```go
// 144: 初始化重试计数器
attempt := 1

// 145: 初始化排除节点映射，用于记录失败的节点
nodesExcluded := make(map[string]*Node)

// 146: 开始主循环，最多重试maxNodeRetries次
for {
    // 147-156: 超时检查和上下文取消处理
    select {
    case <-childCtx.Done():
        return nil, &api.APIError{
            Code:      http.StatusRequestTimeout,
            ClientMsg: "Failed to create sandbox",
            Err:       fmt.Errorf("timeout while creating sandbox, attempt #%d", attempt),
        }
    default:
        // Continue - 继续执行后续逻辑
    }

    // 158-164: 最大重试次数检查
    if attempt > maxNodeRetries {
        return nil, &api.APIError{
            Code:      http.StatusInternalServerError,
            ClientMsg: "Failed to create sandbox", 
            Err:       errSandboxCreateFailed,
        }
    }

    // 166-177: 获取可用节点
    if node == nil {
        node, err = o.getLeastBusyNode(childCtx, nodesExcluded)
        if err != nil {
            telemetry.ReportError(childCtx, "failed to get least busy node", err)
            
            return nil, &api.APIError{
                Code:      http.StatusInternalServerError,
                ClientMsg: "Failed to get node to place sandbox on.",
                Err:       fmt.Errorf("failed to get least busy node: %w", err),
            }
        }
    }
```

**逐行解读**:
- **L144**: 重试计数从1开始，便于日志记录和错误信息
- **L145**: `map[string]*Node` 存储已尝试失败的节点，避免重复选择
- **L147-155**: 使用 `select` 检查上下文是否被取消（超时或手动取消）
- **L158**: 检查重试次数是否超过限制（3次）
- **L166**: `node == nil` 表示需要选择新节点（初次选择或上次失败）
- **L167**: 调用核心的最忙节点选择算法，传入排除列表

### 第三阶段：资源预占和沙盒创建

```go
    // 179-183: 在节点上预占资源，防止并发创建时资源冲突
    node.sbxsInProgress.Insert(sandboxID, &sbxInProgress{
        MiBMemory: build.RamMb,     // 预占内存资源
        CPUs:      build.Vcpu,      // 预占CPU资源
    })

    // 185: 通过gRPC调用节点创建沙盒
    _, err = node.Client.Sandbox.Create(childCtx, sbxRequest)
    
    // 187-190: 创建成功，跳出循环
    if err == nil {
        break
    }

    // 192: 创建失败，释放预占的资源
    node.sbxsInProgress.Remove(sandboxID)

    // 194: 记录失败日志，包含详细的错误信息
    log.Printf("failed to create sandbox '%s' on node '%s', attempt #%d: %v", 
               sandboxID, node.Info.ID, attempt, utils.UnwrapGRPCError(err))

    // 197-200: 标记节点失败，准备下次重试
    node.createFails.Add(1)              // 增加节点失败计数
    nodesExcluded[node.Info.ID] = node   // 将节点加入排除列表
    node = nil                           // 重置节点，下次循环会选择新节点
    attempt += 1                         // 增加重试计数
}
```

**逐行解读**:
- **L179-183**: 预占资源机制，防止并发创建时多个请求选择同一节点导致资源不足
- **L185**: 实际的沙盒创建请求，通过gRPC与Orchestrator节点通信
- **L187**: 成功时直接break，跳出重试循环
- **L192**: 失败时立即释放预占资源，避免资源泄露
- **L197**: 原子操作增加节点失败计数，用于节点健康评估
- **L198**: 将失败节点加入排除列表，下次重试不会再选择
- **L199**: 重置节点为nil，强制下次循环重新选择

## 核心算法：getLeastBusyNode

**文件位置**: `packages/api/internal/orchestrator/create_instance.go:267-294`

```go
// 267: 函数定义，返回最不忙的节点
func (o *Orchestrator) getLeastBusyNode(parentCtx context.Context, nodesExcluded map[string]*Node) (leastBusyNode *Node, err error) {
    // 268: 设置60秒超时，防止无限等待
    ctx, cancel := context.WithTimeout(parentCtx, leastBusyNodeTimeout)
    defer cancel()

    // 271-272: 创建分布式追踪span
    childCtx, childSpan := o.tracer.Start(ctx, "get-least-busy-node")
    defer childSpan.End()

    // 275-278: 首次尝试，不等待直接查找
    leastBusyNode, err = o.findLeastBusyNode(nodesExcluded)
    if err == nil {
        return leastBusyNode, nil
    }

    // 281-293: 如果没有可用节点，轮询等待
    ticker := time.NewTicker(10 * time.Millisecond)
    for {
        select {
        case <-childCtx.Done():
            return nil, childCtx.Err()
        case <-ticker.C:
            leastBusyNode, err = o.findLeastBusyNode(nodesExcluded)
            if err == nil {
                return leastBusyNode, nil
            }
        }
    }
}
```

**逐行解读**:
- **L268**: 设置独立的超时上下文，避免无限期等待节点可用
- **L275**: 第一次尝试立即查找，大多数情况下能够直接找到
- **L281**: 创建10毫秒间隔的定时器，进行轮询等待
- **L284**: 检查上下文是否超时或被取消
- **L286**: 定时器触发时重新尝试查找节点

## 最核心算法：findLeastBusyNode

**文件位置**: `packages/api/internal/orchestrator/create_instance.go:298-335`

```go
// 298: 查找最不忙节点的核心实现
func (o *Orchestrator) findLeastBusyNode(nodesExcluded map[string]*Node) (leastBusyNode *Node, err error) {
    // 299: 遍历所有节点
    for _, node := range o.nodes.Items() {
        // 301-303: 跳过空节点（并发删除场景）
        if node == nil {
            continue
        }

        // 306-308: 跳过非Ready状态的节点
        if node.Status() != api.NodeStatusReady {
            continue
        }

        // 311-313: 跳过已排除的节点
        if nodesExcluded[node.Info.ID] != nil {
            continue
        }

        // 316-318: 跳过超载的节点
        if node.sbxsInProgress.Count() > maxStartingInstancesPerNode {
            continue
        }

        // 320-324: 计算节点当前正在启动的CPU使用量
        cpuUsage := int64(0)
        for _, sbx := range node.sbxsInProgress.Items() {
            cpuUsage += sbx.CPUs
        }

        // 325-327: 选择CPU使用率最低的节点
        if leastBusyNode == nil || (node.CPUUsage.Load()+cpuUsage) < leastBusyNode.CPUUsage.Load() {
            leastBusyNode = node
        }
    }

    // 330-334: 返回结果
    if leastBusyNode != nil {
        return leastBusyNode, nil
    }

    return nil, fmt.Errorf("no node available")
}
```

**逐行解读**:
- **L299**: 遍历节点池中的所有节点
- **L301-303**: 防御性编程，处理并发删除导致的nil节点
- **L306**: `node.Status()` 检查节点健康状态，只选择Ready状态的节点
- **L311**: 跳过在本次重试中已经失败的节点
- **L316**: `maxStartingInstancesPerNode = 3` 限制每个节点同时启动的实例数
- **L320-324**: 计算节点当前正在启动的实例的CPU总使用量
- **L325**: 核心选择逻辑：选择 `已运行CPU + 正在启动CPU` 最小的节点
- **L334**: 如果没有找到任何可用节点，返回错误

## 节点数据结构

**文件位置**: `packages/api/internal/orchestrator/node.go:28-45`

```go
type Node struct {
    CPUUsage atomic.Int64       // 已运行实例的CPU使用量（原子操作）
    RamUsage atomic.Int64       // 已运行实例的内存使用量（原子操作）
    Client   *GRPCClient        // gRPC客户端，用于与节点通信

    Info           *node.NodeInfo    // 节点基本信息（ID、IP等）
    orchestratorID string            // Orchestrator实例ID
    commit         string            // 代码提交hash
    version        string            // 版本号
    status         api.NodeStatus    // 节点状态
    statusMu       sync.RWMutex      // 状态读写锁

    sbxsInProgress *smap.Map[*sbxInProgress]  // 正在启动的沙盒映射

    buildCache *ttlcache.Cache[string, interface{}]  // 构建缓存

    createFails atomic.Uint64       // 创建失败计数（原子操作）
}

type sbxInProgress struct {
    MiBMemory int64    // 预占内存大小（MB）
    CPUs      int64    // 预占CPU核心数
}
```

## 算法特点和优势

### 1. 多层过滤机制
```go
// 过滤条件优先级（从高到低）：
1. node != nil              // 节点存在性检查
2. Status == Ready          // 节点健康状态检查  
3. not in nodesExcluded     // 避免重复选择失败节点
4. sbxsInProgress <= 3      // 防止节点过载
5. CPU使用率最低            // 负载均衡
```

### 2. 原子操作保证并发安全
```go
// 所有关键指标都使用原子操作
CPUUsage atomic.Int64       // 避免并发读写竞争
RamUsage atomic.Int64       // 保证数据一致性
createFails atomic.Uint64   // 线程安全的失败计数
```

### 3. 预占机制防止资源冲突
```go
// 在实际创建前预占资源
node.sbxsInProgress.Insert(sandboxID, &sbxInProgress{
    MiBMemory: build.RamMb,
    CPUs:      build.Vcpu,
})
// 创建失败时立即释放
node.sbxsInProgress.Remove(sandboxID)
```

### 4. 智能重试和错误处理
```go
// 重试策略：
- 最多重试3次
- 每次重试排除之前失败的节点
- 记录详细的失败信息和遥测数据
- 超时保护防止无限等待
```

## 算法复杂度分析

- **时间复杂度**: O(N×M)，其中N是节点数，M是最大重试次数(3)
- **空间复杂度**: O(N)，主要是nodesExcluded映射的存储
- **并发安全**: 通过原子操作和读写锁保证线程安全

## 监控和可观测性

```go
// 关键监控点：
1. telemetry.ReportEvent() - 记录关键事件
2. telemetry.ReportError() - 记录错误信息
3. log.Printf() - 详细的失败日志
4. node.createFails.Add(1) - 节点失败统计
5. 分布式追踪span - 性能分析
```

## 总结

E2B Infrastructure的节点选择算法是一个高度优化的智能调度系统，具有以下特点：

1. **高可用性**: 多重重试机制和故障转移
2. **负载均衡**: 基于CPU使用率的智能选择
3. **并发安全**: 原子操作和预占机制
4. **可观测性**: 完整的监控和日志记录
5. **性能优化**: 恢复操作优先选择原节点，减少数据传输

这个算法确保了沙盒创建的成功率和系统的整体稳定性，是E2B Infrastructure能够提供高质量服务的关键技术基础。

---

**文件位置**: `packages/api/internal/orchestrator/create_instance.go`  
**核心函数**: `CreateSandbox`, `getLeastBusyNode`, `findLeastBusyNode`  
**算法复杂度**: O(N×M) 时间, O(N) 空间