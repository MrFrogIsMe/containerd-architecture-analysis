# 同學資料驗證報告

> 驗證對象：docs/gpt_res/mer.md + 架構解析containerd.pdf  
> 驗證方式：對照 containerd 原始碼（core/snapshots/, core/unpack/, core/containers/）

---

## 總評

**整體方向正確，適合當報告主軸。**  
三張圖的敘事邏輯清楚，Class → Activity → Sequence 的順序合理。  
但有 **1 個結構性錯誤** 需要修正，說明時要特別注意。

---

## 逐圖驗證

---

### Class Diagram ✅ 大致正確，一處需修正

**正確的部分：**

| 物件 | 驗證結果 |
|------|---------|
| Image → Layer (1對多) | ✅ 正確 |
| Layer 存在 ContentStore | ✅ 正確，Content Store 是 SHA256-addressed 的 blob 存放點 |
| Container 分離 Task 設計 | ✅ 這是最重要的設計點，完全正確 |
| Task 有 pid / status / exitCode | ✅ 對應原始碼 `api/runtime/task/v3/shim.proto` |
| RuntimeShim → Runtime → OCISpec | ✅ 正確，shim 呼叫 runc，runc 依 OCI spec 執行 |
| Container → OCISpec | ✅ 正確 |

**需要修正的地方：**

```
❌ 圖中寫的：Snapshotter --> Image : uses
✅ 正確的關係：Snapshotter 完全不認識 Image
```

原始碼驗證（`core/unpack/unpacker.go` 的 import）：
```go
import (
    "github.com/containerd/containerd/v2/core/content"     // 從 Content Store 讀 layer bytes
    "github.com/containerd/containerd/v2/core/snapshots"   // 呼叫 Snapshotter
    "github.com/opencontainers/image-spec/specs-go/v1"     // OCI Image spec
)
```

Snapshotter 的實際 API：
```go
// 正確的 signature（core/snapshots/snapshotter.go）
Prepare(ctx, key string, parent string) ([]mount.Mount, error)
// key = 這個快照的 ID（字串）
// parent = 上一層快照的 ID（字串）
// 完全沒有 Image 型別的參數
```

**真正的呼叫鏈是：**
```
Image（記錄 layer digest 列表）
  ↓
core/unpack/unpacker.go（中間人）
  ├─→ ContentStore.ReaderAt(layer digest)  ← 讀取 layer bytes
  └─→ Snapshotter.Prepare(key, parent)     ← 建立可寫快照
      Snapshotter.Commit(name, key)         ← 提交為唯讀快照
```

Snapshotter 刻意設計成不知道 Image 的存在，這是 containerd 解耦架構的核心之一。

**修正建議（說明時講這句話就夠了）：**
> Snapshotter 不直接使用 Image，而是由 unpacker 把 Content Store 裡的 layer bytes 解壓進去。Snapshotter 只懂 key 和 parent，不懂 image 是什麼。

---

### Sequence Diagram ✅ 正確，適合報告使用

整體流程描述準確：
- User → Docker/K8s → containerd → Image check → Snapshotter → Container → Task → Shim → runc → Kernel

**幾個說明時要注意的點（不是錯誤，但要知道）：**

1. `Image Service` 這個框代表的是整個 pull pipeline（包含 fetcher + content store + unpack），圖中簡化成一個方塊是 OK 的

2. Shim 和 containerd 之間的通訊是透過 **unix socket + ttRPC**（輕量版 gRPC），不是直接函式呼叫。這一點圖中沒有特別標示，但說明時可以補充

3. runc → Kernel 的三步驟（namespaces / cgroups / mounts）✅ 完全正確

---

### Activity Diagram ✅ 完全正確

流程圖的每一步都可以對應到原始碼：

| 步驟 | 對應位置 |
|------|---------|
| Pull image from registry | `core/remotes/docker/fetcher.go` |
| Store image layers in Content Store | `plugins/content/local/store.go` |
| Prepare snapshot / root filesystem | `plugins/snapshots/overlay/` |
| Create container metadata | `plugins/services/containers/local.go` |
| Create task | `plugins/services/tasks/local.go` |
| Start runtime shim | `core/runtime/v2/shim_manager.go` |
| Invoke runc with OCI spec | `cmd/containerd-shim-runc-v2/runc/container.go` |
| Setup namespaces / cgroups / mounts | runc → libcontainer → kernel syscalls |

---

## 是否足夠作為報告主軸？

**足夠，而且結構很好。**

這三張圖分別回答了三個不同問題：

```
Class Diagram     → 這個系統裡有哪些物件？它們的關係是什麼？
Activity Diagram  → 啟動一個容器，流程是什麼？
Sequence Diagram  → 流程中，誰跟誰說話？
```

剛好對應報告要求的「靜態結構（Class）」和「動態結構（Activity + Sequence）」。

---

## 3 分鐘說明建議

### 開場白（10 秒）
> 我負責的是架構解析，用三張圖說明 containerd 內部怎麼運作。

---

### Slide 1：Class Diagram（50 秒）

先指著整張圖說：
> containerd 把容器管理拆成這幾個物件，每個物件只負責一件事。

然後說三個重點，每個一句話：

**重點 1：ContentStore**
> Image 的 layer bytes 存在 ContentStore，用 SHA256 當 key，天然去重複，不同容器可以共用同一份 layer。

**重點 2：Container 和 Task 是分開的**
> Container 是靜態設定，就像一份食譜。Task 是真正跑起來的 process，有 PID。你可以先建 Container 但不啟動，需要的時候再建 Task。

**重點 3：Snapshotter（修正說明）**
> Snapshotter 負責準備 container 的 root filesystem，但它完全不認識 Image——它只看 key 和 parent。真正把 Image layer 解壓進去的，是中間的 unpacker。這樣的設計讓你可以換掉 storage 後端（overlayfs 換 btrfs），完全不影響 image 邏輯。

---

### Slide 2：Activity Diagram（40 秒）

> 這張圖是 containerd 接到啟動請求的完整流程。

指著分支點說：
> 先判斷 image 在不在，不在就從 registry 拉、存進 Content Store。

指著 Snapshot 那步：
> 然後用 Snapshotter 準備 root filesystem，這是 container 真正看到的檔案系統。

指著下半段說：
> 建好 container metadata 之後，建 Task，啟動 shim，shim 再呼叫 runc，runc 去設定 namespace、cgroup、mount，最後才啟動 container process。

> **重點：containerd 的工作是把這整條流程標準化，讓 Docker 和 Kubernetes 不用自己重複實作。**

---

### Slide 3：Sequence Diagram（40 秒）

> 同樣的流程，這張圖強調誰跟誰溝通。

從右邊往左指：
> 最底層是 Linux Kernel，runc 直接對它設定 namespace 和 cgroup。runc 上面是 Shim，Shim 是 containerd 和 runc 之間的橋。

指著 Shim：
> 這個 Shim 是一個獨立的 process，不是 containerd daemon 的一部分。好處是 containerd 重啟或升級時，Shim 繼續活著，container 不會死。

最後一句：
> 所以 containerd 的核心價值，就是當這個協調者——它自己不直接執行容器，而是協調 Image Service、Content Store、Snapshotter、Task Service、Shim、runc 完成整件事。

---

## 一句話總結（10 秒）

> containerd 的架構設計是「把複雜的事拆成簡單的介面，每個介面可以獨立替換」。這就是它能成為 Docker 和 Kubernetes 共同底層標準的原因。

---

## 需要改動的地方（給同學）

Class Diagram 的這條關係改一下：
```
❌ Snapshotter --> Image : uses
✅ 刪掉這條線，改成：
   Snapshotter : "只接受 key + parent（字串），不認識 Image"
   （在圖上加一個 note 或直接說明時補充即可）
```

其他圖不需要改，直接用。
