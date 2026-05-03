# containerd 架構概論 — 第一張簡報

> 對應圖片：docs/historical/design/architecture.png

---

## 架構圖快速導覽

```
┌─────────────────────────────┬─────────────────┐
│           GRPC              │     Metrics      │
├─────────────────────────────┴─────────────────┤
│  ┌─────────────────────────┐  ┌─────────────┐ │
│  │  Storage                │  │  Metadata   │ │
│  │ ┌───────┬──────┬──────┐ │  │ ┌────┬────┐ │ │
│  │ │Content│Snap- │ Diff │ │  │ │Img │Cont│ │ │
│  │ │       │shot  │      │ │  │ │    │    │ │ │
│  │ └───────┴──────┴──────┘ │  └─┴────┴────┘ │ │
│  │           [DB]          │       [DB]      │ │
│  └─────────────────────────┘                 │ │
│                                Tasks  Events  │ │
│ ┌─────────────────────────────┐    │    ▲    │ │
│ │ OS                          │    ▼    │    │ │
│ └─────────────────────────────┘  Runtimes    │ │
└──────────────────────────────────────────────┘
```

---

## 各元件說明

### 最上層：對外介面

| 元件 | 作用 |
|------|------|
| **GRPC** | 唯一入口。Docker、Kubernetes、ctr CLI 統一透過 gRPC API 進入 containerd |
| **Metrics** | 可觀測性出口。輸出 Prometheus 格式監控指標，與業務邏輯完全分離 |

---

### 中間層：核心服務

#### Storage 群組（左側，有 DB）

| 元件 | 作用 | 來源 |
|------|------|------|
| **Content** | 以 SHA256 存放所有不可變 bytes（image layer、manifest、config）。相同內容只存一份，所有容器共用 | `plugins/content/local/` |
| **Snapshot** | 管理容器的 root filesystem。Image 層是唯讀 committed snapshot，容器啟動加一層可寫 active snapshot（overlayfs） | `plugins/snapshots/overlay/` |
| **Diff** | 計算兩個 snapshot 的差異，負責把 image layer tar 解壓進 snapshot。是 Content → Snapshot 的橋樑 | `plugins/diff/walking/` |

#### Metadata 群組（中間，有 DB）

| 元件 | 作用 | 來源 |
|------|------|------|
| **Images** | 記錄 image 名稱對應的 manifest digest。只是 name → digest 的對照表，實際 bytes 在 Content | `core/images/` |
| **Containers** | 記錄容器靜態定義（id、image、snapshot key、runtime）。**不是執行中的程序**，只是設定資料 | `core/containers/` |

#### 右側：執行層

| 元件 | 作用 |
|------|------|
| **Tasks** | 代表真正執行中的 container process。由 Container 建立，透過 ShimManager fork shim，shim 再呼叫 runc |
| **Events** | Pub/Sub 事件匯流排。容器啟動、停止、OOM 等狀態變化發布到此，上層系統訂閱後得知狀態 |

---

### 最下層

| 元件 | 作用 |
|------|------|
| **Runtimes** | 實際執行容器的 shim + runc 組合。疊加方塊表示可同時存在多個實例 |
| **OS** | Linux kernel。runc 透過 kernel API（clone/execve/cgroups/mount）建立隔離環境。虛線框表示 containerd 邊界之外 |

---

### 箭頭意義

```
Tasks  ──實線──▶  Runtimes    Tasks 主動呼叫 Runtime 啟動容器（同步）
Runtimes ─虛線──▶  Events     Runtime 非同步回報事件（容器啟動/退出）
```

對應分散式系統概念：
- **直接通訊**（Direct / RPC）：Tasks → Runtimes
- **間接通訊**（Indirect / Pub-Sub）：Runtimes → Events

---

## 簡報放置文字

```
對外介面
  • GRPC   — 唯一入口，Docker / Kubernetes / ctr 統一由此進入
  • Metrics — 可觀測性出口，Prometheus metrics

Storage（不可變資料）
  • Content  — SHA256 addressed，image bytes，跨容器共享
  • Snapshot — Container rootfs，overlay 疊加層
  • Diff     — 解壓 image layer 進 snapshot

Metadata（狀態資料，存 bolt DB）
  • Images     — name → digest 對照表
  • Containers — 容器靜態定義，不是執行中的程序

執行層
  • Tasks    — 執行中的 container process
  • Events   — Pub/Sub，非同步狀態通知
  • Runtimes — shim + runc，建立 namespace/cgroup/rootfs
```

---

## 一句話總結

> containerd 把「管理」和「執行」嚴格分開。
> Storage 和 Metadata 負責管理狀態，Tasks 和 Runtimes 負責執行，
> Events 負責通知。GRPC 是唯一對外窗口，每一層只做自己的事。

---

## ELI5：Content / Snapshot / Diff / Images 是什麼

---

### Content — 倉庫

**比喻：SHA256 智慧倉庫**

想像一個倉庫，規則只有一條：**相同的東西只存一份，每樣東西用內容的指紋（SHA256）來識別**。

你說「幫我拿 `sha256:abc123` 這個東西」，倉庫就給你那個 bytes。

- 100 個容器都用 alpine:latest → 倉庫只存一份 alpine 的 layer
- 刪掉一個容器 → 倉庫的 bytes 還在，其他容器不受影響
- 不認識「image」或「container」，只認識 digest 和 bytes

```
存入：Pull 時把 registry 的 bytes 寫進來
取出：啟動容器時讀出 layer bytes
位置：/var/lib/containerd/.../blobs/sha256/<digest>
```

---

### Snapshot — 備好的食材盤

**比喻：備料台**

廚師（container）開始工作前，需要把所有食材（filesystem）備好放在盤子上。

Image 有 3 層（layer 0、1、2），就像三層透明片疊在一起：
```
layer 0（最底，唯讀）← 來自 Content Store 解壓
layer 1（中間，唯讀）← 疊在 layer 0 上
layer 2（最上，唯讀）← 疊在 layer 1 上
──────────────────────
可寫層（container A 專屬）← 只有這層可以寫東西
```

容器 A 和容器 B 可以共用同樣的三層底層，各自有自己的可寫層。這就是 **overlayfs（疊加檔案系統）**。

- **Committed snapshot**：唯讀，可以當 parent（image 的每一層）
- **Active snapshot**：可寫，container 執行時使用
- Snapshot **不知道 Image 是什麼**，只知道 key 和 parent

---

### Diff — 搬運工

**比喻：拆包裹的工人**

Content Store 裡存的 image layer 是壓縮的 tar 包（`.tar.gz`）。這些 tar 包不能直接拿來用，要解壓到 Snapshot 裡才能當成 filesystem。

Diff 做的事：
```
1. 從 Content Store 讀出 layer tar bytes
2. 解壓縮
3. 把內容寫進 Snapshot 的可寫層
4. Snapshot.Commit() → 變成唯讀
5. 下一層的 parent = 這層
```

Pull 一個有 3 層的 image：Diff 要跑 3 次，每次解壓一層、提交成 committed snapshot，形成一條 parent 鏈。

---

### Images — 書的目錄頁

**比喻：圖書館的索引卡**

索引卡只有兩個欄位：
- **書名**：`docker.io/library/alpine:latest`
- **架位**：`sha256:abc123`（manifest 的 digest）

Images 不存任何 bytes，只記錄「這個名字對應到 Content Store 裡的哪個 manifest」。

```
Images 說：alpine:latest 的 manifest 在 sha256:abc123
         ↓
Content Store 說：sha256:abc123 這個 manifest 裡有 3 個 layer
         ↓
每個 layer 也在 Content Store（sha256:def456、sha256:ghi789...）
```

**你刪掉 Image（名字）**：Content Store 的 bytes 還在，只是沒有名字了。
**你刪掉 Content**：Image 的名字還在，但內容不見了，無法啟動新容器。

---

### 四者的完整協作流程

```
ctr pull alpine:latest

1. Images 記錄：alpine:latest → sha256:manifest
2. Content 存入：manifest bytes（sha256:manifest）
3. Content 存入：layer 0 bytes（sha256:layer0）
4. Content 存入：layer 1 bytes（sha256:layer1）
5. Diff 解壓 layer 0 → Snapshot.Prepare → Snapshot.Commit（parent=""）
6. Diff 解壓 layer 1 → Snapshot.Prepare → Snapshot.Commit（parent=layer0）

ctr run alpine:latest mycontainer sh

7. Snapshot.Prepare（key="mycontainer", parent=layer1）→ active snapshot（可寫層）
8. 把 active snapshot 的 mount 資訊傳給 shim/runc
9. runc 掛載 overlayfs，啟動 sh
```
