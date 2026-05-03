# Class Diagram 元件說明文件

> 對齊原始碼：containerd v2.x main branch
> 每個元件的來源 package 均標注

---

## 目錄

1. [Class Diagram 全圖](#class-diagram-全圖)
2. [層級總覽](#層級總覽)
3. [元件詳細說明（技術版）](#元件詳細說明技術版)
4. [元件詳細說明（ELI5 版）](#元件詳細說明eli5-版)
5. [彼此如何看待彼此](#彼此如何看待彼此)

---

## Class Diagram 全圖

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
        +note: 純 metadata，存 bolt DB
        +note: 不等於執行中的程序
    }

    class RuntimeInfo {
        +String name
        +Any options
        +note: e.g. io.containerd.runc.v2
        +note: ShimManager 用 name 找 binary
    }

    class Image {
        +String name
        +Descriptor target
        +note: name 是人類可讀的 ref
        +note: target 指向 manifest
    }

    class Descriptor {
        +String digest
        +int64 size
        +String mediaType
        +note: digest = ContentStore 的 key
        +note: 來自 OCI image-spec
    }

    class ContentStore {
        <<interface>>
        +ReaderAt(descriptor) ReaderAt
        +Writer(ref) Writer
        +Info(digest) Info
        +Delete(digest) error
        +note: SHA256-addressed
        +note: 所有 namespace 共享
    }

    class Snapshotter {
        <<interface>>
        +Prepare(key, parent string) Mount
        +Commit(name, key string)
        +View(key, parent string) Mount
        +Remove(key string)
        +Stat(key string) Info
        +note: 完全不知道 Image 存在
        +note: 只操作 key 和 parent 字串
    }

    class Snapshot {
        +String name
        +String parent
        +Kind kind
        +note: parent 是字串 key，非物件引用
    }

    class Kind {
        <<enumeration>>
        Active
        Committed
        View
    }

    class Mount {
        +String type
        +String source
        +List options
        +note: overlay: lowerdir/upperdir/workdir
        +note: 傳給 runc 掛載用
    }

    class OCISpec {
        +Process process
        +List~Mount~ mounts
        +List~Namespace~ namespaces
        +Cgroups cgroups
        +note: 來自 OCI runtime-spec
        +note: 定義容器的完整隔離規格
    }

    class Task {
        <<interface>>
        +PID() uint32
        +Start(ctx)
        +Kill(signal)
        +Wait() ExitStatus
        +Pause() / Resume()
        +Exec() ExecProcess
        +Delete()
        +note: in-memory，代表跑起來的程序
        +note: daemon 重啟後從 shim 重建
    }

    class ShimManager {
        +start(id, runtimeName) ShimInstance
        +get(ctx, id) ShimInstance
        +delete(ctx, id)
        +note: 維護 id→ShimInstance 的 map
        +note: fork shim binary，建立 ttRPC 連線
    }

    class ShimInstance {
        +String address
        +int version
        +Bundle() string
        +Namespace() string
        +TaskService() via ttRPC
        +SandboxService() via ttRPC
        +note: 代表一個獨立的 shim OS process
        +note: 是 container process 的真正 parent
    }

    %% 第一層：Container 的直接組成
    Container --> RuntimeInfo    : runtime（embedded 欄位）
    Container --> OCISpec        : spec（執行規格）
    Container --> Image          : image（name 字串引用）
    Container --> Snapshot       : snapshotKey（字串引用）
    Container --> Task           : 0..1 lazily created

    %% Image → ContentStore 鏈
    Image       --> Descriptor   : target（manifest ref）
    Descriptor  --> ContentStore : digest 是 ContentStore 的 key

    %% Snapshot 體系
    Snapshot --> Kind            : kind（Active/Committed/View）
    Snapshot "many" --> "0..1" Snapshot : parent\n（只能指向 Committed）
    Snapshotter --> Snapshot     : 管理（以 key 字串存取）
    Snapshotter --> Mount        : 回傳掛載資訊

    %% RuntimeInfo → ShimManager
    RuntimeInfo --> ShimManager  : name 用來查找 binary

    %% Task → Shim 鏈
    Container   --> ShimManager  : 建立 Task 時透過
    Task        --> ShimInstance : managed by（1:1 or 1:many）
    ShimManager --> ShimInstance : owns
```

---

## 層級總覽

```
Container（頂層：定義一個容器）
├── Image（我從哪個映像檔來的？）
│   └── Descriptor（manifest 的 digest + size + mediaType）
│       └── ContentStore（根據 digest 取得實際 bytes）
│           └── [磁碟上的 blobs/sha256/<digest> 檔案]
│
├── Snapshot（我的 rootfs 在哪？）
│   ├── Kind（這個 snapshot 是可寫/唯讀/臨時唯讀？）
│   └── parent Snapshot（下面疊的那些唯讀 image layer）
│       └── ... （一路往下到 base layer，parent=""）
│
├── RuntimeInfo（要用哪個 runtime binary？）
│   └── name → ShimManager → ShimInstance（fork 出來的 shim process）
│                               └── TaskService via ttRPC → runc
│
├── OCISpec（容器的隔離規格：process, mounts, namespaces, cgroups）
│   └── [寫成 bundle/config.json，傳給 shim/runc]
│
└── Task（0 or 1：實際跑起來的程序）
    └── ShimInstance（獨立 OS process，container 的 parent）
        └── runc → container process
```

---

## 元件詳細說明（技術版）

---

### Container
**來源**：`core/containers/containers.go`  
**性質**：純 metadata struct，存在 bolt DB 的 `v1/<namespace>/containers/<id>` bucket

Container 是容器的**靜態定義**，不代表任何執行中的 process。它只記錄「這個容器應該怎麼被執行」。

```go
type Container struct {
    ID          string          // 唯一識別，不可變
    Image       string          // image 名稱（字串引用，不是物件）
    Runtime     RuntimeInfo     // 要 fork 哪個 shim binary
    Spec        typeurl.Any     // OCI runtime spec（序列化 protobuf）
    SnapshotKey string          // rootfs snapshot 的 key
    Snapshotter string          // 用哪個 snapshotter（e.g. "overlayfs"）
    SandboxID   string          // 如果屬於某個 sandbox（k8s pod）
    Labels      map[string]string
}
```

**重要設計**：Container 不持有 Task 的引用，Task 是另外建立的。可以有 Container 但沒有 Task（已建立但未啟動）。

---

### RuntimeInfo
**來源**：`core/containers/containers.go`（Container 的 embedded struct）  
**性質**：Container 的一個欄位，不是獨立 class

```go
type RuntimeInfo struct {
    Name    string      // e.g. "io.containerd.runc.v2"
    Options typeurl.Any // shim-specific 設定（序列化 protobuf）
}
```

`Name` 的用途：ShimManager 呼叫 `resolveRuntimePath(name)`，轉換邏輯：
```
"io.containerd.runc.v2"
  → 取最後兩段 → "runc.v2"
  → 把 "." 換成 "-" → "runc-v2"
  → 加前綴 → "containerd-shim-runc-v2"
  → 在 $PATH 查找這個 binary
```

RuntimeInfo **不是一個獨立的 Runtime subsystem**，它只是告訴 ShimManager「去 PATH 找哪個 binary 來 fork」。

---

### Image
**來源**：`core/images/image.go`  
**性質**：metadata struct，存在 bolt DB 的 `v1/<namespace>/images/<name>` bucket

```go
type Image struct {
    Name      string               // e.g. "docker.io/library/alpine:latest"
    Target    ocispec.Descriptor   // 指向 manifest 的 descriptor
    Labels    map[string]string
    CreatedAt time.Time
}
```

Image 只有一個欄位有實質內容：`Target`（Descriptor）。Image 本身不存任何 bytes，只記錄「這個名稱對應到 ContentStore 裡的哪個 manifest digest」。

**Image 和 Container 的關係**：Container.Image 是字串（`"docker.io/library/alpine:latest"`），不是物件引用。containerd 不強制要求 Container.Image 指向一個真實存在的 Image——可以建立一個 Container 但已把 Image 刪掉（Image 的 bytes 仍在 ContentStore，只是 name 不見了）。

---

### Descriptor
**來源**：`github.com/opencontainers/image-spec/specs-go/v1`（OCI 標準，外部套件）  
**性質**：指向 ContentStore 中某個 blob 的指標

```go
type Descriptor struct {
    MediaType string        // e.g. "application/vnd.oci.image.manifest.v1+json"
    Digest    digest.Digest // e.g. "sha256:abc123..."
    Size      int64
}
```

`Descriptor.Digest` 就是 ContentStore 的 key。`ContentStore.ReaderAt(descriptor)` 根據 digest 讀出實際 bytes。

Descriptor 本身不是 containerd 的 class，是 OCI 標準定義的型別，但它是 Image 和 ContentStore 之間的橋樑。

---

### ContentStore
**來源**：`core/content/content.go`（interface）；實作在 `plugins/content/local/store.go`  
**性質**：interface，可替換後端；預設實作是本地檔案系統

ContentStore 是 **content-addressed storage**：所有 blob 以 SHA256 digest 為 key 儲存，相同內容只存一份，所有 namespace 共用。

```go
type Store interface {
    Provider    // ReaderAt(descriptor) → 讀
    Ingester    // Writer(ref) → 寫
    Manager     // Info/Walk/Delete
    IngestManager
}
```

實際存放位置：`/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/<digest>`

**ContentStore 不知道 Image、Layer、Container 的存在**。它只知道 digest 和 bytes。語義由上層（Image Service、Unpacker）賦予。

---

### Snapshotter
**來源**：`core/snapshots/snapshotter.go`（interface）；實作在 `plugins/snapshots/overlay/`、`btrfs/`、`zfs/` 等  
**性質**：interface，是 containerd 最重要的抽象之一

```go
type Snapshotter interface {
    Prepare(ctx, key, parent string) ([]mount.Mount, error)
    View(ctx, key, parent string) ([]mount.Mount, error)
    Commit(ctx, name, key string) error
    Remove(ctx, key string) error
    Stat(ctx, key string) (Info, error)
    Mounts(ctx, key string) ([]mount.Mount, error)
    Walk(ctx, fn WalkFunc) error
}
```

**最關鍵的設計**：Snapshotter 的所有方法只接受 **字串 key**，完全不知道 Image 或 Container 是什麼。這是刻意的——讓 Snapshotter 可以被任何 storage 後端實作，不限於容器場景。

`Prepare(key, parent)` 的語義：「給我一個以 parent 為基礎的可寫快照，key 是它的名字，回傳掛載資訊」。

---

### Snapshot（Info）
**來源**：`core/snapshots/snapshotter.go`  
**性質**：Snapshotter 管理的狀態單位

```go
type Info struct {
    Kind   Kind   // KindActive / KindCommitted / KindView
    Name   string // 這個 snapshot 的名字（就是 key）
    Parent string // parent snapshot 的名字（空字串 = base layer）
    Labels map[string]string
}
```

**parent 關係**：parent 是 **字串**，是另一個 Snapshot 的 Name。bolt storage 在 `CreateSnapshot` 時驗證 parent 存在：
```go
if parent != "" {
    spbkt = bkt.Bucket([]byte(parent))  // 找 parent bucket
    if spbkt == nil → ErrNotFound       // parent 不存在就報錯
}
```

**overlay 的 parent 鏈如何運作**：
```
image layer 0 → committed snapshot（parent=""）
image layer 1 → committed snapshot（parent=layer0）
image layer 2 → committed snapshot（parent=layer1）
container A   → active snapshot  （parent=layer2）← 可寫層

overlay 組合：
  lowerdir=layer2:layer1:layer0
  upperdir=containerA_upper（可寫）
  workdir=containerA_work
```

---

### Kind（enumeration）
**來源**：`core/snapshots/snapshotter.go`

| Kind | 意義 | 可否當 parent | 可否寫入 |
|------|------|-------------|---------|
| Active | `Prepare()` 建立，可寫 | ❌ 不能 | ✅ 可以 |
| Committed | `Commit()` 建立，唯讀 | ✅ 可以 | ❌ 不能 |
| View | `View()` 建立，臨時唯讀 | ❌ 不能 | ❌ 不能 |

---

### Mount
**來源**：`core/mount/mount.go`（type alias to syscall mount params）

```go
type Mount struct {
    Type    string   // "overlay", "bind", "tmpfs"
    Source  string   // device or directory
    Target  string   // mount point（由上層決定）
    Options []string // e.g. ["lowerdir=...", "upperdir=...", "workdir=..."]
}
```

Snapshotter 的 `Prepare()` 回傳 `[]Mount`。這些 Mount 描述「怎麼掛載這個 snapshot」，但不會自動掛載——由 TaskManager 在建立 container bundle 時執行實際的 mount syscall，再把 bundle 路徑傳給 shim。

---

### OCISpec
**來源**：`github.com/opencontainers/runtime-spec/specs-go`（OCI 標準）；在 containerd 中 `pkg/oci/spec.go` 有 `type Spec = specs.Spec`

OCI Runtime Spec 是容器的完整執行規格，寫成 bundle 目錄下的 `config.json`。

主要欄位（對應隔離機制）：

| 欄位 | 功能 | Linux 機制 |
|------|------|-----------|
| `Process` | 要跑的 process、環境變數、capabilities | execve |
| `Mounts` | 要掛載的 filesystem | mount syscall |
| `Linux.Namespaces` | 哪些 namespace 要隔離 | clone(2) flags |
| `Linux.Resources` | CPU、memory 限制 | cgroups |
| `Linux.Seccomp` | system call 白名單 | seccomp(2) |

**Container 持有 OCISpec 的方式**：`Container.Spec` 是 `typeurl.Any`（序列化的 protobuf），在建立 Task 時反序列化成 `specs.Spec`，寫入 bundle 目錄。

---

### Task
**來源**：`core/runtime/task.go`（interface）  
**性質**：in-memory 物件，代表一個**正在執行**的 container process

```go
type Task interface {
    Process                    // ID, State, Kill, Start, Wait, CloseIO
    PID(ctx) (uint32, error)
    Namespace() string
    Pause(ctx) error
    Resume(ctx) error
    Exec(ctx, id, opts) (ExecProcess, error)
    Checkpoint(ctx, path, opts) error
    Update(ctx, resources, annotations) error
    Delete(ctx, opts) (*runtime.Exit, error)
}
```

**Task 的實際實作**：`core/runtime/v2/shim.go` 的 `shim` struct 實作 Task interface，它持有一個對 ShimInstance 的 ttRPC 連線。每次 Task.Kill()、Task.Wait() 等呼叫，底層都是送一個 ttRPC request 給 ShimInstance。

**Container 和 Task 的關係**：Container 建立後 Task 不存在（`0..1`）。呼叫 `ctr run` 或 `NewTask()` 才建立 Task。daemon 重啟後，Task 從 ShimInstance 重建（重連已有的 shim socket）。

---

### ShimManager
**來源**：`core/runtime/v2/shim_manager.go`  
**性質**：Plugin（`plugins.ShimPlugin`），管理所有 ShimInstance 的生命週期

```go
type ShimManager struct {
    shims          *runtime.NSMap[ShimInstance]  // id → ShimInstance 的 map
    events         *exchange.Exchange             // 發布 TaskExit 等事件
    containers     containers.Store              // 讀 container metadata
    sandboxStore   sandbox.Store                 // 讀 sandbox metadata
    runtimePaths   sync.Map                      // runtime name → binary path 快取
    containerdAddress string                     // 自己的 gRPC socket 位址
}
```

ShimManager 做的事：
1. 收到 `Start(id, runtimeName)` → `resolveRuntimePath(runtimeName)` → `fork/exec` shim binary
2. 傳送 BootstrapParams（containerd 位址、namespace 等）給 shim 的 stdin
3. 接收 BootstrapResult（shim 的 unix socket 位址）
4. 建立 ttRPC 連線，包裝成 ShimInstance 回傳
5. Daemon 重啟時：掃描所有現存 shim socket，重新建立連線（`loadExistingShims()`）

---

### ShimInstance
**來源**：`core/runtime/v2/shim.go` 的 `shim` struct（實作 `ShimInstance` interface）  
**性質**：代表一個獨立 OS process（`containerd-shim-runc-v2` binary）

```go
type shim struct {
    bundle  *Bundle  // bundle 目錄（rootfs + config.json）
    client  any      // ttRPC 或 gRPC 連線
    address string   // unix socket 路徑
    version int      // shim API 版本
}
```

**ShimInstance 的三個關鍵性質**：

1. **獨立 OS process**：不在 containerd daemon 的 process 裡，有自己的 PID
2. **Container 的 parent process**：runc fork 出 container process 後，container 被 re-parent 到 shim（不是 containerd）
3. **Daemon 重啟後仍然存活**：containerd 可以透過 `address`（unix socket 路徑）重新連接

ShimInstance 暴露兩個 ttRPC service：
- `TaskService`：Create/Start/Kill/Wait/Delete/Exec（管理 container process）
- `SandboxService`（若 shim 支援）：管理 Pod sandbox 生命週期

---

## 元件詳細說明（ELI5 版）

---

### Container 是什麼

**比喻**：Container 是一張**食譜**。

食譜記錄了：
- 要用哪些食材（Image）
- 用哪種爐子（RuntimeInfo）
- 如何擺盤（OCISpec）
- 食材放在哪個架子上（SnapshotKey）

食譜本身不是一道菜，它只是文字說明。真正開始煮（執行 container），才會有一道菜（Task）。

---

### Image 是什麼

**比喻**：Image 是一個**地址索引**。

「docker.io/library/alpine:latest」這個名字，指向 ContentStore 裡面的一個 manifest。就像書的目錄頁——目錄本身不是內容，它告訴你去哪一頁找內容。

---

### Descriptor 是什麼

**比喻**：Descriptor 是一個**快遞單號**。

上面寫了：包裹大小（size）、包裹類型（mediaType）、包裹識別碼（digest）。ContentStore 就是倉庫，你拿著快遞單號去倉庫取貨。

---

### ContentStore 是什麼

**比喻**：ContentStore 是一個**智慧倉庫**。

這個倉庫有一個規則：**相同的東西只存一份，用內容的 SHA256 指紋來識別**。100 個容器都用同一個 alpine:latest，倉庫裡只有一份 alpine 的 layer。取貨時拿 SHA256 指紋去領，不用名字。

---

### Snapshotter 是什麼

**比喻**：Snapshotter 是一個**相片沖洗店**。

你帶著底片（image layers）來，說「幫我沖洗，並在上面加一層空白可以寫字的透明片（active snapshot）」。沖洗店只管沖洗和組合，不在乎照片裡拍的是什麼（不知道 Image 是什麼）。

組合結果是一疊透明片（overlay）：
- 最底層是 image layer 0（唯讀）
- 往上疊 layer 1、layer 2…（唯讀）
- 最頂層是你的透明片（可寫）

---

### Snapshot 的 parent 鏈是什麼

**比喻**：就像 Git 的 commit 鏈。

每個 committed snapshot 就像一個 git commit，parent 指向前一個 commit。最底部的 commit 沒有 parent（parent=""）。當你建立一個 active snapshot，等於是 git checkout 了一個新 branch，可以在上面寫東西，但不影響已有的 commits。

```
commit_0 (layer 0, parent="")
    └── commit_1 (layer 1, parent=commit_0)
            └── commit_2 (layer 2, parent=commit_1)
                    ├── active_A (container A 的可寫層, parent=commit_2)
                    └── active_B (container B 的可寫層, parent=commit_2)
```

---

### OCISpec 是什麼

**比喻**：OCISpec 是給 runc 的一張**施工說明書**。

上面寫：
- 隔離哪些資源（namespace：這個房間不和隔壁共用網路、PID）
- 限制多少資源（cgroups：這個容器最多用 2 個 CPU、512MB 記憶體）
- 掛載哪些目錄（mounts：把宿主機的這個目錄貼進容器的那個路徑）
- 跑什麼程式（process：執行 /bin/sh）

---

### Task 是什麼

**比喻**：Task 是一道**正在烹飪中的菜**。

食譜（Container）是靜態的，菜（Task）才是活的——它有 PID（這道菜在第幾號爐子上），可以暫停（Pause）、繼續（Resume）、停止（Kill）、等它完成（Wait）。

---

### ShimManager 是什麼

**比喻**：ShimManager 是**工頭**。

工頭維護一個名冊，記錄哪些師傅（ShimInstance）在工作。你說「幫我找一個能處理 runc 工作的師傅」，工頭就根據名字找到 binary，僱用一個新師傅（fork shim），然後把師傅的聯絡方式（unix socket）給你。

---

### ShimInstance 是什麼

**比喻**：ShimInstance 是一個**獨立的包工師傅**。

這個師傅住在工廠外面，有自己的聯絡方式（unix socket 地址）。他的工作是：
1. 幫你實際啟動 container（呼叫 runc）
2. 負責看管 container 直到它結束
3. 就算工廠（containerd daemon）關閉，師傅還在繼續工作
4. 工廠重開後，可以重新聯絡師傅，繼續管理容器

---

## 彼此如何看待彼此

### Container → Image

```
Container 只存 Image 的名稱字串
Container 不持有 Image 物件的引用
```

Container 不知道 Image 的內容，只知道一個名稱。真正需要的時候（建立 Task 時），才透過 Image Service 去查 manifest → 找到 snapshot 鏈。

如果 Image 被刪了，Container 不受影響（snapshot 還在）。

---

### Container → Snapshot

```
Container 只存 snapshotKey（字串）和 snapshotter（名稱字串）
Container 不直接呼叫 Snapshotter
```

建立 Task 時，TaskManager 拿 `snapshotKey` 去呼叫 `Snapshotter.Mounts(snapshotKey)`，取得掛載資訊，再放進 CreateTaskRequest 傳給 shim。

---

### Container → OCISpec

```
Container.Spec 是序列化的 protobuf bytes（typeurl.Any）
不是直接持有 specs.Spec 物件
```

建立 Task 時反序列化，寫成 `bundle/config.json`，傳給 shim → runc 使用。

---

### Snapshotter → Snapshot

```
Snapshotter 不持有 Snapshot 物件
Snapshotter 透過 key 字串操作 Snapshot
Snapshot 的 metadata 存在 bolt DB
```

Snapshotter 呼叫 `storage.CreateSnapshot(ctx, kind, key, parent)` 操作 bolt bucket，而不是直接操作 `Info` struct。`Stat(key)` 才從 bolt 讀出 `Info`。

---

### Snapshotter → Image

```
完全沒有關係
Snapshotter 不知道 Image 存在
```

是 `core/unpack/unpacker.go` 在中間橋接：

```
Image manifest（layer digest 列表）
    ↓ unpacker 讀取
ContentStore.ReaderAt(layerDescriptor)
    ↓ unpacker 解壓 tar
Snapshotter.Prepare(key, parent)  ← 建立可寫快照
    mount → 解壓進去
Snapshotter.Commit(chainID, key)  ← 提交為唯讀
```

---

### Task → ShimInstance

```
Task interface 的實作（shim struct）持有一個 ttRPC client
每個 Task 方法 = 一個 ttRPC call
```

```go
// Task.Kill 的實際實作
func (s *shim) Kill(ctx context.Context, signal uint32, all bool) error {
    // s.task() 回傳 ttRPC TaskServiceClient
    _, err := s.task(ctx).Kill(ctx, &task.KillRequest{...})
    return err
}
```

Task 和 ShimInstance 的關係是 **1:1 或 1:many**：
- 一般情況：1 shim process 管 1 container（1:1）
- Kubernetes pod：1 shim process 管 pod 內所有 container（1:many），以 sandbox label 分組

---

### ShimInstance → runc

```
ShimInstance 不知道 runc 的存在（依賴倒置）
ShimInstance 只知道自己要 exec 什麼 binary（由 RuntimeInfo.Name 決定）
runc 的路徑在 CreateTaskRequest.Options 裡傳入
```

shim 自己 fork/exec runc，runc 執行後：
1. 設定 namespaces / cgroups / mounts（根據 config.json）
2. execve container 的 entry process
3. runc 退出，container process 成為 shim 的 child（被 OS 自動 reparent）

---

### RuntimeInfo → ShimManager

```
RuntimeInfo 本身不做任何事，只是一筆資料
ShimManager 讀取 RuntimeInfo.Name，做 binary name 轉換
```

轉換邏輯（`resolveRuntimePath`）：
```
"io.containerd.runc.v2" → "containerd-shim-runc-v2"
"/usr/bin/my-shim"      → "/usr/bin/my-shim"（絕對路徑直接用）
```

---

## 設計意圖總結

| 設計選擇 | 原因 |
|---------|------|
| Container 只存 Image 名稱字串 | Image 可以刪除，Container 仍然可以從 snapshot 啟動 |
| Snapshotter 不知道 Image | Snapshotter 可以被任何 storage 後端替換，不限容器場景 |
| Container 和 Task 分開 | Container 是持久定義，Task 是暫時的執行狀態；daemon 重啟只影響 Task |
| RuntimeInfo 是字串而非物件 | Runtime 實作完全外部化，新 runtime 不需修改 containerd |
| ShimInstance 是獨立 OS process | Daemon crash 不影響容器；upgrade daemon 不中斷服務 |
| ContentStore 用 SHA256 定址 | 自動去重；100 個 container 共用同一 alpine layer |
| Snapshot parent 是字串 | Bolt DB key，不是 Go 物件引用；跨 process restart 也能找到 |
