# Windows 支援可行性評估與規格

狀態：提案，尚未實作或驗證。評估日期：2026-10-12。

## 1. 目標與結論

`openab-pty` 的目的，是讓使用者透過 WebSocket 操作與代理程式共用工作區的互動式終端。Windows 有三種不同需求，不能以單一「支援 Windows」涵蓋：

| 路線 | 使用者取得的環境 | 可行性 | 本倉庫現況 |
| --- | --- | --- | --- |
| A. Windows 主機上的 WSL2／Linux 容器 | Linux shell 與現有 Linux 代理環境 | 高；主要是部署驗證 | 未提供 WSL2 文件或 CI；Linux 映像已存在 |
| B. Windows 本機瀏覽器工具反向連入 Linux 工作階段 | Linux shell 加上經明確授權的 Windows 瀏覽器能力 | 中高；服務端協定已存在，Windows 工具提供者待實作 | `tools/attach` 已存在，文件和實作以 Mac 為例 |
| C. 原生 Windows 服務端 | Windows 工作區、PowerShell／`cmd.exe` 與 Windows 代理 CLI | 技術上可行，正式安全支援的成本高 | 未建置、未測試、未部署；現有 Linux 安全與清理機制不能直接沿用 |

**建議順序：A → B → C。** A 與 B 即可滿足「在 Windows 使用」及「操作 Windows 本機瀏覽器」的大部分需求；當代理與工作區本身必須位於 Windows 時，才需要 C。三條路線各自發布，不能以 A 或 B 的成功宣稱 C 已受支援。

首輪驗證目標為 **x86_64 Windows 11 + WSL2**（A、B），以及具備 Hyper-V 隔離能力的 x86_64 Windows 環境（C）。這是測試範圍，不是宣稱其他 Windows 版本必然不相容。Windows on ARM、Windows Server 與 WSL1 均不納入首輪正式支援；ARM64 Linux nightly 映像的存在不等於 Windows on ARM 已驗證。Windows 10 的 ConPTY／WSL2 最低版本以第 7 節的 Microsoft 文件為準，需在後續測試矩陣中逐一納入。

## 2. 現有功能基線

以下是 Windows 支援必須保留或提供等效行為的基線，而非只求能開啟一個 shell：

| 基線 | 現有來源 | A | B | C |
| --- | --- | --- | --- | --- |
| PTY 輸入、輸出、resize、互動式程式 | `runtime/src/session.rs` | 原樣 | 原樣 | 改以 ConPTY 驗證 |
| 終端能力查詢與回覆過濾、有限佇列與缺口通知 | `runtime/src/capproxy.rs`、`termfilter.rs`、`ringbuf.rs` | 原樣 | 原樣 | 驗證 Windows CLI 的 VT 輸出與現有客戶端相容 |
| 管理 API、命名工作階段、restart、renew、delete | `runtime/src/server.rs`、`CLIENT-CONTRACT.md` | 原樣 | 原樣 | 協定相同，需端對端測試 |
| 單一 attach、接管、scrollback offset、斷線續接、close codes | `runtime/src/session.rs`、`CLIENT-CONTRACT.md` | 原樣 | 原樣 | 協定相同，需端對端測試 |
| Admin verifier、工作階段 token、TTL、撤銷、限流 | `runtime/src/admin_auth.rs`、`token.rs`、`server.rs` | 原樣 | 原樣 | 演算法可沿用；機密隔離需重做 |
| 工作階段環境清理、播種、startup hook | `session.rs`、`seed.rs`、`hook.rs` | 原樣 | 原樣 | Windows 路徑、大小寫、連結與腳本需適配 |
| 行程清理與存活者稽核 | `runtime/src/killdomain.rs` | 原樣 | 原樣 | Windows 實作必需，不能使用現有非 Unix 空實作 |
| 反向工具接入、單工作階段授權、到期與撤銷 | `runtime/src/tools.rs`、`CLIENT-CONTRACT.md` §9 | 原樣 | Windows 工具提供者須符合協定 | 須測試原生部署的 loopback 與工具連線 |
| Shell 無主機憑證、唯讀根檔案系統、暫存工作區 | `SECURITY.md`、部署 manifest | Linux 容器仍須通過驗證 | Linux 容器仍須通過驗證；瀏覽器是明確授權能力 | 須建立可驗證的 Windows 對等隔離 |

「功能對等」指上述可觀察行為及安全結果，不要求 Windows 使用 Linux 的 syscall。Linux 版既有 `teardown_best_effort` 與 `absolute_ttl_best_effort` 標示也必須誠實保留或依平台能力修訂，不能把未驗證的 Windows 終止行為標為保證。

## 3. 可行性證據與缺口

### A. WSL2／Linux 容器

WSL2 使用真正的 Linux 核心；本倉庫發行 Linux musl 執行檔，正式映像流程使用 `linux/amd64`。因此在 x86_64 WSL2 中運行 Linux 容器有合理技術基礎。本次評估所在機器的 `wsl -e uname -a` 回傳 WSL2 Linux x86_64；這只證明本機 WSL2 可用，**沒有證明本映像已在 WSL2 啟動或安全通過**。

直接把執行檔放進一般 WSL2 發行版，只能算功能實驗。正式路線須使用符合現有沙盒設定的 Linux 容器，並驗證唯讀根檔案系統、非 root、能力移除、無主機或服務帳戶憑證、暫存工作區及 tailnet 可達性。工作區預設留在容器／Linux 檔案系統；掛載 Windows 主機目錄會改變「無主機存取」的安全聲明，須另外設計和驗證。WSL1 不列為支援目標。

### B. Windows 瀏覽器工具提供者

現有 `tools/attach/{session}` 是由工具提供者主動撥入的 MCP 通道：管理者核發單工作階段秘密，服務端只保留 verifier，工作階段內的 CLI 使用 loopback MCP URL。協定本身不依賴 Mac；「Mac」是現有實作和文件的部署對象。Windows 提供者可依 `CLIENT-CONTRACT.md` §9 實作 `initialize`、`tools/list`、`tools/call`、心跳、重撥與撤銷處理。服務端現有工具轉送不會替提供者限制瀏覽器權限，因此**瀏覽器設定檔、可呼叫工具與操作範圍必須由 Windows 提供者強制執行**。

提供者預設使用獨立的瀏覽器設定檔與明確的工作階段授權。使用既有登入設定檔是另一種高權限模式，須有可見授權、撤銷及審計；不能因為 PTY token 有效就自動取得 Windows 瀏覽器。反向接入屬於對工作階段顯式授予的主機能力，不應被表述為沙盒內原本就有的權限。`kiro-cli` 的 MCP 設定有現成 startup hook；其他代理變體須逐一確認如何載入 URL，不能只看環境變數存在。

Windows 提供者程式可能位於 `openabdev/instance-mcp` 或另一個客戶端倉庫；本倉庫需要維護跨平台協定、測試和部署文件，但單靠本倉庫不能交付提供者二進位檔。正式發布前須確認其所有權、發行方式與版本相容性。

### C. 原生 Windows 服務端

`portable-pty` 0.9 在 Windows 使用 ConPTY，故 PTY 本身並非主要障礙。但目前 `PortablePtySpawner` 於 spawn 後假設子行程執行 `setsid`，並以 `pid == pgid` 登記清理領域；Windows 不成立。`killdomain.rs` 的非 Unix `SystemSignals` 回傳 `Unsupported`，`hook.rs` 的非 Unix 行程群組清理是空實作。`containment.rs` 在非 Linux 上回報 `Unsupported`，而啟動流程只記警告；這不能作為 Windows 正式安全支援。預設 `/bin/bash`、`/workspace`、`HOME`、Unix 檔案權限與 symlink 行為，以及以 Linux `openab` 為基底的 Dockerfile 都需調整。

原生 Windows 版的建議隔離單位是**專用 VM 或 Hyper-V 隔離的 Windows 容器**，內含代理、PTY runtime 與暫存工作區；runtime 與 shell 還需不同權限身分及適當 ACL，並在讀取管理 verifier 前建立隔離。單靠 ConPTY、Job Object 或「非管理員」身分，無法替代現有容器和機密隔離。Windows Job Object 可作為行程樹管理的候選機制，但需驗證 ConPTY 子行程能否在執行前加入、子行程繼承與 breakaway、關閉時行程是否存活，以及既有外層 Job 的衝突。現有非 Linux「警告後繼續啟動」路徑須在 Windows 正式執行模式改為 fail closed；若無法證明清理與隔離，該路線維持實驗版且拒絕宣稱正式安全模式。

Windows tailnet 入口也尚無已驗證方案。候選是每個專用 VM 擁有獨立 tailnet 身分並轉送到 runtime loopback；Windows 容器內 sidecar／共享網路命名空間可另行驗證。不能直接把監聽位址改成主機外部介面，或借用含主機權限的既有 tailnet 身分，來宣稱與 Linux manifest 等價。代理 CLI 變體須逐一查證 Windows 二進位檔、授權及執行方式；沒有 Windows 版本者不得發布相應原生變體。

## 4. 需求與驗收條件

### R1. 協定相容

- A、B、C 均維持現有 admin 與 PTY endpoint、授權語意、訊息格式、offset 與 close code；既有 Linux 客戶端不需因 Windows 版而改動。
- Windows PowerShell 及 `cmd.exe` 在 ConPTY 下測試 UTF-8、控制鍵、resize、程式退出、重連與重播；binary PTY bytes 的順序與 offset 不得變更。
- 連線接管、renew、restart、TTL、慢速客戶端、意外斷線及 runtime 更換需通過與 Linux 相同的端對端案例。

### R2. 安全與隔離

- Shell 無管理憑證、服務端機密、宿主機憑證或未授權的工作階段權限；工作階段 A 不能管理 B。用對抗測試驗證，而非只檢查設定檔。
- Windows 環境變數名稱按不分大小寫處理；`Path`／`PATH`、`pty_forward_path` 等大小寫變體不得繞過現有 allowlist 或覆寫規則。Windows 特有必要變數須逐項列入 allowlist，且不得洩漏秘密。
- 播種檔案對 Windows 路徑、磁碟代號、UNC、symlink／junction、hard link、大小寫碰撞與檔案權限採用平台安全檢查；不支援的項目明確拒絕，且在工作階段可見前失敗。
- Windows 瀏覽器提供者只暴露核准工具，明確綁定一個工作階段；撤銷或 TTL 結束後，新的工具呼叫必須失敗，既有呼叫須有定義好的取消或完成語意。
- 任何平台都維持 loopback 監聽與可信網路入口；服務端不得因移植而放寬現有非 loopback bind guard。

### R3. 清理與故障

- 刪除、restart、工作階段 TTL、runtime 關閉與 startup hook 超時，均測試父子孫行程以及嘗試脫離 Job 的程式。存活者須被稽核並在 API 中如實標示。
- 認證失敗、seed 失敗、隔離建立失敗、工具提供者掉線與 tailnet 中斷必須各有明確結果；不能靜默降級為較弱的安全模式。
- Windows 原生版若尚未通過 R2 的隔離驗收，只能以實驗標籤提供，不得列入正式支援矩陣。

## 5. 實作工作包與決策門檻

1. **基線與 WSL2 驗證。** 新增 WSL2 部署指南與可重現的 smoke／對抗檢查；使用現有 Linux 映像驗證 R1、R2、R3。若容器安全設定或 tailnet 入口不成立，先修正部署，不擴大支援聲明。
2. **Windows 瀏覽器 lender。** 在提供者倉庫實作 MCP lender；本倉庫增加通用平台措辭、跨平台協定測試與授權／撤銷案例。至少驗證一個代理 CLI 可以呼叫瀏覽器工具，以及一個未連線／撤銷案例。
3. **原生 Windows 技術試驗。** 增加 Windows CI 的 `cargo check`、Clippy 與單元測試；以 ConPTY 驗證 PowerShell／cmd；單獨驗證 Job Object 接入點及行程逃逸。這一步只決定技術可行性，不發布正式映像。
4. **原生 Windows 安全原型。** 在隔離部署內實作不同身分、機密 ACL、暫存工作區及獨立 tailnet 身分；完成等同 Linux adversary suite 的 Windows 測試後，再決定 Windows 發行包與代理變體。
5. **正式發布門檻。** R1–R3 全數通過、Linux 既有 CI 無退化、部署文件可由第二人重現、平台功能差異寫入安全與客戶端文件，才將原生 Windows 版標為支援。

粗估（單一熟悉 Rust 與 Windows 系統 API 的工程師、未含外部提供者團隊排程）：A 的部署驗證約 3–7 工作日；B 的 Windows 提供者與整合約 2–5 週；C 的功能原型約 3–6 週，安全與部署達正式支援通常另需 6–12 週以上。這是規劃級估算，**未經原型驗證**；ConPTY 與 Job Object 整合、Windows 代理 CLI 可用性及 tailnet 入口是主要變數。

## 6. 尚待證實的事項

| 問題 | 決定方式 | 對支援聲明的影響 |
| --- | --- | --- |
| WSL2 Linux 容器能否維持所有現有沙盒條件？ | 實機 smoke 與 adversary 測試 | 未通過前標「可嘗試」，不標「已驗證」 |
| Windows lender 的所有權與瀏覽器權限模型？ | 提供者原型與授權審查 | 未定義前不宣稱瀏覽器功能已交付 |
| ConPTY 子行程能否可靠歸入 Job 並在 teardown 清理？ | Windows 實機對抗測試 | 未通過前不宣稱原生 Windows 功能對等 |
| 選用 VM／Hyper-V 容器後能否保留獨立 tailnet 身分與 loopback-only listener？ | 部署原型及封包／權限驗證 | 未通過前不發布正式原生 Windows 部署 |
| 哪些代理 CLI 有 Windows 版本且可與相同工作區並行？ | 逐變體建置與 license 清單 | 只發布通過驗證的變體 |

## 7. 參考依據

- 本倉庫：[README](../README.md)、[DESIGN](../DESIGN.md)、[SECURITY](../SECURITY.md)、[client contract](../runtime/CLIENT-CONTRACT.md)、[CI](../.github/workflows/ci.yml)、[image workflow](../.github/workflows/image.yml)。
- Microsoft：[WSL2 架構](https://learn.microsoft.com/en-us/windows/wsl/compare-versions)、[ConPTY API 與最低版本](https://learn.microsoft.com/en-us/windows/console/createpseudoconsole)、[Job Objects 與 breakaway](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)、[Windows 容器隔離模式](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)、[Windows 容器安全邊界](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-security)。
- `portable-pty`：[0.9 原始碼中的 Windows ConPTY 對應](https://docs.rs/crate/portable-pty/0.9.0/source/src/lib.rs)。
