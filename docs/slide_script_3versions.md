# 架構解析三頁簡報逐字稿

> 對應簡報頁：第 10 頁（v1 架構圖）、第 11 頁（v2 架構圖）、第 12 頁（CRI 架構圖）

---

## 1 分鐘版（共約 60 秒）

> 節奏：快，每頁約 20 秒，重點一句帶過

**【第 10 頁 — Containerd v1 架構圖】**

containerd 的架構分三層。
最上層是對外介面，gRPC 是唯一入口，所有 client 都從這裡進來；Metrics 負責可觀測性。
中間是儲存層，Content 存 image bytes，Snapshot 建 rootfs，Metadata 存容器狀態，都在 bolt DB。
最下層是執行層，Runtimes 在 v1 裡就是直接呼叫 runc，還沒有 shim 的概念。

**【第 11 頁 — Containerd v2 架構圖】**

v2 最大的變化是 Logic Layer 多了 Sandbox API，把沙箱管理和映像傳輸獨立出來。
更重要的是 ShimManager，它在 Daemon 內部統一管理所有 shim 的生命週期。
底層的 containerd-shim-runc-v2，shim 是長駐 process，即使 daemon 重啟，容器仍繼續運行。
runc 只負責建立容器，完成就退出，stdio 和 exit code 由 shim 維持。

**【第 12 頁 — CRI 架構圖】**

這頁看 Kubernetes 怎麼接上來。
kubelet 不直接呼叫 containerd，而是透過 CRI 標準介面。
containerd 的 CRI Plugin 實作了 ImageService 和 RuntimeService，網路交給 CNI。
右邊是 Pod 視角：一個 Pod 對應一個 shim，sandbox container 佔住 namespace，
其他容器共用同一組 namespace 和 cgroup，這就是 Shim v2 一對多的實際應用。

---

## 2 分鐘版（共約 120 秒）

> 節奏：適中，每頁約 40 秒，補充設計原因

**【第 10 頁 — Containerd v1 架構圖】**

我們先看 v1 的架構，這是 containerd 的原始設計。

架構分三層。最上層是對外介面，gRPC 是所有 client 的唯一入口，不管是 Docker、Kubernetes 還是 ctr，都從同一個 API 進來。Metrics 提供 Prometheus 格式的可觀測性輸出。

中間是儲存層，分兩塊：Storage 負責實際資料，Content 用 SHA256 定址存 image bytes，Snapshot 管理 overlay 疊加層，建出容器的 rootfs。Metadata 這邊存的是容器的靜態狀態，包括 image 對照表和容器定義，全部存在 bolt DB。

最下層是執行層，Tasks 代表執行中的 process，Runtimes 在 v1 裡直接對接 runc。這是 v1 的主要問題：runtime 跟 daemon 耦合，daemon 掛掉容器就一起掛。

**【第 11 頁 — Containerd v2 架構圖】**

v2 針對 v1 的問題做了兩個關鍵改動。

第一，Logic Layer 新增了 Sandbox API，把沙箱管理、映像傳輸抽出來獨立處理，讓 Kubernetes 的 Pod 模型可以更自然地對應。Data Layer 也對應新增了 Sandboxes 和 Meta。

第二，也是最核心的改動：引入 ShimManager 和 containerd-shim-runc-v2。ShimManager 在 Daemon 內部統一管理所有 shim 的生命週期。每個容器對應一個獨立的 shim process，shim 跟 daemon 完全分離。

所以現在即使 daemon 重啟，容器仍然跑著，因為 shim 是獨立的 OS process。runc 只負責建立容器後就退出，是 short-lived 的，stdio 和 exit code 由 shim 長駐維持。

**【第 12 頁 — CRI 架構圖】**

最後看 Kubernetes 怎麼接上 containerd。

Kubernetes 的節點代理叫 kubelet，它不直接呼叫 containerd，而是透過一個標準介面叫 CRI。containerd 內建 CRI Plugin 實作這個介面，分兩塊：ImageService 負責 image pull，RuntimeService 負責容器生命週期。網路完全交給 CNI，containerd 本身不碰。

右邊這張圖是 Pod 視角。一個 Pod 裡有一個 sandbox container，就是 pause 容器，它的作用是佔住 namespace 和 cgroup，讓同 Pod 的容器共用網路和 IPC。每個 Pod 對應一個 shim，這個 shim 服務 Pod 內所有容器，大幅減少系統 process 數量。

下方小圖是最簡化的視角：kubelet 只看到 CRI，底下的 shim、runc、snapshot 全部被封裝，這就是分層解耦的核心價值。

---

## 3 分鐘版（共約 180 秒）

> 節奏：完整，每頁約 60 秒，補充設計動機、對比、舉例

**【第 10 頁 — Containerd v1 架構圖】**

我們先從 v1 的架構開始，理解 containerd 原始的設計思路。

containerd 被設計成「嵌入式 runtime」，不是給終端使用者直接操作的，而是給 Docker、Kubernetes 這類系統當底層用的。所以它的對外介面非常乾淨，只有 gRPC 這一個入口，不管是誰來呼叫，全部走同一套 API。Metrics 則是提供 Prometheus 格式的 metrics，讓外部監控系統可以觀察 daemon 狀態。

中間的儲存層是 containerd 設計上最有特色的地方。它把資料分成兩類：Storage 是不可變資料，Content 用 SHA256 做內容定址，存的是 image 的 bytes，所有 namespace 共享同一份，不會重複存。Snapshot 負責把 image 的各個 layer 解壓後疊起來，用 overlayfs 建出容器的 rootfs。Metadata 這邊存的是容器的靜態定義，包括 image 名稱、snapshot key、runtime 設定，全部用 bolt DB 存，重開機也不會消失。

執行層方面，Tasks 代表跑起來的 process，Events 是 pub/sub 事件系統。v1 的 Runtimes 直接對接 runc，架構比較簡單，但問題是 runtime 跟 daemon 生命週期綁在一起，daemon 一掛容器就全死，這是 v2 要解決的核心問題。

**【第 11 頁 — Containerd v2 架構圖】**

v2 的架構在 v1 基礎上做了兩個重要升級，而且都是為了解決真實的生產問題。

第一個升級是 Logic Layer 新增了 Sandbox API。v1 時代，Kubernetes 的 Pod 概念在 containerd 裡沒有直接對應的抽象，CRI Plugin 要自己處理很多 sandbox 邏輯。v2 把沙箱管理、映像傳輸、NRI 資源介面全部獨立成 Sandbox API，Data Layer 對應新增了 Sandboxes 和 Meta。這樣 Kubernetes 和 containerd 的邊界更清晰，擴充性也更好。

第二個升級是最核心的：Shim v2 架構。ShimManager 在 Daemon 內部統一管理所有 shim 的生命週期。當要建立一個容器時，ShimManager 根據 RuntimeInfo 的名稱找到對應的 shim binary，fork 出一個獨立的 OS process，這個 process 就是 ShimInstance。

ShimInstance 開一個 ttRPC server，之後所有 Task 操作，start、kill、wait，都是 containerd 透過 ttRPC 打到 shim，再由 shim 呼叫 runc。runc 執行完 create 就退出，是 short-lived 的，容器 process 本身由 shim 維持。

這個設計的關鍵好處是：shim 完全獨立於 daemon。daemon 重啟時，shim 還活著，容器還跑著，重啟完可以重新 attach，容器感受不到任何中斷。

**【第 12 頁 — CRI 架構圖】**

最後這頁，我們從 Kubernetes 的視角看 containerd 怎麼接進去。

Kubernetes 的節點代理叫 kubelet，負責管理這台機器上所有 Pod 的生命週期。它不直接呼叫 containerd，而是透過一個標準介面叫 CRI，Container Runtime Interface，這是 Google 在 2016 年定義的。只要實作 CRI，任何 runtime 都可以接上 Kubernetes，不一定要用 containerd，這就是標準介面的價值。

containerd 從 v1.1 開始內建 CRI Plugin，不需要額外跑 cri-containerd process。CRI Plugin 實作兩個介面：ImageService 負責 image 的 pull 和管理，RuntimeService 負責 Pod 和容器的建立、啟動、刪除。網路完全交給 CNI plugin，ocicni 負責呼叫 CNI，containerd 本身完全不碰網路。

右邊這張圖是 Pod 的內部視角。一個 Pod 裡有一個 sandbox container，也就是大家熟知的 pause 容器，它唯一的作用是佔住 Linux namespace 和 cgroup，讓同 Pod 的其他容器可以共用網路和 IPC。每個 Pod 對應一個 containerd shim，這個 shim 服務 Pod 內的所有容器，這正是 Shim v2 一對多設計的實際應用場景，大幅減少了系統的 process 數量。

最後下方的小圖是整個故事的結語：kubelet 只看到 CRI 這一層，底下的 ShimManager、shim binary、runc、snapshot，全部被封裝。這就是分散式系統設計裡「關注點分離」的具體實踐。

---

## 版本對照

| 版本 | 總時長 | 每頁時長 | 適用場景 |
|------|--------|----------|----------|
| 1 分鐘 | ~60 秒 | ~20 秒 | 快速帶過，配合 Q&A 留時間 |
| 2 分鐘 | ~120 秒 | ~40 秒 | 一般報告節奏，補充設計原因 |
| 3 分鐘 | ~180 秒 | ~60 秒 | 完整版，設計動機 + 對比 + 舉例 |
