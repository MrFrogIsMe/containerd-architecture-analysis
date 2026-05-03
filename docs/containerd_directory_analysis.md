# containerd 完整目錄與模組關係分析

> 基於原始碼掃描 + 關鍵檔案讀取

---

## 頂層目錄一覽

```
containerd/
├── api/          ← Protobuf / gRPC 介面定義（獨立 Go module）
├── bin/          ← 編譯產物
├── client/       ← Go SDK（對外使用的 client library）
├── cmd/          ← 可執行程式入口
├── contrib/      ← 周邊工具（apparmor, seccomp, fuzz, ansible）
├── core/         ← 核心業務邏輯（純 Go interface + 實作）
├── defaults/     ← 各平台預設值
├── docs/         ← 文件
├── integration/  ← 整合測試
├── internal/     ← 內部套件（CRI 實作、工具函式）
├── pkg/          ← 共用工具（archive, OCI spec, shim, tracing...）
├── plugins/      ← 插件註冊 + 具體實作
├── releases/     ← Release notes
├── script/       ← 建構 / 測試 shell 腳本
├── test/         ← E2E 測試輔助
└── version/      ← 版本資訊
```

---

## 各目錄詳細說明

### `api/` — 所有 Protobuf 定義

獨立 Go module（`github.com/containerd/containerd/api`），專門放 `.proto` 與生成的 `.pb.go`。

```
api/
├── events/          ← 事件型別：Container/Image/Task/Snapshot/Sandbox 事件
├── runtime/
│   ├── bootstrap/v1 ← Shim 啟動協議（BootstrapParams / BootstrapResult）
│   ├── sandbox/v1   ← SandboxService RPC（CreateSandbox/StartSandbox...）
│   └── task/v2,v3   ← TaskService RPC（Create/Start/Kill/Delete...）
├── services/        ← 外部 gRPC 服務定義
│   ├── containers/  ← ContainerService
│   ├── content/     ← ContentService
│   ├── diff/        ← DiffService
│   ├── events/      ← EventsService
│   ├── images/      ← ImagesService
│   ├── leases/      ← LeasesService
│   ├── namespaces/  ← NamespacesService
│   ├── sandbox/     ← SandboxService（external API）
│   ├── snapshots/   ← SnapshotsService
│   ├── tasks/       ← TasksService
│   ├── transfer/    ← TransferService
│   └── version/     ← VersionService
└── types/           ← 共用型別：Mount, Descriptor, Platform, Metrics
```

**角色**：只存放介面定義，不含任何業務邏輯。所有 gRPC/ttRPC 通訊都基於此。

---

### `core/` — 核心業務邏輯

**純 Go 介面 + bolt/記憶體實作**，不直接參與 plugin 系統，被 `plugins/` 引用。

```
core/
├── containers/    ← Container 資料結構定義（純 struct）
├── content/       ← Content Store 介面 + proxy + helpers
├── diff/          ← Differ 介面（apply / compare layers）
├── events/        ← Events 介面 + exchange（pub/sub 實作）
├── images/        ← Image 資料結構 + handlers + archive
├── introspection/ ← Introspection 介面
├── leases/        ← Lease 介面（防止 GC 意外刪除資源）
├── metadata/      ← bolt DB 核心（namespaced KV store for all metadata）
│   ├── db.go          ← DB struct，管理 bbolt + GC
│   ├── containers.go  ← container CRUD
│   ├── images.go      ← image CRUD
│   ├── snapshot.go    ← snapshot metadata
│   ├── sandbox.go     ← sandbox metadata
│   ├── content.go     ← content store proxy（namespace-aware）
│   └── gc.go          ← garbage collector（mark & sweep）
├── metrics/       ← cgroups v1/v2 metrics 收集
├── mount/         ← Mount 操作（overlayfs, loopback, idmapped）
├── remotes/       ← Registry 拉取/推送（docker resolver）
├── runtime/
│   └── v2/        ← Runtime v2 核心
│       ├── shim_manager.go  ← ShimManager：管理 shim 生命週期
│       ├── task_manager.go  ← TaskManager：RuntimePlugin 實作
│       ├── bundle.go        ← Bundle 目錄管理（rootfs + config.json）
│       └── shim.go          ← Shim 連線封裝
├── sandbox/       ← Sandbox Controller 介面 + store
├── snapshots/     ← Snapshotter 介面 + storage（bolt-backed meta）
├── streaming/     ← Streaming 介面
├── transfer/      ← Transfer 介面（image pull/push pipeline）
└── unpack/        ← Layer 解包器（並行解壓 + snapshot 操作）
```

**關鍵檔案：`core/metadata/db.go`**

`metadata.DB` 是整個 containerd 的狀態核心：
- 底層：`bbolt`（embedded KV，ACID）
- 功能：namespaced 存取所有 image / container / snapshot / sandbox / lease 資料
- GC：`mark & sweep`，清除無 lease 且無引用的 content / snapshot

---

### `plugins/` — 插件實作與註冊

**每個子套件透過 `init()` 呼叫 `registry.Register()`，在程式啟動時完成自動註冊。**

```
plugins/
├── types.go              ← 所有 plugin.Type 常數定義
│                           (ContentPlugin, SnapshotPlugin, GRPCPlugin, RuntimePluginV2...)
├── content/local/        ← 本地 Content Store 實作（檔案系統 + SHA256 addressing）
├── cri/                  ← Kubernetes CRI plugin（GRPCPlugin）
│   ├── cri.go            ← CRI plugin 主入口，整合 runtime + images
│   ├── images/           ← CRI image service
│   └── runtime/          ← CRI runtime service
├── diff/
│   ├── walking/          ← 預設 differ（walking 差異比對 + tar packing）
│   └── erofs/            ← erofs differ（唯讀壓縮格式）
├── events/               ← Events exchange plugin
├── gc/                   ← GC scheduler plugin
├── metadata/             ← Metadata bolt plugin（wraps core/metadata.DB）
├── nri/                  ← NRI（Node Resource Interface）plugin
├── restart/              ← Container restart monitor
├── sandbox/              ← Sandbox controller plugin
├── server/
│   ├── grpc/             ← gRPC server plugin（TLS, namespace middleware）
│   └── ttrpc/            ← ttRPC server plugin
├── services/             ← 所有 ServicePlugin 實作
│   ├── containers/       ← Container service（CRUD on metadata DB）
│   ├── content/          ← Content service
│   ├── diff/             ← Diff service
│   ├── events/           ← Events service
│   ├── images/           ← Images service
│   ├── leases/           ← Leases service
│   ├── namespaces/       ← Namespaces service
│   ├── sandbox/          ← Sandbox controller + store service
│   ├── snapshots/        ← Snapshots service
│   ├── tasks/            ← Tasks service（呼叫 RuntimePluginV2）
│   └── transfer/         ← Transfer service
├── snapshots/
│   ├── overlay/          ← overlayfs snapshotter（預設）
│   ├── btrfs/            ← btrfs snapshotter
│   ├── devmapper/        ← device mapper thin provisioning
│   ├── erofs/            ← erofs snapshotter（唯讀壓縮）
│   ├── native/           ← native（plain copy, no COW）
│   └── blockfile/        ← block file based snapshotter
└── transfer/             ← Transfer plugin（image pull/push 流水線）
```

**Plugin 依賴宣告方式（來自原始碼）：**

```go
// core/runtime/v2/shim_manager.go
registry.Register(&plugin.Registration{
    Type: plugins.ShimPlugin,          // 自己的 Type
    ID:   "manager",
    Requires: []plugin.Type{
        plugins.EventPlugin,           // 依賴 EventPlugin
        plugins.MetadataPlugin,        // 依賴 MetadataPlugin
    },
    InitFn: func(ic *plugin.InitContext) (any, error) {
        m, _ := ic.GetSingle(plugins.MetadataPlugin)
        ep, _ := ic.GetByID(plugins.EventPlugin, "exchange")
        // ...
    },
})
```

---

### `cmd/` — 可執行程式

```
cmd/
├── containerd/                   ← 主 daemon
│   ├── main.go                   ← 程式入口
│   ├── builtins/                 ← blank import 所有內建 plugin（觸發 init()）
│   │   ├── builtins.go           ← 平台無關的 plugin 載入
│   │   ├── builtins_linux.go     ← Linux 特有：overlay, btrfs, devmapper, erofs...
│   │   ├── builtins_windows.go   ← Windows 特有：hcsshim, cimfs...
│   │   └── cri.go                ← CRI plugin 載入
│   ├── command/
│   │   ├── main.go               ← CLI flags 解析（--config, --log-level...）
│   │   └── oci-hook.go           ← OCI hook 處理
│   └── server/
│       ├── server.go             ← 啟動 gRPC + ttRPC server，初始化 plugin registry
│       └── config/config.go      ← config.toml 解析
├── ctr/                          ← CLI 工具
│   ├── main.go
│   ├── app/main.go               ← urfave/cli app 定義
│   └── commands/                 ← 各子命令實作
│       ├── images/               ← ctr images pull/push/list...
│       ├── containers/           ← ctr containers create/delete...
│       ├── tasks/                ← ctr tasks start/kill/list...
│       ├── snapshots/            ← ctr snapshots list/info...
│       ├── content/              ← ctr content list/fetch...
│       ├── run/                  ← ctr run（shortcut: create + start）
│       └── sandboxes/            ← ctr sandboxes list...
├── containerd-shim-runc-v2/      ← Runtime v2 Shim
│   ├── main.go                   ← 入口：讀 BootstrapParams，啟動 ttRPC server
│   ├── manager/manager_linux.go  ← Shim 管理（1:1 or 1:many 判斷）
│   ├── process/
│   │   ├── init.go               ← 初始 container process（runc create + start）
│   │   ├── exec.go               ← exec process（runc exec）
│   │   └── io.go                 ← stdio 管理（fifo / pty）
│   ├── runc/container.go         ← runc 呼叫封裝
│   └── task/service.go           ← TaskService ttRPC 實作
└── containerd-stress/            ← 壓力測試工具
```

**`builtins_linux.go` 的 blank import 機制：**

```go
import (
    _ "github.com/containerd/containerd/v2/plugins/snapshots/overlay/plugin"   // 觸發 overlay plugin init()
    _ "github.com/containerd/containerd/v2/plugins/snapshots/btrfs/plugin"     // 觸發 btrfs plugin init()
    _ "github.com/containerd/containerd/v2/plugins/diff/walking/plugin"        // 觸發 differ init()
    _ "github.com/containerd/containerd/v2/core/metrics/cgroups"               // 觸發 cgroups init()
)
```

所有 plugin 的 `init()` 執行後，`registry` 內有完整的依賴圖，server 啟動時做 topological sort 依序初始化。

---

### `client/` — Go SDK

外部程式（ctr, integration tests, Docker, nerdctl）使用此 library 連接 containerd。

```
client/
├── client.go         ← Client struct（封裝 gRPC 連線 + 所有 stub）
├── container.go      ← Container 操作（NewTask, Spec, Delete）
├── task.go           ← Task 操作（Start, Wait, Kill, Delete）
├── image.go          ← Image 操作（Unpack, Name, Size）
├── pull.go           ← Pull 流程（RemoteContext → Transfer pipeline）
├── events.go         ← Event 訂閱
├── sandbox.go        ← Sandbox 操作
└── services.go       ← gRPC stub 初始化（ContainersClient, TasksClient...）
```

**`Client` 的核心作用**：將 gRPC stub 包裝成友善的 Go API，隱藏 Protobuf 序列化細節。

---

### `internal/` — 內部套件

```
internal/
├── cri/              ← Kubernetes CRI 完整實作
│   ├── server/       ← CRI service（RunPodSandbox, CreateContainer, StartContainer...）
│   │   ├── sandbox_run.go        ← RunPodSandbox 邏輯
│   │   ├── container_create.go   ← CreateContainer 邏輯
│   │   ├── container_start.go    ← StartContainer 邏輯
│   │   └── podsandbox/           ← pause container 實作（Sandbox Controller 之一）
│   ├── store/        ← CRI 層的 in-memory state（container, sandbox, image status）
│   ├── config/       ← CRI 配置解析
│   ├── io/           ← CRI container IO 管理
│   └── opts/         ← OCI spec 生成 options（cgroup, seccomp, apparmor...）
├── nri/              ← NRI（Node Resource Interface）內部實作
├── eventq/           ← event queue 實作
├── kmutex/           ← keyed mutex（per-container lock）
├── oom/              ← OOM 監控
├── fsview/           ← filesystem view（overlay + erofs）
└── userns/           ← user namespace ID mapping
```

---

### `pkg/` — 共用工具套件

```
pkg/
├── archive/          ← tar 操作（含壓縮、link 處理、時間戳）
├── cio/              ← Container IO（fifo, pty, logging）
├── gc/               ← GC tricolor mark algorithm
├── namespaces/       ← namespace context 工具（context.WithValue）
├── oci/              ← OCI spec 生成（WithHostNamespace, WithMounts, WithCapabilities...）
├── shim/             ← Shim bootstrap 協議（讀/寫 BootstrapParams stdin/stdout）
├── sys/              ← OS 底層（pidfd, namespace clone, socket, reaper）
├── tracing/          ← OpenTelemetry 整合
├── filters/          ← 通用 filter DSL（用於 list API）
├── dialer/           ← Unix socket dialer
├── progress/         ← 進度列顯示
├── timeout/          ← 可配置 timeout context
└── labels/           ← label 驗證
```

---

### `contrib/` — 周邊工具

```
contrib/
├── apparmor/     ← AppArmor profile 生成
├── seccomp/      ← seccomp 預設 profile
├── fuzz/         ← OSS-Fuzz 模糊測試
├── ansible/      ← Ansible playbook（k8s 部署）
├── checkpoint/   ← CRIU checkpoint/restore 測試腳本
├── diffservice/  ← 外部 diff service 範例
└── snapshotservice/ ← 外部 snapshot service 範例
```

---

## 模組串聯關係圖

### 靜態依賴關係

```
cmd/containerd/builtins/
    │ blank import → 觸發所有 plugin init()
    ▼
plugins/ (每個 plugin 呼叫 registry.Register)
    │
    ▼
registry (plugin.Set)
    │ topological sort by Requires
    ▼
cmd/containerd/server/server.go
    │ 依序呼叫 InitFn
    ▼
┌─────────────────────────────────────────────────────┐
│ Plugin 依賴鏈（由底層到上層）                         │
│                                                     │
│  ContentPlugin (local file store)                   │
│       ↓                                             │
│  SnapshotPlugin (overlay/btrfs/zfs...)              │
│       ↓ ↘                                          │
│  MetadataPlugin (bolt DB)  EventPlugin (exchange)   │
│       ↓ ↘         ↙                                │
│    ShimPlugin (ShimManager)                         │
│         ↓                                           │
│   RuntimePluginV2 (TaskManager)                     │
│         ↓                                           │
│   ServicePlugin (tasks/containers/images...)        │
│         ↓                                           │
│   GRPCPlugin (gRPC server + CRI)                    │
│   TTRPCPlugin (ttRPC server)                        │
└─────────────────────────────────────────────────────┘
```

### 執行期呼叫關係

```
外部 client (kubectl / docker / ctr)
    │ gRPC
    ▼
plugins/server/grpc  ← namespace middleware, TLS
    │
    ▼
plugins/services/tasks/service.go  (TasksService gRPC handler)
    │ 呼叫
    ▼
core/runtime/v2/task_manager.go  (TaskManager)
    │ 呼叫
    ▼
core/runtime/v2/shim_manager.go  (ShimManager)
    │ fork/exec
    ▼
cmd/containerd-shim-runc-v2/  (shim process, 獨立 PID)
    │ ttRPC TaskService
    ▼
cmd/containerd-shim-runc-v2/task/service.go
    │ fork/exec runc
    ▼
runc (OCI runtime engine)
    │ libcontainer → kernel namespaces/cgroups
    ▼
Container Process
```

### Image Pull 資料流

```
client/pull.go
    │ Transfer pipeline
    ▼
core/transfer/local/pull.go
    │
    ├─→ core/remotes/docker/fetcher.go  ← HTTPS to registry
    │         │ stream bytes
    │         ▼
    │   plugins/content/local/store.go  ← 寫入 SHA256-addressed 檔案
    │
    └─→ core/unpack/unpacker.go
              │ read from content store
              ▼
        plugins/snapshots/overlay/  ← Prepare → mount → unpack tar → Commit
              │
              ▼
        core/metadata/db.go  ← 記錄 image name → digest 對應
```

### Metadata DB 的核心地位

```
core/metadata/db.go (bbolt)
    │
    ├── namespaces bucket
    │     └── <ns>/containers/<id>  → container spec
    │     └── <ns>/images/<name>    → image digest
    │     └── <ns>/snapshots/<key>  → snapshot metadata
    │     └── <ns>/leases/<id>      → lease → 防 GC
    │     └── <ns>/sandboxes/<id>   → sandbox metadata
    │
    └── content bucket
          └── <digest>/info         → size, labels
                                    （實際內容在 content store 檔案系統）
```

---

## Plugin Type 完整清單（來自 `plugins/types.go`）

| Plugin Type 常數 | 值（字串） | 功能 |
|-----------------|-----------|------|
| `ContentPlugin` | `io.containerd.content.v1` | 內容存取（SHA256 addressed） |
| `SnapshotPlugin` | `io.containerd.snapshotter.v1` | 檔案系統快照後端 |
| `MetadataPlugin` | `io.containerd.metadata.v1` | bolt DB（所有 metadata） |
| `EventPlugin` | `io.containerd.event.v1` | 事件 exchange（pub/sub） |
| `ShimPlugin` | `io.containerd.shim.v1` | Shim 生命週期管理 |
| `RuntimePlugin` | `io.containerd.runtime.v1` | Runtime（deprecated） |
| `RuntimePluginV2` | `io.containerd.runtime.v2` | Runtime v2（TaskManager） |
| `ServicePlugin` | `io.containerd.service.v1` | 內部服務（tasks, images...） |
| `GRPCPlugin` | `io.containerd.grpc.v1` | 對外 gRPC handler |
| `TTRPCPlugin` | `io.containerd.ttrpc.v1` | ttRPC handler |
| `DiffPlugin` | `io.containerd.differ.v1` | Layer diff（apply/compare） |
| `GCPlugin` | `io.containerd.gc.v1` | GC policy |
| `LeasePlugin` | `io.containerd.lease.v1` | Lease manager |
| `NRIApiPlugin` | `io.containerd.nri.v1` | Node Resource Interface |
| `TransferPlugin` | `io.containerd.transfer.v1` | Transfer pipeline |
| `MetricsPlugin` | `io.containerd.metrics.v1` | Prometheus metrics |
| `TracingProcessorPlugin` | `io.containerd.tracing.processor.v1` | OpenTelemetry span |
| `SandboxStorePlugin` | `io.containerd.sandbox.store.v1` | Sandbox metadata store |
| `SandboxControllerPlugin` | `io.containerd.sandbox.controller.v1` | Sandbox controller |
| `StreamingPlugin` | `io.containerd.streaming.v1` | Streaming manager |
| `MountManagerPlugin` | `io.containerd.mount.manager.v1` | Mount 管理 |
| `TaskMonitorPlugin` | `io.containerd.monitor.task.v1` | Task 監控 |

---

## 平台差異處理

containerd 對多平台的處理策略：**同名介面，不同實作，透過 build tag 切換。**

```
plugins/snapshots/
├── overlay/   ← Linux only（overlayfs kernel feature）
├── btrfs/     ← Linux only（btrfs module）
├── windows/   ← Windows only（hcsshim）
└── native/    ← 所有平台（plain copy，無 COW）

cmd/containerd/builtins/
├── builtins_linux.go    ← import overlay, btrfs, erofs, cgroups
├── builtins_windows.go  ← import windows snapshotter, hcsshim
└── builtins.go          ← 平台無關（content, metadata, events...）
```

---

*文件生成時間：2025-05-01，基於 containerd v2.x main branch*
