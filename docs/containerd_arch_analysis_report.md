# containerd 架構解析報告

> Distributed Systems 期中報告 — 架構解析  
> NCCU CS, Chun-Feng Liao  
> 分析對象：containerd v2.x（CNCF Graduated）

---

## 目錄

1. [問題背景與設計動機](#1-問題背景與設計動機)
2. [靜態結構](#2-靜態結構)
   - 2.1 Component Diagram（整體）
   - 2.2 Plugin 依賴 Component Diagram
   - 2.3 Class Diagram：核心介面
3. [動態結構](#3-動態結構)
   - 3.1 Sequence：Image Pull
   - 3.2 Sequence：Container Run（ctr run）
   - 3.3 Sequence：Kubernetes Pod 建立（Sandbox API）
   - 3.4 Activity：Container 生命週期
   - 3.5 Activity：Plugin 初始化
4. [此設計如何解決問題](#4-此設計如何解決問題)

---

## 1. 問題背景與設計動機

### 舊架構的問題（Docker monolithic daemon）

Docker 最初是一個「all-in-one daemon」——所有功能（build、network、volume、runtime）耦合在同一個 process 裡。

```
問題列表：
1. Daemon 是容器的 parent process
   → daemon crash = 所有容器 SIGHUP，全部死亡

2. Storage 後端（graphdriver）與 image import/export 邏輯高度耦合
   → 想換 storage 後端需要改大量核心代碼

3. Runtime 硬編碼為 runc
   → 無法替換成 gVisor（沙箱）、Kata（VM-based）、WasmEdge（WASM）

4. 沒有標準化介面
   → Kubernetes 必須維護 dockershim 才能使用 Docker 作為 runtime
   → Kubernetes 1.24 起移除 dockershim

5. 多個系統（Docker + k8s）無法共用同一個底層 runtime
   → 各自管理資源，主機上有重複的 image 存放
```

### containerd 的回應

containerd 的設計目標是：**做最小的事，定義最清晰的介面，讓上層系統能安全嵌入。**

```
核心設計原則：

1. Shim 是容器的 parent（不是 daemon）  → 解決 daemon crash 問題
2. Snapshotter 介面分離 storage 與 image 邏輯 → 解決 storage 耦合問題
3. Runtime v2 ttRPC 介面抽象 runtime 實作   → 解決 runtime 鎖死問題
4. CRI plugin 提供標準 Kubernetes 介面       → 解決 k8s 整合問題
5. Namespace 多租戶 + Content-Addressed 共享 → 解決多系統共存問題
```

---

## 2. 靜態結構

### 2.1 整體 Component Diagram

```mermaid
graph TB
    subgraph Clients["External Clients"]
        CTR["ctr CLI\ncmd/ctr/"]
        K8S["kubelet (CRI gRPC)\nkubernetes"]
        DOCKER["Docker / nerdctl\nclient/"]
    end

    subgraph Daemon["containerd Daemon  cmd/containerd/"]
        subgraph API["API Layer  plugins/server/"]
            GRPC["gRPC Server\nplugins/server/grpc/"]
            TTRPC["ttRPC Server\nplugins/server/ttrpc/"]
        end

        subgraph Services["Service Layer  plugins/services/"]
            TSVC["tasks-service"]
            CSVC["containers-service"]
            ISVC["images-service"]
            SSVC["snapshots-service"]
            NSVC["namespaces-service"]
            SBSVC["sandbox-service"]
        end

        subgraph CoreLayer["Core Layer  core/"]
            META["Metadata DB\ncore/metadata/db.go\n(bbolt, namespaced)"]
            EVENTS["Events Exchange\ncore/events/exchange/"]
        end

        subgraph Runtime["Runtime Layer  core/runtime/v2/"]
            SHIMMGR["ShimManager\nshim_manager.go"]
            TASKMGR["TaskManager\ntask_manager.go"]
        end

        subgraph Storage["Storage Layer  plugins/snapshots/ + content/"]
            CONTENT["Content Store\nplugins/content/local/\nSHA256-addressed"]
            OVERLAY["overlayfs\nplugins/snapshots/overlay/"]
            BTRFS["btrfs\nplugins/snapshots/btrfs/"]
            EROFS["erofs\nplugins/snapshots/erofs/"]
        end

        CRI["CRI Plugin\nplugins/cri/\ninternal/cri/server/"]
    end

    subgraph ShimProc["Shim Process  cmd/containerd-shim-runc-v2/"]
        SHIM["containerd-shim-runc-v2\n(independent OS process)"]
        RUNC["runc\n(OCI runtime engine)"]
    end

    CTR -->|gRPC| GRPC
    K8S -->|CRI gRPC| CRI
    DOCKER -->|gRPC| GRPC

    GRPC --> Services
    CRI --> Services
    Services --> META
    Services --> TASKMGR
    TASKMGR --> SHIMMGR
    SHIMMGR -->|"fork/exec\n+ ttRPC"| SHIM
    SHIM -->|"fork/exec"| RUNC
    Services --> CONTENT
    Services --> OVERLAY
    META --> EVENTS
```

---

### 2.2 Plugin 依賴 Component Diagram

> 展示 `registry.Register()` 宣告的 `Requires` 關係，決定初始化順序。

```mermaid
graph TB
    C["ContentPlugin\nplugins/content/local/\n(base, no deps)"]
    S["SnapshotPlugin\nplugins/snapshots/overlay/\n(base, no deps)"]
    E["EventPlugin\ncore/events/exchange/\n(base, no deps)"]
    D["DiffPlugin\nplugins/diff/walking/\n← Content"]
    M["MetadataPlugin\nplugins/metadata/\n← Content + Snapshots"]
    L["LeasePlugin\nplugins/leases/local/\n← Metadata"]
    SH["ShimPlugin\ncore/runtime/v2/shim_manager.go\n← Event + Metadata"]
    RT["RuntimePluginV2\ncore/runtime/v2/task_manager.go\n← Shim + Mount"]
    SVC["ServicePlugins\nplugins/services/\n← Metadata + Runtime + Diff"]
    GC["GCPlugin\nplugins/gc/scheduler.go\n← Metadata"]
    GRPC["GRPCPlugin\nplugins/server/grpc/\n← All Services"]
    CRI["CRIPlugin (GRPCPlugin)\nplugins/cri/\n← Metadata + Runtime + Snapshots"]

    C --> M
    C --> D
    S --> M
    E --> SH
    M --> SH
    M --> L
    M --> SVC
    M --> GC
    SH --> RT
    RT --> SVC
    D --> SVC
    SVC --> GRPC
    SVC --> CRI
    M --> CRI
    RT --> CRI
```

**初始化機制**：`cmd/containerd/builtins/builtins_linux.go` 用 blank import 觸發所有 `init()`，server 啟動時對 plugin registry 做 topological sort，依序呼叫 `InitFn`。

---

### 2.3 Class Diagram：核心介面

#### Snapshotter 介面（`core/snapshots/snapshotter.go`）

```mermaid
classDiagram
    class Snapshotter {
        <<interface>>
        +Prepare(ctx, key, parent string) []mount.Mount
        +View(ctx, key, parent string) []mount.Mount
        +Commit(ctx, name, key string) error
        +Remove(ctx, key string) error
        +Stat(ctx, key string) Info
        +Update(ctx, info Info) Info
        +Walk(ctx, fn WalkFunc) error
        +Mounts(ctx, key string) []mount.Mount
        +Usage(ctx, key string) Usage
        +Close() error
    }

    class Info {
        +Kind Kind
        +Name string
        +Parent string
        +Labels map~string,string~
        +Created time.Time
        +Updated time.Time
    }

    class Kind {
        <<enumeration>>
        KindActive
        KindCommitted
        KindView
    }

    class overlaySnapshotter {
        -root string
        -ms MetaStore
    }

    class nativeSnapshotter {
        -root string
    }

    Snapshotter <|.. overlaySnapshotter : implements
    Snapshotter <|.. nativeSnapshotter  : implements
    Snapshotter --> Info                : returns
    Info        --> Kind                : has
```

#### Runtime v2 介面（`api/runtime/task/v3/shim.proto`）

```mermaid
classDiagram
    class TaskService {
        <<interface ttRPC>>
        +State(ctx, req) StateResponse
        +Create(ctx, req) CreateTaskResponse
        +Start(ctx, req) StartResponse
        +Delete(ctx, req) DeleteResponse
        +Exec(ctx, req) ExecProcessResponse
        +Kill(ctx, req) KillResponse
        +Wait(ctx, req) WaitResponse
        +Pause(ctx, req) PauseResponse
        +Resume(ctx, req) ResumeResponse
        +Checkpoint(ctx, req) CheckpointTaskResponse
        +Shutdown(ctx, req) ShutdownResponse
        +Stats(ctx, req) StatsResponse
        +CloseIO(ctx, req) CloseIOResponse
        +ResizePty(ctx, req) ResizePtyResponse
    }

    class ShimManager {
        -shims sync.Map
        -events Exchange
        -cs ContainerStore
        -ss SandboxStore
        +Start(ctx, id, opts) ShimInstance
        +Get(ctx, id) ShimInstance
        +Delete(ctx, id) error
    }

    class ShimInstance {
        -pid int
        -socket string
        -bundle string
        +Task() TaskService
        +Sandbox() SandboxService
        +ID() string
        +Namespace() string
    }

    class TaskManager {
        -shims ShimManager
        -mounts MountManager
        +Create(ctx, taskInfo) Task
        +Get(ctx, id) Task
        +Tasks(ctx, all) []Task
        +Delete(ctx, id) ExitStatus
    }

    ShimManager     --> ShimInstance : manages
    ShimInstance    --> TaskService  : exposes via ttRPC
    TaskManager     --> ShimManager  : uses
```

#### Metadata DB 與相關型別（`core/metadata/db.go`）

```mermaid
classDiagram
    class DB {
        -db   Transactor (bbolt)
        -ss   map~string,snapshotter~
        -cs   contentStore
        -wlock sync.RWMutex
        +NewContainerStore() containers.Store
        +NewImageStore() images.Store
        +NewSnapshotStore(name) snapshots.Store
        +NewSandboxStore() sandbox.Store
        +GarbageCollect(ctx) GCStats
    }

    class ContainerStore {
        <<interface>>
        +Get(ctx, id) Container
        +List(ctx, filters) []Container
        +Create(ctx, c Container) Container
        +Update(ctx, c Container) Container
        +Delete(ctx, id) error
    }

    class ImageStore {
        <<interface>>
        +Get(ctx, name) Image
        +List(ctx, filters) []Image
        +Create(ctx, image) Image
        +Update(ctx, image) Image
        +Delete(ctx, name) error
    }

    class Container {
        +ID string
        +Labels map~string,string~
        +Image string
        +Runtime RuntimeInfo
        +Spec *anypb.Any
        +Snapshotter string
        +SnapshotKey string
        +SandboxID string
    }

    DB --> ContainerStore : creates
    DB --> ImageStore     : creates
    ContainerStore --> Container : manages
```

---

## 3. 動態結構

### 3.1 Sequence Diagram：Image Pull

> 呼叫路徑：`ctr pull` → `client/pull.go` → `core/transfer/local/pull.go` → registry → content store → snapshotter → metadata DB

```mermaid
sequenceDiagram
    participant ctr
    participant Client as client/pull.go
    participant Transfer as core/transfer/local
    participant Fetcher as core/remotes/docker/fetcher.go
    participant Content as plugins/content/local
    participant Unpack as core/unpack/unpacker.go
    participant Snap as plugins/snapshots/overlay
    participant Meta as core/metadata/db.go

    ctr->>Client: Pull("docker.io/library/alpine:latest")
    Client->>Transfer: Transfer(ctx, src=RemoteSource, dst=ImageStore)

    Transfer->>Fetcher: Resolve(ref) → descriptor{digest}
    Fetcher-->>Transfer: manifest descriptor

    Transfer->>Fetcher: Fetch(manifest digest)
    Fetcher-->>Transfer: manifest JSON bytes
    Transfer->>Content: Write + Commit(sha256:manifest)

    loop for each layer in manifest
        Transfer->>Fetcher: Fetch(layer digest)
        Fetcher-->>Transfer: layer tar.gz stream
        Transfer->>Content: Write stream chunks
        Transfer->>Content: Commit(sha256:layer)
        Note over Content: 檔案存入 /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/<digest>
    end

    Transfer->>Unpack: Unpack(image, snapshotter="overlayfs")

    loop for each layer（bottom to top）
        Unpack->>Snap: Prepare(key="unpack-N", parent=prevChainID)
        Snap-->>Unpack: []Mount{overlay options}
        Note over Unpack: mount + unpack tar diff into mountpoint
        Unpack->>Snap: Commit(name=chainID_N, key)
        Note over Snap: snapshot 成為 committed，可作為下一層 parent
    end

    Transfer->>Meta: images.Create(name="alpine:latest", digest=sha256:manifest)
    Meta-->>Transfer: Image stored
    Transfer-->>Client: Image{}
    Client-->>ctr: Pull complete
```

---

### 3.2 Sequence Diagram：Container Run（ctr run）

> 完整路徑：`ctr run` → gRPC → `plugins/services/tasks` → `core/runtime/v2/task_manager.go` → `shim_manager.go` → `containerd-shim-runc-v2` → `runc`

```mermaid
sequenceDiagram
    participant ctr
    participant GRPC as plugins/server/grpc
    participant ContSvc as plugins/services/containers
    participant TaskSvc as plugins/services/tasks
    participant TaskMgr as core/runtime/v2/task_manager.go
    participant ShimMgr as core/runtime/v2/shim_manager.go
    participant Shim as containerd-shim-runc-v2
    participant Runc as runc binary

    ctr->>GRPC: ContainersService.Create(id, image, spec, snapshotter)
    GRPC->>ContSvc: Create(container)
    ContSvc->>ContSvc: Snapshotter.Prepare(id, imageChainID) → active snapshot
    ContSvc->>ContSvc: Generate OCI spec (config.json) into bundle dir
    ContSvc->>ContSvc: metadata.DB.containers.Create(container)
    ContSvc-->>GRPC: Container{id}
    GRPC-->>ctr: ContainerID

    ctr->>GRPC: TasksService.Create(containerID, stdin/stdout/stderr fifos)
    GRPC->>TaskSvc: Create(task)
    TaskSvc->>TaskMgr: Create(ctx, taskCreateInfo)
    TaskMgr->>ShimMgr: Start(id, runtime="io.containerd.runc.v2")

    Note over ShimMgr: 將 runtime name 轉換為 binary name<br/>"io.containerd.runc.v2" → "containerd-shim-runc-v2"
    ShimMgr->>Shim: exec containerd-shim-runc-v2 start<br/>(stdin: BootstrapParams protobuf)
    Note over Shim: 讀取 BootstrapParams<br/>啟動 ttRPC server 監聽 unix socket
    Shim-->>ShimMgr: BootstrapResult{address="unix:///run/containerd/s/xxx"}

    ShimMgr->>Shim: ttRPC TaskService.CreateTaskRequest{bundle, rootfs mounts, io fifos}
    Shim->>Runc: exec runc create --bundle <path>
    Runc-->>Shim: container PID（寫入 state.json）
    Shim-->>ShimMgr: CreateTaskResponse{pid}
    ShimMgr-->>TaskMgr: ShimInstance
    TaskMgr-->>TaskSvc: Task{id, pid}
    TaskSvc-->>GRPC: Task
    GRPC-->>ctr: TaskID

    ctr->>GRPC: TasksService.Start(taskID)
    GRPC->>TaskSvc: Start
    TaskSvc->>Shim: ttRPC TaskService.StartRequest
    Shim->>Runc: exec runc start <id>
    Runc-->>Shim: OK
    Shim-->>TaskSvc: StartResponse
    TaskSvc-->>ctr: OK

    Note over ctr,Runc: Container is now running<br/>Shim 是 container process 的 parent（不是 containerd daemon）

    ctr->>GRPC: TasksService.Wait(taskID)
    GRPC->>Shim: ttRPC TaskService.WaitRequest
    Note right of Shim: block on waitpid()
    Runc-->>Shim: process exit (code=0)
    Shim->>Shim: Publish TaskExitEvent to containerd
    Shim-->>GRPC: WaitResponse{exitStatus=0}
    GRPC-->>ctr: ExitCode=0

    ctr->>GRPC: TasksService.Delete(taskID)
    GRPC->>Shim: ttRPC DeleteRequest
    Shim-->>GRPC: DeleteResponse
    GRPC->>Shim: ttRPC ShutdownRequest
    ShimMgr->>Shim: exec containerd-shim-runc-v2 delete
    ShimMgr->>ShimMgr: cleanup bundle directory
    ContSvc->>ContSvc: Snapshotter.Remove(containerKey)
    GRPC-->>ctr: OK
```

---

### 3.3 Sequence Diagram：Kubernetes Pod 建立（Sandbox API）

> Kubelet → containerd CRI plugin → Sandbox API → Shim → Container

```mermaid
sequenceDiagram
    participant Kubelet
    participant CRI as internal/cri/server
    participant SandboxCtrl as core/sandbox controller
    participant CNI as CNI Plugin
    participant ShimMgr as ShimManager
    participant Shim as containerd-shim-*
    participant TaskSvc as TaskService (ttRPC)

    Note over Kubelet,TaskSvc: Phase 1 — RunPodSandbox

    Kubelet->>CRI: CRI RunPodSandbox(PodSandboxConfig)
    CRI->>CRI: Create sandbox metadata in bolt DB
    CRI->>CNI: Setup network namespace + assign PodIP
    CNI-->>CRI: NetworkNamespace, IP

    CRI->>SandboxCtrl: Controller.Create(sandboxID, config)
    SandboxCtrl->>ShimMgr: Start shim for sandbox
    ShimMgr->>Shim: exec shim start（BootstrapParams）
    Shim-->>ShimMgr: socket address
    SandboxCtrl->>Shim: SandboxService.CreateSandbox(req)
    Shim-->>SandboxCtrl: OK

    SandboxCtrl->>Shim: SandboxService.StartSandbox(req)
    Shim-->>SandboxCtrl: SandboxPID, Endpoint

    CRI->>CRI: Store sandbox endpoint in metadata DB
    CRI-->>Kubelet: PodSandboxID

    Note over Kubelet,TaskSvc: Phase 2 — CreateContainer

    Kubelet->>CRI: CRI CreateContainer(PodSandboxID, ContainerConfig)
    CRI->>Shim: SandboxService.Platform（取得 OS/Arch）
    Shim-->>CRI: Platform{os=linux, arch=amd64}
    CRI->>CRI: Prepare container snapshot (reuse sandbox network ns)
    CRI->>CRI: Create container metadata（linked to sandboxID）
    CRI-->>Kubelet: ContainerID

    Note over Kubelet,TaskSvc: Phase 3 — StartContainer

    Kubelet->>CRI: CRI StartContainer(ContainerID)
    CRI->>CRI: Lookup sandbox shim endpoint
    CRI->>Shim: TaskService.Create（reuse sandbox shim connection）
    Shim-->>CRI: TaskPID
    CRI->>Shim: TaskService.Start
    Shim-->>CRI: OK
    CRI-->>Kubelet: OK

    Note over Kubelet,TaskSvc: Phase 4 — StopPodSandbox

    Kubelet->>CRI: CRI StopPodSandbox(PodSandboxID)
    loop for each container in pod
        CRI->>Shim: TaskService.Kill(SIGTERM)
        CRI->>Shim: TaskService.Delete
        Shim-->>CRI: OK
    end
    CRI->>SandboxCtrl: Controller.Stop(sandboxID)
    SandboxCtrl->>Shim: SandboxService.StopSandbox
    Shim-->>SandboxCtrl: OK
    CRI->>CNI: Teardown network namespace
    CRI-->>Kubelet: OK
```

---

### 3.4 Activity Diagram：Container 生命週期

```mermaid
stateDiagram-v2
    [*] --> Created : NewContainer()\n+ NewTask()\n→ runc create

    Created --> Running : task.Start()\n→ runc start

    Running --> Paused : task.Pause()\n→ SIGSTOP to cgroup
    Paused --> Running : task.Resume()\n→ SIGCONT to cgroup

    Running --> Stopped : process exits (natural)\nOR task.Kill(SIGTERM/SIGKILL)

    Stopped --> [*] : task.Delete()\n+ container.Delete()\n→ snapshot.Remove()

    note right of Created
        Snapshot: active（writable）
        OCI spec: config.json in bundle
        Shim: alive（ttRPC server）
        runc: created but not started
    end note

    note right of Running
        Shim: process parent
        runc: running PID
        Event: TaskStartEvent published
    end note

    note right of Stopped
        ExitCode: recorded by shim
        Shim: still alive（waiting Delete）
        Event: TaskExitEvent published
    end note
```

---

### 3.5 Activity Diagram：Plugin 初始化（Daemon 啟動）

```mermaid
flowchart TD
    A([containerd 啟動]) --> B[解析 config.toml]
    B --> C["blank imports 觸發所有 plugin init()\n→ registry 收集所有 Registration"]
    C --> D[讀取環境 disabled plugins]
    D --> E[Topological Sort by Requires 依賴]
    E --> F{For each plugin\nin order}

    F --> G{config 中是否\ndisabled？}
    G -- Yes --> H[標記 disabled，跳過]
    G -- No --> I["呼叫 plugin.InitFn(InitContext)"]

    I --> J{"InitFn\n回傳 error？"}
    J -- No --> K[註冊到 plugin.Set]
    J -- Yes --> L{此 plugin 被\n其他 plugin 依賴？}
    L -- Yes --> M([Fatal: daemon 退出])
    L -- No --> N[Log warning，跳過]

    K --> F
    H --> F
    N --> F
    F -- 全部完成 --> O[啟動 gRPC server]
    O --> P[啟動 ttRPC server]
    P --> Q([Daemon ready\nsystemd notify])
```

**關鍵設計**：`InitFn` 透過 `ic.GetSingle(plugins.MetadataPlugin)` 取得依賴，runtime 注入，不是靜態 import。

---

## 4. 此設計如何解決問題

### 問題一：Daemon crash → 容器全死

**解法：Shim as Container Parent（`core/runtime/v2/shim_manager.go`）**

```
傳統 Docker：
  dockerd ──parent──→ container process
  dockerd crash → container SIGHUP

containerd：
  containerd ──fork/exec──→ containerd-shim-runc-v2
                                    │ parent
                                    ▼
                            runc ──fork──→ container process

  containerd crash：
    shim process 繼續存活（reparented to init by OS）
    container 繼續執行
    containerd restart → ShimManager.loadExistingShims()
                       → 重新連接既有的 shim unix socket
                       → 無縫恢復控制
```

**實作位置**：`core/runtime/v2/shim_manager.go` `Start()` + `shim_load.go` `loadExistingShims()`

**效果**：滿足 Availability NFR。Shim 的設計使 containerd 不是 SPOF（Single Point of Failure）。

---

### 問題二：Runtime 鎖死 runc

**解法：Runtime v2 ttRPC 介面抽象（`api/runtime/task/v3/shim.proto`）**

```
containerd 只認識 TaskService ttRPC 介面：
  Create / Start / Kill / Wait / Delete / Stats...

不同 shim binary 各自實作此介面：
  io.containerd.runc.v2  → containerd-shim-runc-v2 → runc（Linux containers）
  io.containerd.runhcs.v2→ containerd-shim-runhcs-v2→ hcsshim（Windows）
  io.containerd.kata.v2  → containerd-shim-kata-v2  → Kata（VM-based）
  io.containerd.wasm.v1  → containerd-shim-wasm-v1  → WasmEdge（WASM）

選擇 runtime 的方式（`core/runtime/v2/task_manager.go`）：
  runtime name "io.containerd.runc.v2"
      → 移除點 → "io-containerd-runc-v2"
      → 取最後兩段 → "runc-v2"
      → 加前綴 → "containerd-shim-runc-v2"
      → 在 $PATH 找 binary 執行
```

**效果**：新 runtime 只需實作 `TaskService` proto，不需修改 containerd 任何代碼。

---

### 問題三：Storage 後端（graphdriver）耦合 image 邏輯

**解法：Snapshotter 介面 + 職責分離**

```
舊 Docker graphdriver：
  image pull → graphdriver → 解包 tar → 管理 layer 關係 → 提供 rootfs
  （graphdriver 知道 image、layer、tar 格式）

containerd Snapshotter：
  只懂 Prepare / Commit / View / Remove
  不知道什麼是 image 或 container
  上層由 core/unpack/unpacker.go 負責解包 tar 並呼叫 Snapshotter

分工：
  core/remotes/docker/fetcher.go → 從 registry 拿 bytes
  plugins/content/local/store.go → 以 SHA256 存入磁碟
  core/unpack/unpacker.go        → 讀 content，解壓，呼叫 Snapshotter.Prepare/Commit
  plugins/snapshots/overlay/     → 只管 overlayfs lowerdir/upperdir

換 storage 後端只需替換 Snapshotter 實作，不改 fetcher 或 unpacker。
```

**實作位置**：`core/snapshots/snapshotter.go`（介面）、`plugins/snapshots/`（各後端）

---

### 問題四：多系統共存資源衝突

**解法：Namespace 多租戶 + Content-Addressed Storage**

```go
// pkg/namespaces/context.go
ctx := namespaces.WithNamespace(context.Background(), "k8s.io")
// 所有 API 呼叫都帶著 namespace，metadata DB 的 KV key 包含 namespace prefix

// core/metadata/db.go 的 bucket 結構：
// v1/<namespace>/containers/<id>  → container spec
// v1/<namespace>/images/<name>    → image ref → digest
// v1/<namespace>/snapshots/...    → namespace-isolated snapshot metadata
// v1/blobsizes/<digest>           → 所有 namespace 共享 content（以 digest 為 key）
```

**效果**：
- Docker（`moby` namespace）與 Kubernetes（`k8s.io` namespace）完全獨立
- `alpine:latest` image 在兩個 namespace 各有一個 name entry，但 content（layers）只存一份
- 符合 Multi-tenancy + Space Efficiency 兩個目標

---

### 問題五：Kubernetes Pod sandbox 無抽象（hardcoded pause container）

**解法：Sandbox API（`core/sandbox/controller.go`）**

```
舊實作：
  CRI plugin 直接 hardcode "pause container" 作為 sandbox
  → VM-based runtime（Kata）必須自己 fork 整個 CRI plugin 才能改 sandbox 實作

Sandbox API（v2.0 stable）：
  CRI plugin 呼叫 SandboxController.Create/Start/Stop
  Controller 是介面，有兩個實作：

  1. shim controller（core/runtime/v2/）
     → 呼叫 shim binary 的 SandboxService RPC
     → 適合 Kata（shim 本身管理 VM sandbox）

  2. podsandbox controller（internal/cri/server/podsandbox/）
     → 啟動 pause container 作為 sandbox
     → 適合一般 runc-based runtime
```

**效果**：Runtime 作者可以自訂 sandbox 實作，不需修改 CRI plugin 或 containerd 核心。

---

### 設計問題與解法對應表（簡報用）

| 設計問題 | 根本原因 | containerd 解法 | 關鍵程式碼位置 |
|---------|---------|----------------|--------------|
| Daemon crash → 容器死 | Daemon 是 container parent | Shim 作為獨立 process parent | `core/runtime/v2/shim_manager.go` |
| Runtime 鎖死 runc | 硬編碼 runc 呼叫 | ttRPC TaskService 介面抽象 | `api/runtime/task/v3/shim.proto` |
| Storage 耦合 image 邏輯 | graphdriver 知道太多 | Snapshotter 介面：只管快照 | `core/snapshots/snapshotter.go` |
| 多系統資源衝突 | 沒有隔離層 | Namespace + Content-Addressed | `core/metadata/db.go` |
| k8s Pod sandbox 無法替換 | pause container hardcoded | Sandbox Controller 介面 | `core/sandbox/controller.go` |
| Runtime 啟動順序難管理 | 靜態依賴 | Plugin registry + topo sort | `cmd/containerd/server/server.go` |

---

*分析基於 containerd v2.x main branch，原始碼路徑：`/Users/pengqize/Documents/code/distributed_system/mid_pres/containerd/`*
