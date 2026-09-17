# Step 2 五問：Flyte 2 這個主題站不站得住

> 只根據 `step1-探索報告.md` 回答。標記「假設」者是報告證據不足、需要外部資料才能確認的判斷。

## 1. 解決什麼問題？

一個團隊有很多 Python 函式要在 Kubernetes 上跑成 ML 或 AI 流程，需要每一步都能重試、快取、追蹤輸入輸出、限制資源、跨 project 與 domain 集中治理，而且流程的形狀常常要到執行中才知道（依前一步結果決定 fan-out 幾個、要不要等人簽核）。Flyte 2 把「一次執行」記成 Run 底下的 Action 樹，每個 Action 都是可重試、可快取、有型別介面的獨立單位，並提供 secrets、settings、queue、trigger 這些平台治理面。相對 Flyte 1，它另外要解的問題是靜態 DAG 編譯限制了動態控制流，以及多 repo 拆分帶來的維護成本；後者從 `main` 只留一個 module、一個 manager 二進位可以佐證，前者從 proto 裡完全沒有 workflow 定義可以佐證。

## 2. 考慮過什麼替代方案？

repo 內可見的取捨有兩個。第一，用「StateService 加 QueueService」兩個介面還是「ActionsService」一個介面：proto 註解直接寫 ActionsService 是為了取代前兩者，理由是狀態與排隊本來就綁在同一個 action 上。第二，用 Kubernetes CRD 當每個 action 的排隊與狀態載體，而不是自建 queue：好處是重用 informer、finalizer、RBAC 與 etcd 的一致性，代價在第 4 問。repo 外的替代方案屬於產品層判斷，這裡標為假設：Airflow 是排程器中心且沒有型別化資料面；Argo Workflows 是 YAML 靜態 DAG；Prefect 與 Dagster 有 Python 動態流程但資料面與多叢集執行較弱；Temporal 是通用 durable execution 而非資料與 ML 導向。這些比較要留到 Step 4 寫 00 頁時再核對。

## 3. 怎麼解決？

機制分三個角色，各有明確邊界。RunService 記帳：把 task、run、action、event 寫進 PostgreSQL，並對外提供 List 與 Watch。ActionsService 排隊與觀察：把 action 變成 TaskAction CR，用 informer 監看，並以至少一次送達的串流把子 action 狀態推給父 action。Executor 執行：controller-runtime operator 依 taskType 找 plugin，plugin 產生 Pod、Ray、Spark 等資源，copilot sidecar 負責搬資料。SDK 在父 task 的 Pod 裡當 controller，邊跑邊 Enqueue 子 action，所以後端不需要知道流程長什麼樣。所有 API 由 flyteidl2 的 proto 定義並產出四種語言 client，`flyte-manager` 一個二進位可整合跑也可拆開跑。

## 4. 有什麼限制與風險？

第一，只支援 Kubernetes，且每個 action 一個 CR、TaskTemplate 以序列化 bytes 放在 etcd，大規模 fan-out 時 etcd 與 API server 會成為壓力點；標記假設，未讀到相關限流或分片機制。第二，API 仍在快速演進：`GetActionData` 已標 deprecated 改走 DataProxy、`cluster` 欄位被 reserved 改成 `queue`、Settings 服務只部分落地、StateService 與 QueueService 仍與 ActionsService 並存。第三，貢獻流程的摩擦：fork PR 不跑完整 CI，維護者要另外驗證；proto 改動需要第二個團隊核可。第四，文件分散：架構文件在 repo 內，使用者文件在 union.ai，Flyte 1 與 2 雙軌維護，新手很容易讀到錯的那一版。第五，SDK 端 controller 是理解全貌的關鍵，卻不在這個 repo 裡。

## 5. 是最佳方案嗎？

對「以 Python 為主、以 Kubernetes 為底、需要型別化資料與快取、又要動態控制流」的團隊，這是目前最一致的解：一個契約、一個二進位、一種執行載體，理解成本集中在 Run 與 Action 樹一個概念上。它刻意不優化的事也很清楚：不支援非 Kubernetes 執行環境；不做通用 durable execution，`TrackedRunService` 只記錄在平台外自行跑的 run，不排程也不介入；不做 UI 優先的低碼編排，流程形狀完全由 Python 程式決定。這三個「不」就是 00 頁「它不是什麼」表格的來源。
