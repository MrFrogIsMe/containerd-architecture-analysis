# containerd 架構解析 — 四張 Mermaid 圖

---

## 圖一：Class Diagram（靜態物件層級）

> 呈現 Container 為根，向下展開所有依賴物件的層級關係。

```mermaid
classDiagram
    direction TB

    class Container {
        +String id
        +String image
        +String snapshotKey
        +String snapshotter
        +String sandboxID
        +RuntimeInfo runtime
        +OCISpec spec
        +createTask() Task
    }

    class Image {
        +String name
        +Descriptor target
        +Map labels
    }

    class Descriptor {
        +String digest
        +int64 size
        +String mediaType
    }

    class ContentStore {
        <<interface>>
        +ReaderAt(descriptor) ReaderAt
        +Writer(ref) Writer
        +Info(digest) Info
        +Delete(digest) error
    }

    class Blob {
        +String digest
        +int64 size
        +bytes data
    }

    class Snapshot {
        +String name
        +String parent
        +Kind kind
    }

    class Kind {
        <<enumeration>>
        Active
        Committed
        View
    }

    class Snapshotter {
        <<interface>>
        +Prepare(key, parent) Mount
        +Commit(name, key)
        +View(key, parent) Mount
        +Remove(key)
        +Stat(key) Info
    }

    class Mount {
        +String type
        +String source
        +String target
        +List options
    }

    class OCISpec {
        +Process process
        +List~Mount~ mounts
        +List~Namespace~ namespaces
        +Cgroups cgroups
    }

    class RuntimeInfo {
        +String name
        +Any options
    }

    class Task {
        <<interface>>
        +String id
        +int pid
        +start()
        +kill(signal)
        +wait() ExitStatus
        +delete()
        +exec() ExecProcess
    }

    class ShimManager {
        +start(id, runtime) ShimInstance
        +get(id) ShimInstance
        +delete(id)
    }

    class ShimInstance {
        +String socketAddr
        +int pid
        +TaskService task()
        +SandboxService sandbox()
    }

    class TaskService {
        <<interface ttRPC>>
        +Create(req) TaskResponse
        +Start(req)
        +Kill(req)
        +Wait(req) ExitStatus
        +Delete(req)
        +Shutdown(req)
    }

    %% Container 頂層 → 第二層
    Container "1" --> "1" Image         : references
    Container "1" --> "1" OCISpec       : has spec
    Container "1" --> "1" RuntimeInfo   : uses runtime
    Container "1" --> "0..1" Snapshot   : snapshotKey
    Container "1" --> "0..1" Task       : creates (lazily)

    %% Image 層
    Image "1" --> "1" Descriptor        : target (manifest)
    Descriptor "*" --> "1" ContentStore : stored in
    ContentStore "1" *-- "*" Blob       : stores blobs

    %% Snapshot 層
    Snapshot --> Kind                   : kind
    Snapshot "0..1" --> "0..1" Snapshot : parent
    Snapshotter --> Snapshot            : manages
    Snapshotter --> Mount               : returns mounts

    %% Task → Shim → runc 層
    Task "1" --> "1" ShimManager        : managed by
    ShimManager "1" *-- "*" ShimInstance: owns
    ShimInstance --> TaskService        : exposes via ttRPC
```

**層級說明：**
```
Container（容器靜態定義）
├── Image（映像名稱 + manifest digest）
│   └── Descriptor（digest, size, mediaType）
│       └── ContentStore（以 SHA256 存所有 layer bytes）
│           └── Blob（實際 bytes，檔案系統上的一個 blob）
├── OCISpec（容器執行規格：process, mounts, namespaces, cgroups）
├── RuntimeInfo（要使用哪個 shim binary）
├── Snapshot（container 的可寫 rootfs，由 Snapshotter 管理）
│   ├── parent Snapshot（唯讀 image layers 疊加鏈）
│   └── Kind：Active（可寫）/ Committed（唯讀）/ View（唯讀臨時）
└── Task（執行中的 container process）
    └── ShimManager（管理 shim 生命週期）
        └── ShimInstance（獨立 process，持有 unix socket）
            └── TaskService via ttRPC（呼叫 runc）
```

> **關鍵修正**：Snapshotter 不認識 Image，只接受 `key`（字串）和 `parent`（字串）。  
> Image layer → ContentStore → Unpacker → Snapshotter.Prepare/Commit 是三層，Snapshotter 在最底。

---

## 圖二：Component Diagram（插件系統靜態架構）

> 呈現 containerd 的插件依賴結構：誰依賴誰、誰初始化誰。  
> 與 Sequence/Activity 不同，這張圖看的是「系統元件的靜態連線關係」。

```mermaid
graph TB
    subgraph Clients["External Clients"]
        K8S["kubelet\nCRI gRPC"]
        DOCKER["Docker / nerdctl\ngRPC"]
        CTR["ctr CLI\ngRPC"]
    end

    subgraph Daemon["containerd Daemon  (single process)"]
        subgraph API["API Layer"]
            GRPC["gRPC Server\nnamespace middleware + TLS"]
            CRI["CRI Plugin\n(Kubernetes interface)"]
        end

        subgraph SVC["Service Layer  (ServicePlugin)"]
            TSVC["Task Service"]
            CSVC["Container Service"]
            ISVC["Image Service"]
            SSVC["Snapshot Service"]
        end

        subgraph CORE["Core Layer"]
            META["Metadata DB\nbbolt KV\n(namespaced)"]
            EVENTS["Events Exchange\n(pub/sub)"]
            GC["GC Scheduler"]
        end

        subgraph RT["Runtime Layer"]
            SHIMMGR["ShimManager\nPlugin: ShimPlugin"]
            TASKMGR["TaskManager\nPlugin: RuntimePluginV2"]
        end

        subgraph STORE["Storage Layer  (Plugins)"]
            CONTENT["Content Store\nSHA256 blobs\n/var/lib/containerd/..."]
            OVERLAY["overlayfs\nSnapshotter"]
            BTRFS["btrfs\nSnapshotter"]
            EROFS["erofs\nSnapshotter"]
        end
    end

    subgraph ShimProc["Shim Processes  (independent OS processes)"]
        SHIM1["shim-runc-v2\n(PID A)"]
        SHIM2["shim-runc-v2\n(PID B)"]
        RUNC["runc\n(OCI engine)"]
    end

    K8S -->|CRI gRPC| CRI
    DOCKER -->|gRPC| GRPC
    CTR -->|gRPC| GRPC

    GRPC --> SVC
    CRI --> SVC

    TSVC --> TASKMGR
    CSVC --> META
    ISVC --> META
    ISVC --> CONTENT
    SSVC --> OVERLAY
    SSVC --> BTRFS
    SSVC --> EROFS

    TASKMGR --> SHIMMGR
    SHIMMGR --> META
    SHIMMGR --> EVENTS

    META --> GC
    META --> CONTENT
    META --> OVERLAY

    SHIMMGR -->|"fork/exec\n+ ttRPC socket"| SHIM1
    SHIMMGR -->|"fork/exec\n+ ttRPC socket"| SHIM2
    SHIM1 -->|"fork/exec"| RUNC
    SHIM2 -->|"fork/exec"| RUNC

    note1["⚠ Shim 是獨立 process\nDaemon crash 不影響容器"]
```

**此圖說明的核心設計問題：**

```
問題：storage 後端應該可以替換
解法：所有 Snapshotter 實作同一個介面，Snapshot Service 上層完全不知道底層是 overlay 還是 btrfs

問題：daemon crash 不能讓容器死
解法：Shim 是獨立 OS process，container 的 parent 是 shim，不是 daemon

問題：不同系統（Docker / k8s）不能干擾彼此
解法：gRPC 中間有 namespace middleware，所有 API 呼叫都帶 namespace
```

---

## 圖三：Sequence Diagram（Container Run 互動流程）

> 呈現 `ctr run` 時，各元件之間「誰跟誰說話、說什麼、順序如何」。

```mermaid
sequenceDiagram
    actor User
    participant Upper as Docker / Kubernetes
    participant Containerd as containerd daemon
    participant ContSvc as Container Service
    participant TaskSvc as Task Service
    participant ShimMgr as ShimManager
    participant Shim as containerd-shim-runc-v2
    participant Runc as runc
    participant Kernel as Linux Kernel

    User->>Upper: run container request
    Upper->>Containerd: gRPC CreateContainer(image, spec, snapshotter)

    Containerd->>ContSvc: Create(container)
    ContSvc->>ContSvc: Snapshotter.Prepare(id, imageChainID)
    Note right of ContSvc: 準備可寫 rootfs snapshot<br/>（Snapshotter 只知道 key + parent，不知道 Image）
    ContSvc->>ContSvc: Generate OCI spec → config.json
    ContSvc->>ContSvc: metadata.DB.containers.Create()
    ContSvc-->>Containerd: Container{id}

    Containerd->>TaskSvc: CreateTask(containerID, stdio fifos)
    TaskSvc->>ShimMgr: Start(id, runtime="io.containerd.runc.v2")
    Note right of ShimMgr: "io.containerd.runc.v2"<br/>→ "containerd-shim-runc-v2" binary

    ShimMgr->>Shim: exec containerd-shim-runc-v2 start
    Note over Shim: 讀 BootstrapParams from stdin<br/>啟動 ttRPC server on unix socket
    Shim-->>ShimMgr: BootstrapResult {socket address}

    ShimMgr->>Shim: ttRPC TaskService.CreateTaskRequest
    Shim->>Runc: exec runc create --bundle <path>
    Runc->>Kernel: Setup namespaces (pid, net, mnt, uts...)
    Runc->>Kernel: Setup cgroups (cpu, memory limits)
    Runc->>Kernel: Setup mounts (overlay rootfs)
    Kernel-->>Runc: OK
    Runc-->>Shim: container PID
    Shim-->>ShimMgr: CreateTaskResponse {pid}
    ShimMgr-->>TaskSvc: Task ready
    TaskSvc-->>Containerd: Task{id, pid}
    Containerd-->>Upper: Container running
    Upper-->>User: show running status

    Note over Shim,Runc: Shim 是容器的 parent process<br/>Daemon crash → Shim 繼續活著 → 容器不死
```

---

## 圖四：Activity Diagram（啟動容器完整流程）

> 呈現 containerd 收到請求後，每一步的決策與行動。

```mermaid
flowchart TD
    A([收到啟動容器請求]) --> B[解析 image 名稱與 runtime spec]
    B --> C{本機有此 image？}

    C -->|No| D[從 Registry 拉取 image]
    D --> E[將 manifest + layer bytes\n存入 Content Store\n以 SHA256 digest 為 key]
    E --> F[Unpacker 解壓各 layer\n呼叫 Snapshotter.Prepare → Commit\n建立 committed snapshot 鏈]

    C -->|Yes| G[取得 image manifest\n查找已有 snapshot ChainID]
    F --> H
    G --> H

    H[Snapshotter.Prepare\n建立 Active Snapshot\n可寫的 rootfs 層]
    H --> I[建立 Container metadata\nid, image, snapshotKey, OCISpec\n存入 Metadata DB]
    I --> J[建立 Task\nfork shim binary\ncontainerd-shim-runc-v2]

    J --> K[Shim 啟動 ttRPC server\n回傳 socket address]
    K --> L[containerd → Shim\nTaskService.CreateTask\nrootfs mounts + stdio fifos]
    L --> M[Shim 呼叫 runc create\n準備 OCI bundle]

    M --> N[runc 設定 Linux namespaces]
    N --> O[runc 設定 cgroups]
    O --> P[runc 設定 mounts / overlay rootfs]
    P --> Q[runc 啟動 container process\nfork → execve]

    Q --> R{啟動成功？}
    R -->|Yes| S[Shim 發布 TaskStartEvent\n狀態回傳至 containerd → 上層]
    R -->|No| T[回傳 error\n清除 snapshot + metadata]

    S --> U([Container Running])
    T --> V([回報錯誤])

    style J fill:#ffd6a5
    style K fill:#ffd6a5
    note1["⬆ Shim 是獨立 OS process\n不依附 daemon 存活"]
```

---

## 四張圖的分工與說明重點

| 圖 | 類型 | 回答的問題 | 說明重點 |
|----|------|-----------|---------|
| Class Diagram | 靜態 | 系統有哪些物件？層級關係是什麼？ | Container 是根，Task 是執行實體，Snapshotter 不認識 Image |
| Component Diagram | 靜態 | 模組怎麼連線？Shim 為什麼獨立？ | Plugin 架構、Shim 是獨立 process、Namespace 隔離 |
| Sequence Diagram | 動態 | 啟動時誰跟誰說話、說什麼？ | containerd 是協調者，shim → runc → kernel 的呼叫鏈 |
| Activity Diagram | 動態 | 啟動流程的每一步決策是什麼？ | image check → unpack → snapshot → task → shim → runc |

**兩張靜態（Class + Component）+ 兩張動態（Sequence + Activity）**  
不重複：Sequence 看的是「訊息傳遞」，Activity 看的是「流程決策」，完全互補。
