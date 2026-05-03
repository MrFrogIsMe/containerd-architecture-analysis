# ctr run Sequence Diagram — 分段版

> 完整流程分三段：準備、執行、清理
> 對應 class diagram 的動態視角

---

## 整體流程概覽

```
[圖一] 準備階段  →  [圖二] 執行階段  →  [圖三] 清理階段
  步驟 1~3              步驟 4~5              步驟 6~7
  Image / Snapshot      ShimManager / runc    Wait / Delete
  / Container 建立       Task 啟動             資源回收
```

---

## 圖一：準備階段（步驟 1~3）

> Image 確認 → Snapshot 建立 rootfs → Container metadata 寫入

```mermaid
sequenceDiagram
    actor User
    participant ctr
    participant containerd as containerd Daemon
    participant MetaDB as bbolt MetaDB
    participant ContentStore
    participant Snapshotter

    User->>ctr: ctr run <image> <id> <cmd>

    Note over ctr,MetaDB: 步驟 1：確認 Image
    ctr->>containerd: ImageService.Get(image ref)
    containerd->>MetaDB: 查詢 Image metadata
    MetaDB-->>containerd: Image{name, Descriptor{digest}}
    containerd-->>ctr: Image

    Note over ctr,Snapshotter: 步驟 2：準備 Snapshot（rootfs）
    ctr->>containerd: SnapshotService.Prepare(snapshotKey, parent)
    containerd->>Snapshotter: Prepare(key, parent)
    Snapshotter->>ContentStore: ReaderAt(digest) — 逐 layer 讀取
    ContentStore-->>Snapshotter: layer blob
    Snapshotter-->>containerd: Mount[]（overlayfs 掛載點）
    containerd-->>ctr: snapshotKey

    Note over ctr,MetaDB: 步驟 3：建立 Container（純 metadata）
    ctr->>containerd: ContainerService.Create(id, image, snapshotKey, runtime, OCISpec)
    containerd->>MetaDB: 寫入 Container record
    MetaDB-->>containerd: ok
    containerd-->>ctr: Container（此時無任何 process）
```

### 簡報說明文字

**準備階段做了什麼？**

- Image 不是「檔案」，是一筆 metadata，真正的 blob 在 ContentStore 裡，用 SHA256 digest 當 key
- Snapshotter 把每一層 layer 解開，用 overlayfs 疊成一個 rootfs，回傳 mount 路徑
- Container 建完只是 bolt DB 裡的一筆記錄，沒有任何 process 存在
- 這三步都是「靜態準備」，還沒有任何東西跑起來

---

## 圖二：執行階段（步驟 4~5）

> ShimManager fork shim binary → shim 啟動 ttRPC → runc 建立並啟動容器

```mermaid
sequenceDiagram
    participant ctr
    participant containerd as containerd Daemon
    participant ShimManager
    participant Shim as containerd-shim-runc-v2
    participant runc

    Note over ctr,ShimManager: 步驟 4：建立 Task（fork shim）
    ctr->>containerd: TaskService.Create(containerID, rootfs Mount[])
    containerd->>ShimManager: start(id, runtimeName)
    ShimManager->>Shim: exec binary（containerd-shim-runc-v2 start）
    Shim->>Shim: 啟動 ttRPC server
    Shim-->>ShimManager: socket address（unix://...sock）
    ShimManager-->>containerd: ShimInstance{socketAddr, pid}

    containerd->>Shim: TaskService.CreateTaskRequest（OCI spec + rootfs）
    Shim->>runc: runc create（OCI bundle）
    runc-->>Shim: container created（namespaces + cgroups 建立完成）
    Shim-->>containerd: Task{pid}
    containerd-->>ctr: Task

    Note over ctr,runc: 步驟 5：啟動 Task
    ctr->>containerd: TaskService.Start(containerID)
    containerd->>Shim: TaskService.StartRequest（ttRPC）
    Shim->>runc: runc start
    runc-->>Shim: process running（pid 存活）
    Shim-->>containerd: started
    containerd-->>ctr: pid
```

### 簡報說明文字

**執行階段的關鍵設計：三層隔離**

- **ShimManager** 透過 PATH 找到 shim binary（`io.containerd.runc.v2` → `containerd-shim-runc-v2`），fork 出獨立 OS process
- **Shim** 是 containerd 與 runc 之間的橋接層，啟動後開 ttRPC server，等 containerd 來連
- **runc** 執行完 `create` 就退出（short-lived），namespaces / cgroups 已建立，container process 由 shim 維持
- 所有 Task 方法（start / kill / wait）都是 containerd → shim 的 ttRPC call，不是直接操作 runc

---

## 圖三：清理階段（步驟 6~7）

> Wait 等待結束 → 反向清理 Task、Shim、Snapshot

```mermaid
sequenceDiagram
    participant ctr
    participant containerd as containerd Daemon
    participant Shim as containerd-shim-runc-v2
    participant ShimManager
    participant runc
    participant Snapshotter

    Note over ctr,Shim: 步驟 6：等待 Task 結束
    ctr->>containerd: TaskService.Wait(containerID)
    containerd->>Shim: TaskService.WaitRequest（ttRPC，blocking）
    Note over Shim: container process 執行中...
    Shim-->>containerd: ExitStatus{code, exitedAt}
    containerd-->>ctr: exit code

    Note over ctr,Snapshotter: 步驟 7：清理資源
    ctr->>containerd: TaskService.Delete(containerID)
    containerd->>Shim: TaskService.DeleteRequest
    Shim->>runc: runc delete（清除 cgroups / namespaces）
    runc-->>Shim: ok
    Shim-->>containerd: DeleteResponse

    containerd->>Shim: TaskService.ShutdownRequest
    Shim-->>containerd: ok
    ShimManager->>Shim: exec binary（containerd-shim-runc-v2 delete）

    ctr->>containerd: SnapshotService.Remove(snapshotKey)
    containerd->>Snapshotter: Remove(key)
    Snapshotter-->>containerd: ok（overlayfs 卸載，layer 釋放）
    containerd-->>ctr: done
```

### 簡報說明文字

**清理階段：反向拆解**

- `Wait` 是 blocking call，containerd 透過 ttRPC 等 shim 回傳 exit event，不是 polling
- `Delete` 順序嚴格：先刪 Task（runc delete 清 cgroups）→ Shutdown shim ttRPC → shim binary delete → 移除 Snapshot
- Snapshot 最後才移除，確保 runc 清理完成前 rootfs 還掛著
- 若 daemon 在 Wait 期間 crash，container 仍繼續跑（ShimInstance 獨立於 daemon），重啟後可 re-attach

---

## 版本紀錄

| 版本 | 說明 |
|------|------|
| V1 | 完整單張版（字太小） |
| V2（本檔） | 分三段版，準備 / 執行 / 清理 |
