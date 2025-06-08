# containerd CPU/MEM 垂直修改机制分析

> 基于 containerd v2.1.0 版本分析

## 概述

containerd 通过完整的调用链实现了不重启容器的资源垂直修改，整个过程包含以下几个关键层次：从 Kubernetes CRI 接口到 Linux cgroups 的完整资源更新流程。

## 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Kubernetes 层                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  kubelet → CRI Client → UpdateContainerResources()                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ CRI gRPC 调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          containerd CRI 服务层                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  func (c *criService) UpdateContainerResources()                      │ │
│  │  📁 internal/cri/server/container_update_resources.go:38              │ │
│  │                                                                        │ │
│  │  1️⃣ 获取容器信息: c.containerStore.Get()                               │ │
│  │  2️⃣ 更新 OCI spec: updateContainerSpec()                              │ │
│  │  3️⃣ 事务更新状态: container.Status.UpdateSync()                        │ │
│  │  4️⃣ 调用底层更新: task.Update()                                         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ containerd API 调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        containerd 客户端层                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  func (t *task) Update(ctx, opts ...UpdateTaskOpts)                   │ │
│  │  📁 client/task.go:620                                                 │ │
│  │                                                                        │ │
│  │  1️⃣ 构建 UpdateTaskRequest                                              │ │
│  │  2️⃣ 序列化资源配置: typeurl.MarshalAny(resources)                        │ │
│  │  3️⃣ 调用 TaskService: t.client.TaskService().Update()                  │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ ttrpc/gRPC 调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       containerd-shim-runc-v2 层                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  func (s *service) Update(ctx, r *taskAPI.UpdateTaskRequest)          │ │
│  │  📁 cmd/containerd-shim-runc-v2/task/service.go:561                   │ │
│  │                                                                        │ │
│  │  1️⃣ 获取容器: s.getContainer(r.ID)                                      │ │
│  │  2️⃣ 调用容器更新: container.Update(ctx, r)                              │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 进程内调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Container 容器层                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  func (c *Container) Update(ctx, r *task.UpdateTaskRequest)           │ │
│  │  📁 cmd/containerd-shim-runc-v2/runc/container.go:465                 │ │
│  │                                                                        │ │
│  │  1️⃣ 获取 Init 进程: c.Process("")                                       │ │
│  │  2️⃣ 调用进程更新: p.(*process.Init).Update(ctx, r.Resources)            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 进程内调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Process Init 层                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  func (p *Init) Update(ctx, r *google_protobuf.Any)                   │ │
│  │  📁 cmd/containerd-shim-runc-v2/process/init.go:454                   │ │
│  │                                                                        │ │
│  │  1️⃣ 状态检查: p.initState.Update(ctx, r)                                │ │
│  │  2️⃣ 资源解析: json.Unmarshal(r.Value, &resources)                       │ │
│  │  3️⃣ 调用 runc: p.runtime.Update(ctx, p.id, &resources)                 │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 状态机检查
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           状态机层                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  runningState.Update() / pausedState.Update()                         │ │
│  │  📁 cmd/containerd-shim-runc-v2/process/init_state.go:255             │ │
│  │                                                                        │ │
│  │  ✅ runningState: 允许更新                                              │ │
│  │  ✅ pausedState: 允许更新                                               │ │
│  │  ❌ stoppedState: 拒绝更新                                              │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ runc 命令调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            runc 运行时层                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  func (r *Runc) Update(ctx, id, resources *specs.LinuxResources)      │ │
│  │  📁 vendor/github.com/containerd/go-runc/runc.go:692                  │ │
│  │                                                                        │ │
│  │  1️⃣ JSON 序列化: json.NewEncoder(buf).Encode(resources)                 │ │
│  │  2️⃣ 执行命令: runc update --resources=- container_id                    │ │
│  │  3️⃣ 通过 stdin 传递 JSON 数据                                           │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 系统调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Linux cgroups 层                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  🐧 Linux 内核 cgroups 接口                                             │ │
│  │                                                                        │ │
│  │  📂 /sys/fs/cgroup/                                                    │ │
│  │    ├── 📄 cpu.cfs_quota_us     (CPU 配额)                              │ │
│  │    ├── 📄 cpu.cfs_period_us    (CPU 周期)                              │ │
│  │    ├── 📄 cpu.shares           (CPU 权重)                              │ │
│  │    ├── 📄 memory.limit_in_bytes (内存限制)                              │ │
│  │    └── 📄 memory.memsw.limit_in_bytes (内存+交换限制)                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 数据流向图

```
UpdateContainerResourcesRequest
         │
         ▼
   ┌─────────────┐    updateOCIResource()    ┌──────────────┐
   │ CRI Request │────────────────────────▶  │ OCI Spec     │
   └─────────────┘                          │ (persistent) │
         │                                  └──────────────┘
         ▼
   ┌─────────────┐    WithResources()       ┌──────────────┐
   │ UpdateTask  │────────────────────────▶  │ UpdateTask   │
   │ Opts        │                          │ Info         │
   └─────────────┘                          └──────────────┘
         │                                         │
         ▼                                         ▼
   ┌─────────────┐    typeurl.MarshalAny()  ┌──────────────┐
   │ LinuxRes    │────────────────────────▶  │ protobuf     │
   │ ources      │                          │ Any          │
   └─────────────┘                          └──────────────┘
         │                                         │
         ▼                                         ▼
   ┌─────────────┐    JSON Marshal          ┌──────────────┐
   │ specs.Linux │────────────────────────▶  │ JSON bytes   │
   │ Resources   │                          │              │
   └─────────────┘                          └──────────────┘
         │                                         │
         ▼                                         ▼
   ┌─────────────┐    runc update cmd       ┌──────────────┐
   │ runc        │────────────────────────▶  │ Linux        │
   │ process     │                          │ cgroups      │
   └─────────────┘                          └──────────────┘
```

## 详细分析

### 1. CRI API 层 (用户接口)

**文件位置**: `internal/cri/server/container_update_resources.go`

```go
// kubernetes 通过 CRI 接口调用
func (c *criService) UpdateContainerResources(ctx context.Context, r *runtime.UpdateContainerResourcesRequest) 
```

**功能**:
- 提供标准的 Kubernetes CRI 接口
- 支持 CPU 配额、内存限制等资源参数的动态调整
- 包含资源验证和转换逻辑

**核心代码**:
```go
// 第38行开始的主要逻辑
container, err := c.containerStore.Get(r.GetContainerId())
if err != nil {
    return nil, fmt.Errorf("failed to find container: %w", err)
}

// NRI 插件处理
resources := r.GetLinux()
updated, err := c.nri.UpdateContainerResources(ctx, &sandbox, &container, resources)

// 事务性更新
if err := container.Status.UpdateSync(func(status containerstore.Status) (containerstore.Status, error) {
    return c.updateContainerResources(ctx, container, r, status)
}); err != nil {
    return nil, fmt.Errorf("failed to update resources: %w", err)
}
```

### 2. containerd CRI 服务层 (资源管理)

**文件位置**: `internal/cri/server/container_update_resources_linux.go`

**核心处理逻辑**:
- **容器状态检查**: 确保容器正在运行且未被删除
- **OCI 规范更新**: 更新容器的 OCI spec，使其成为资源限制的 "source of truth"
- **事务性操作**: 使用 `container.Status.UpdateSync()` 确保原子性更新
- **运行时分发**: 根据容器状态决定是否需要立即应用到运行中的容器

**关键函数**:
```go
// 第31行: Linux平台的OCI资源更新
func updateOCIResource(ctx context.Context, spec *runtimespec.Spec, r *runtime.UpdateContainerResourcesRequest, config criconfig.Config) (*runtimespec.Spec, error)

// 应用资源配置
if err := opts.WithResources(r.GetLinux(), config.TolerateMissingHugetlbController, config.DisableHugetlbController)(ctx, nil, nil, &cloned); err != nil {
    return nil, fmt.Errorf("unable to set linux container resources: %w", err)
}
```

### 3. containerd 客户端层 (任务管理)

**文件位置**: `client/task.go:620`

```go
func (t *task) Update(ctx context.Context, opts ...UpdateTaskOpts) error
```

**功能**:
- 将资源配置封装为 `UpdateTaskInfo`
- 序列化 `specs.LinuxResources` 为 protobuf 格式
- 调用底层 TaskService 进行实际更新

**核心代码**:
```go
// 构建更新请求
request := &tasks.UpdateTaskRequest{
    ContainerID: t.id,
}

// 处理资源配置
if i.Resources != nil {
    r, err := typeurl.MarshalAny(i.Resources)
    if err != nil {
        return err
    }
    request.Resources = typeurl.MarshalProto(r)
}

// 调用底层服务
_, err := t.client.TaskService().Update(ctx, request)
```

### 4. Shim 进程层 (容器生命周期)

**文件位置**: `cmd/containerd-shim-runc-v2/task/service.go:561`

```go
func (s *service) Update(ctx context.Context, r *taskAPI.UpdateTaskRequest) (*ptypes.Empty, error)
```

**功能**:
- containerd-shim-runc-v2 作为容器运行时的代理
- 处理容器级别的资源更新请求
- 将请求转发给具体的容器实例

**实现**:
```go
container, err := s.getContainer(r.ID)
if err != nil {
    return nil, err
}
if err := container.Update(ctx, r); err != nil {
    return nil, errgrpc.ToGRPC(err)
}
return empty, nil
```

### 5. Container 进程层 (状态机管理)

**文件位置**: `cmd/containerd-shim-runc-v2/runc/container.go:465`

```go
func (c *Container) Update(ctx context.Context, r *task.UpdateTaskRequest) error
```

**关键机制**:
- **状态机模式**: 不同状态下的资源更新行为不同
  - `runningState`: 允许资源更新
  - `pausedState`: 允许资源更新  
  - `stoppedState`: 拒绝资源更新
- **资源解析**: 将 protobuf 数据反序列化为 `specs.LinuxResources`

**状态机实现** (`cmd/containerd-shim-runc-v2/process/init_state.go:255`):
```go
// runningState 允许更新
func (s *runningState) Update(ctx context.Context, r *google_protobuf.Any) error {
    return s.p.update(ctx, r)
}

// pausedState 也允许更新
func (s *pausedState) Update(ctx context.Context, r *google_protobuf.Any) error {
    return s.p.update(ctx, r)
}

// stoppedState 拒绝更新
func (s *stoppedState) Update(ctx context.Context, r *google_protobuf.Any) error {
    return errors.New("cannot update a stopped container")
}
```

### 6. runc 运行时层 (底层实现)

**文件位置**: `vendor/github.com/containerd/go-runc/runc.go:692`

```go
func (r *Runc) Update(context context.Context, id string, resources *specs.LinuxResources) error
```

**最终实现**:
- 将 `LinuxResources` 序列化为 JSON
- 通过命令行调用：`runc update --resources=- <container_id>`
- runc 直接操作 Linux cgroups 来应用新的资源限制

**核心代码**:
```go
buf := getBuf()
defer putBuf(buf)

// JSON 序列化资源配置
if err := json.NewEncoder(buf).Encode(resources); err != nil {
    return err
}

// 构建runc命令
args := []string{"update", "--resources=-", id}
cmd := r.command(context, args...)
cmd.Stdin = buf

// 执行命令
return r.runOrError(cmd)
```

### 7. Linux cgroups 层 (内核接口)

runc 最终通过 Linux cgroups 接口实现资源控制：

**控制文件**:
- **CPU 控制**: `cpu.cfs_quota_us`, `cpu.cfs_period_us`, `cpu.shares`
- **内存控制**: `memory.limit_in_bytes`, `memory.memsw.limit_in_bytes`
- **cgroups v1/v2 兼容**: 自动检测并使用相应的 cgroups 版本

## 关键优势

1. **无需重启**: 直接修改运行中容器的 cgroups 配置
2. **原子性操作**: 通过事务确保更新过程的一致性
3. **向后兼容**: 同时支持 cgroups v1 和 v2
4. **状态一致性**: 更新 OCI spec 确保配置的持久化
5. **错误恢复**: 失败时自动回滚到原始配置

## 实际工作流程

1. Kubernetes 发起 `UpdateContainerResources` 请求
2. containerd 更新容器的 OCI specification
3. 如果容器正在运行，同时调用 task.Update()
4. shim 进程接收更新请求并转发给容器
5. 容器进程根据当前状态决定是否应用更新
6. runc 通过 `runc update` 命令修改 cgroups
7. Linux 内核立即应用新的资源限制

## 关键文件位置总览

```
📁 containerd/
├── 📁 internal/cri/server/
│   ├── 📄 container_update_resources.go:38          # CRI 服务入口
│   ├── 📄 container_update_resources_linux.go:31   # Linux 特定实现
│   └── 📄 container_update_resources_windows.go    # Windows 特定实现
│
├── 📁 client/
│   ├── 📄 task.go:620                              # Task.Update() 方法
│   └── 📄 task_opts.go:207                         # WithResources() 选项
│
├── 📁 cmd/containerd-shim-runc-v2/
│   ├── 📁 task/
│   │   └── 📄 service.go:561                       # Shim 服务层
│   ├── 📁 runc/
│   │   └── 📄 container.go:465                     # 容器层
│   └── 📁 process/
│       ├── 📄 init.go:454                          # Init 进程
│       └── 📄 init_state.go:255                    # 状态机
│
├── 📁 vendor/github.com/containerd/go-runc/
│   └── 📄 runc.go:692                              # runc 命令封装
│
└── 📁 integration/
    └── 📄 container_update_resources_test.go       # 集成测试
```

## 测试用例

containerd 项目包含了完整的集成测试来验证资源更新功能：

**文件**: `integration/container_update_resources_test.go`

主要测试场景：
- `TestUpdateContainerResources_MemorySwap`: 测试内存和交换空间限制的更新
- `TestUpdateContainerResources_MemoryLimit`: 测试内存限制的更新
- `TestUpdateContainerResources_StatusUpdated`: 测试容器状态的正确更新

这些测试验证了从 CRI 接口到 cgroups 的完整调用链，确保资源更新功能的正确性和稳定性。

## containerd v2.1.0 支持的完整原地修改能力

除了基本的 CPU 和内存限制之外，containerd v2.1.0 还支持多种其他资源的原地修改能力：

### Linux 容器资源 (LinuxContainerResources)

#### 1. **CPU 资源控制**
```go
type LinuxContainerResources struct {
    // CPU CFS (Completely Fair Scheduler) 周期 (纳秒)
    CpuPeriod int64 `json:"cpu_period,omitempty"`
    
    // CPU CFS 配额 (纳秒)
    CpuQuota int64 `json:"cpu_quota,omitempty"`
    
    // CPU 权重 (相对于其他容器)
    CpuShares int64 `json:"cpu_shares,omitempty"`
    
    // CPU 亲和性：允许的逻辑 CPU 集合
    CpusetCpus string `json:"cpuset_cpus,omitempty"`
    
    // 内存节点亲和性：允许的内存节点集合
    CpusetMems string `json:"cpuset_mems,omitempty"`
}
```

**对应的 cgroups 文件**:
- `cpu.cfs_period_us`: CPU 调度周期
- `cpu.cfs_quota_us`: CPU 时间配额
- `cpu.shares`: CPU 权重
- `cpuset.cpus`: CPU 亲和性
- `cpuset.mems`: 内存节点亲和性

#### 2. **内存资源控制**
```go
type LinuxContainerResources struct {
    // 内存限制 (字节)
    MemoryLimitInBytes int64 `json:"memory_limit_in_bytes,omitempty"`
    
    // 内存+交换空间限制 (字节)
    MemorySwapLimitInBytes int64 `json:"memory_swap_limit_in_bytes,omitempty"`
}
```

**对应的 cgroups 文件**:
- `memory.limit_in_bytes`: 内存限制
- `memory.memsw.limit_in_bytes`: 内存+交换限制

#### 3. **OOM (Out of Memory) 控制**
```go
type LinuxContainerResources struct {
    // OOM killer 调整分数 (-1000 到 1000)
    OomScoreAdj int64 `json:"oom_score_adj,omitempty"`
}
```

**对应的系统文件**:
- `/proc/[pid]/oom_score_adj`: 进程 OOM 分数调整

#### 4. **大页内存 (HugePages) 控制**
```go
type LinuxContainerResources struct {
    // 大页内存限制列表
    HugepageLimits []*HugepageLimit `json:"hugepage_limits,omitempty"`
}

type HugepageLimit struct {
    // 页面大小 (如: "2MB", "1GB")
    PageSize string `json:"page_size,omitempty"`
    
    // 限制值 (字节)
    Limit uint64 `json:"limit,omitempty"`
}
```

**对应的 cgroups 文件**:
- `hugetlb.<hugepagesize>.limit_in_bytes`: 大页内存限制

#### 5. **cgroups v2 统一接口 (Unified)**
```go
type LinuxContainerResources struct {
    // cgroups v2 统一资源配置
    Unified map[string]string `json:"unified,omitempty"`
}
```

**支持的 cgroups v2 控制器**:
- `memory.max`: 内存限制
- `memory.swap.max`: 交换空间限制
- `cpu.max`: CPU 配额和周期
- `cpu.weight`: CPU 权重
- `io.max`: IO 带宽限制
- `io.weight`: IO 权重
- `pids.max`: 进程数量限制

**示例配置**:
```go
Unified: map[string]string{
    "memory.max": "1073741824",      // 1GB 内存限制
    "cpu.max": "50000 100000",       // 50% CPU 配额
    "io.weight": "default 100",      // IO 权重
    "pids.max": "1024",              // 最大进程数
}
```

### Windows 容器资源 (WindowsContainerResources)

#### 1. **Windows CPU 控制**
```go
type WindowsContainerResources struct {
    // CPU 权重 (相对权重)
    CpuShares int64 `json:"cpu_shares,omitempty"`
    
    // 可用 CPU 数量
    CpuCount int64 `json:"cpu_count,omitempty"`
    
    // CPU 最大使用率 (百分比 × 100)
    CpuMaximum int64 `json:"cpu_maximum,omitempty"`
    
    // CPU 亲和性配置
    AffinityCpus []*WindowsCpuGroupAffinity `json:"affinity_cpus,omitempty"`
}
```

#### 2. **Windows 内存和存储控制**
```go
type WindowsContainerResources struct {
    // 内存限制 (字节)
    MemoryLimitInBytes int64 `json:"memory_limit_in_bytes,omitempty"`
    
    // 根文件系统/临时空间大小 (字节)
    RootfsSizeInBytes int64 `json:"rootfs_size_in_bytes,omitempty"`
}
```

### 任务级别的扩展配置

#### **自定义注解 (Annotations)**
```go
type UpdateContainerResourcesRequest struct {
    // 任意键值对，用于实验性或扩展资源配置
    Annotations map[string]string `json:"annotations,omitempty"`
}
```

**使用场景**:
- GPU 资源配置
- 网络 QoS 配置
- 自定义 cgroups 控制器
- 运行时特定配置

### 资源更新的完整支持矩阵

| 资源类型 | Linux | Windows | cgroups v1 | cgroups v2 | 原地更新 |
|---------|-------|---------|------------|------------|----------|
| CPU 配额/周期 | ✅ | ✅ | ✅ | ✅ | ✅ |
| CPU 权重 | ✅ | ✅ | ✅ | ✅ | ✅ |
| CPU 亲和性 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 内存限制 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 交换空间限制 | ✅ | ❌ | ✅ | ✅ | ✅ |
| OOM 分数调整 | ✅ | ❌ | ✅ | ✅ | ✅ |
| 大页内存 | ✅ | ❌ | ✅ | ✅ | ✅ |
| IO 限制 | ✅* | ❌ | ✅* | ✅ | ✅ |
| 进程数限制 | ✅* | ❌ | ✅* | ✅ | ✅ |
| 根文件系统大小 | ❌ | ✅ | ❌ | ❌ | ✅ |

> ✅ = 支持, ❌ = 不支持, ✅* = 通过 Unified 字段支持

### Pod Sandbox 资源更新

**当前状态**: containerd v2.1.0 中 `UpdatePodSandboxResources` 功能尚未实现：

```go
// 📁 internal/cri/server/sandbox_update_resources.go:25
func (c *criService) UpdatePodSandboxResources(ctx context.Context, r *runtime.UpdatePodSandboxResourcesRequest) (*runtime.UpdatePodSandboxResourcesResponse, error) {
    return nil, errors.New("not implemented yet")
}
```

### 实际使用示例

#### 1. **基本 CPU/内存更新**
```go
request := &runtime.UpdateContainerResourcesRequest{
    ContainerId: "container-123",
    Linux: &runtime.LinuxContainerResources{
        CpuPeriod:          100000,    // 100ms
        CpuQuota:           50000,     // 50ms (50% CPU)
        MemoryLimitInBytes: 1073741824, // 1GB
        MemorySwapLimitInBytes: 2147483648, // 2GB
    },
}
```

#### 2. **CPU 亲和性设置**
```go
request := &runtime.UpdateContainerResourcesRequest{
    ContainerId: "container-123",
    Linux: &runtime.LinuxContainerResources{
        CpusetCpus: "0-3",     // 使用 CPU 0-3
        CpusetMems: "0",       // 使用内存节点 0
    },
}
```

#### 3. **大页内存配置**
```go
request := &runtime.UpdateContainerResourcesRequest{
    ContainerId: "container-123",
    Linux: &runtime.LinuxContainerResources{
        HugepageLimits: []*runtime.HugepageLimit{
            {
                PageSize: "2MB",
                Limit:    1073741824, // 1GB of 2MB pages
            },
            {
                PageSize: "1GB", 
                Limit:    2147483648, // 2GB of 1GB pages
            },
        },
    },
}
```

#### 4. **cgroups v2 高级配置**
```go
request := &runtime.UpdateContainerResourcesRequest{
    ContainerId: "container-123",
    Linux: &runtime.LinuxContainerResources{
        Unified: map[string]string{
            "memory.max":    "1073741824",  // 1GB 内存
            "cpu.max":       "50000 100000", // 50% CPU
            "io.max":        "8:0 rbps=2097152 wbps=1048576", // IO 限制
            "pids.max":      "1024",        // 最大进程数
        },
    },
}
```

#### 5. **实验性注解配置**
```go
request := &runtime.UpdateContainerResourcesRequest{
    ContainerId: "container-123",
    Annotations: map[string]string{
        "nvidia.com/gpu":           "1",     // GPU 配置
        "network.alpha.io/class":   "gold",  // 网络 QoS
        "custom.runtime.io/config": "high-perf", // 自定义配置
    },
}
```

### 技术限制和注意事项

1. **状态依赖**: 只有运行中或暂停的容器才能更新资源
2. **cgroups 可用性**: 某些资源控制器可能在系统中不可用
3. **向下兼容**: 新的 cgroups v2 功能在 v1 系统中不可用
4. **运行时支持**: 不同的容器运行时可能有不同的支持程度
5. **权限要求**: 某些资源修改需要特定的系统权限

---

这个调用链展示了从 Kubernetes 发起资源更新请求到最终修改 Linux cgroups 的完整流程，每一层都有明确的职责分工，确保了系统的模块化和可维护性。 