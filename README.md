# containerd 架構解析 — 期中報告輔助材料

> NCCU CS 分散式系統 期中報告  
> 學生：廖俊峰  
> 主題：containerd 架構解析

---

## 專案簡介

本 repo 記錄了以 AI Agent（Claude / Hermes CLI）為主要工具，對 containerd 原始碼進行架構解析，
並產出 UML 圖、架構說明文件的完整過程。

---

## 目錄結構

```
mid_pres/
├── docs/
│   ├── containerd_architecture_analysis.md   # 第一份架構分析報告（含 component diagram）
│   ├── containerd_arch_report.md             # 第二份報告（靜態 + 動態 + 設計問題分析）
│   ├── containerd_arch_analysis_report.md    # 第三份報告（更深度原始碼掃描版）
│   ├── containerd_directory_analysis.md      # containerd 原始碼目錄結構分析
│   ├── architecture_overview_slide.md        # 架構概覽簡報用文字
│   ├── modern_architecture_diagrams.md       # 架構圖（Component 層）
│   ├── class_diagram_explanation.md          # Class Diagram 元件說明
│   ├── sequence_diagram_ctr_run.md           # ctr run 流程 Sequence Diagram（分三段）
│   ├── 架構解析_簡報稿_3min.md               # 3 分鐘口頭報告稿
│   ├── 分散式系統期中報告.pdf                 # 最終繳交 PDF
│   ├── gpt_res/
│   │   ├── class_diagram_versions.md         # Class Diagram V0~V4 演進紀錄（含每版修改原因）
│   │   ├── mermaid_history.md                # Mermaid 圖歷史版本
│   │   ├── mermaid_corrected.md              # 修正版 Mermaid
│   │   ├── mer.md                            # 同學原版 class diagram（作為基準）
│   │   └── 架構解析containerd.pdf            # gpt 初版分析 PDF
│   └── teacher_requirment/
│       ├── course introduction.pdf           # 課程說明
│       └── remoting.pdf                      # 課程參考資料
└── containerd/                               # containerd 官方原始碼（供閱讀分析用）
    └── docs/                                 # 官方文件（runtime-v2.md, sandbox-api.md 等）
```

---

## AI Agent 互動說明

### 使用工具

| 工具 | 用途 |
|------|------|
| **Hermes CLI + Claude (claude-sonnet-4-6)** | 主要 AI Agent，架構分析、圖表設計、文字產出 |
| **nano banana（AI 繪圖工具）** | 根據 prompt 繪製 UML 圖（手繪風格 class diagram / architecture diagram） |
| **Mermaid** | sequence diagram、class diagram 語法，在 renderer 中預覽 |

### 互動流程

整個分析過程共分為以下幾個階段，均透過對話式指令與 Agent 協作完成：

#### 階段一：架構初探

1. 將 containerd 官方原始碼下載至本機（`containerd/` 目錄）
2. 請 Agent 掃描 `docs/`、`plugins/`、`cmd/`、`internal/` 等目錄，產出第一份架構分析報告
3. Agent 閱讀 `runtime-v2.md`、`sandbox-api.md` 等官方設計文件，提取官方 sequence diagram

#### 階段二：Component Diagram 設計

1. 依分析報告，請 nano banana 繪製 containerd 整體架構圖
2. 多輪審查：逐一確認層級（External Clients → Daemon → Shim → OS）、元件分類、箭頭方向
3. 主要修正點：
   - ShimManager 位置從 Daemon 框外移入框內
   - ShimManager → containerd-shim-runc-v2 補上箭頭
   - SHIM / RUNC 父子關係（從並列改為包含）

#### 階段三：Class Diagram 設計與版本演進

與 Agent 共同設計 containerd 核心資料模型的 Class Diagram，歷經 V0（同學原版）→ V4（最終版）共五個版本。

| 版本 | 主要修正 |
|------|----------|
| V0 | 同學原版，Snapshotter → Image 錯誤，Runtime 虛構 class |
| V1 | 修正關聯錯誤，加入 Descriptor、Kind，但 note 太多版面爆炸 |
| V2 | 去除 note，但 Task 層級被 renderer 拉錯 |
| V3 | 移除 Task→ShimInstance 實線，確認層級正確 |
| V4 | 加回 Snapshotter 虛線、Task ttRPC 虛線，最終確認版 |

V4 之後，根據審查補上 `Process` 類別（Task exec 子行程），並調整版面避免線交叉。

詳細演進紀錄：`docs/gpt_res/class_diagram_versions.md`

#### 階段四：Sequence Diagram 設計

以 `ctr run` 完整流程為主題，繪製三段式 sequence diagram：

| 圖 | 範圍 | 步驟 |
|----|------|------|
| 圖一：準備階段 | Image 確認 → Snapshot 建立 → Container 寫入 | 1~3 |
| 圖二：執行階段 | ShimManager fork → ttRPC → runc start | 4~5 |
| 圖三：清理階段 | Wait blocking → Delete → Shutdown → Snapshot 移除 | 6~7 |

詳細 Mermaid 與說明文字：`docs/sequence_diagram_ctr_run.md`

---

### 典型對話模式

**審查式互動**：每次 nano banana 生圖後，截圖傳給 Claude 審查，Claude 指出具體錯誤（欄位錯誤、關聯方向、版面問題），再產出精確 prompt 讓 nano banana 修正。

```
截圖 → Claude 審查 → 列出問題 → 產出修正 prompt → nano banana 重繪 → 截圖 → ...
```

**累積式設計**：每次修正只做最小改動，保留已確認正確的部分，避免全部重畫。

**分段驗證**：大圖（如 ctr run 全流程）先確認邏輯正確，再拆成小張以利簡報呈現。

---

## 主要產出文件

| 文件 | 說明 |
|------|------|
| `docs/containerd_arch_report.md` | 最完整的架構解析報告，含 component、class、sequence、activity diagram |
| `docs/gpt_res/class_diagram_versions.md` | Class Diagram 五版演進，含每版錯誤分析與修正決策 |
| `docs/sequence_diagram_ctr_run.md` | ctr run 三段式 sequence diagram + 簡報說明文字 |
| `docs/架構解析_簡報稿_3min.md` | 3 分鐘口頭報告逐字稿 |

---

## 關鍵技術發現

1. **Container ≠ 執行中的程序**：Container 是 bolt DB 的一筆 metadata，Task 才是執行期物件
2. **Snapshotter 不認識 Image**：純介面，只操作字串 key，完全解耦
3. **ShimInstance 獨立於 daemon**：daemon crash 後容器繼續跑，重啟可 re-attach
4. **runc 是 short-lived**：create 完就退出，namespaces/cgroups 由 OS 維持，shim 負責 stdio/exit
5. **ttRPC 而非 gRPC**：containerd ↔ shim 使用 ttRPC（tiny ttrpc），比 gRPC 更輕量，走 unix socket
