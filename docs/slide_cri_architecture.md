# CRI 架構圖 — 簡報說明文字

> 對應簡報頁：架構解析 / CRI 架構圖

---

## 簡報上的重點文字

**CRI（Container Runtime Interface）**

- Kubernetes 定義的標準介面，kubelet 只認 CRI，不直接依賴 containerd
- containerd 內建 CRI Plugin，實作 `ImageService` 與 `RuntimeService` 兩個介面
- `ocicni` 負責網路（呼叫 CNI plugin），containerd 本身不管網路

**Pod 與 Shim 的對應關係**

- 一個 Pod = 一個 sandbox container（pause）+ 多個 app container
- 同一個 Pod 的 container 共用同一組 namespaces + cgroups（Pod A 區塊）
- 每個 Pod 對應一個 containerd shim，shim 跨容器共用（Shim v2 特性）

**簡化視角（下方小圖）**

```
kubelet → CRI → CRI Plugin → containerd → container × N
```
kubelet 只透過 CRI 溝通，以下的細節（shim、runc、snapshot）完全被封裝

---

## 口頭介紹方式

### 從上一頁銜接

> 上一頁我們看到 containerd 的整體架構，現在來看它怎麼跟 Kubernetes 接起來。

### 介紹這頁

> Kubernetes 的節點代理叫 kubelet，它不直接呼叫 containerd，而是透過一個標準介面叫 CRI。
>
> containerd 實作了這個介面，叫做 CRI Plugin。裡面有兩塊：
> 一個是 ImageService，負責 pull image；
> 一個是 RuntimeService，負責建立和管理容器。
>
> 網路的部分交給 ocicni，由 CNI plugin 處理，containerd 本身不碰網路。
>
> 右邊這張圖是 Pod 的視角。一個 Pod 裡有一個 sandbox container，就是那個 pause 容器，
> 它的作用是佔住 namespace 和 cgroup，讓同 Pod 的容器可以共用網路和 IPC。
> 每個 Pod 對應一個 shim，這就是我們之前說的 Shim v2 可以服務多個容器的實際應用。
>
> 下方小圖是簡化版，kubelet 只看到 CRI 這一層，下面的 shim、runc、snapshot 全部被封裝起來。
> 這就是分層解耦的好處：Kubernetes 不需要知道底層是 runc 還是其他 runtime。

### 銜接下一頁

> 接下來我們進一步看 containerd 的核心資料模型，也就是 Class Diagram。

---

## 與其他內容的關聯

| 關聯頁面 | 說明 |
|----------|------|
| Component Diagram（前頁） | CRI Plugin 就是 component diagram 裡 Logic Layer 的灰色區塊，這頁展示它的 Kubernetes 視角 |
| Class Diagram（後頁） | CRI Plugin 內部的 Tasks / Events / sandbox 對應到 class diagram 的 Task、ShimInstance、Container |
| Sequence Diagram（後頁） | CRI RunPodSandbox 是 `ctr run` 的 Kubernetes 版本，流程類似但多了 sandbox 建立步驟 |

---

## 關鍵術語速查（口頭備用）

| 術語 | 一句說明 |
|------|----------|
| CRI | kubelet 與 runtime 之間的標準 gRPC 介面，Google 2016 年定義 |
| CRI Plugin | containerd 內建的 CRI 實作，v1.1+ 內建，不需額外 cri-containerd process |
| pause container | Pod 的 namespace 佔位容器，不跑任何應用，只維持 network/IPC namespace |
| ocicni | 呼叫 CNI plugin 的橋接層，負責設定 Pod 網路（veth、bridge、IP） |
| Shim v2 一對多 | 同一個 Pod 的所有 container 共用一個 shim process，減少系統開銷 |
