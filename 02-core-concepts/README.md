# 02 · 核心概念：術語表與八個必懂概念

> 這一頁回答：Flyte 2 的詞彙對應 Flyte 1 的什麼，以及八個核心概念各是什麼、去哪一頁看圖。

## 術語表

Flyte 1 與 Flyte 2 換了一套名詞。讀 issue、Slack 或舊文章時先對照這張表，避免把兩代混在一起。

| Flyte 1 | Flyte 2 | 一句解釋 |
|---|---|---|
| workflow | 沒有獨立概念 | 流程就是父 task 的 Python 程式，見 [沒有 DAG](./no-dag-dynamic-enqueue.md) |
| execution | run | 一次執行，含一棵 action 樹 |
| node | action | 樹上的一個節點，有 task、trace、condition 三種 |
| task | task | 不變。可部署的最小單位，有 org、project、domain、name、version |
| launch plan | trigger（近似） | 帶排程或自動化條件的任務入口，由 `TriggerService` 管 |
| project、domain | org、project、domain | 多了一層 org |
| flyteadmin | runs、actions 等多個服務 | 記帳與 API 拆進 `flyte-manager` 的各 component |
| flytepropeller | executor 加 SDK 端 controller | 執行拆成兩半：Kubernetes operator 與父 task 內的 Python |
| datacatalog | cache_service | 任務輸出快取 |
| flytectl | flyte CLI | CLI 移到 `flyte-sdk` repo |
| flyteidl | flyteidl2 | 契約，同 repo 內 |
| flyteconsole | flyteconsole，經 app 服務代理 | UI 仍是獨立 repo |
| 沒有 | TaskAction | 每個 action 在 Kubernetes 上的自訂資源 |
| 沒有 | TraceAction、ConditionAction | 記錄非確定性結果、等待外部訊號 |

## 八個概念總表

| # | 概念 | 一句話 | 頁面 |
|---|---|---|---|
| 1 | Task、TaskSpec、TaskTemplate | 可部署的最小單位，`type` 字串決定由誰執行 | [run-and-action-tree.md](./run-and-action-tree.md) |
| 2 | Run 與 Action 樹 | 一次執行是一棵樹，根是 `a0`，節點有三種 | [run-and-action-tree.md](./run-and-action-tree.md) |
| 3 | ActionPhase 狀態機 | 十個狀態，五個終態 | [action-phase.md](./action-phase.md) |
| 4 | 沒有 DAG，子 action 由 SDK 端動態排入 | 流程形狀由父 task 的 Python 在執行中決定 | [no-dag-dynamic-enqueue.md](./no-dag-dynamic-enqueue.md) |
| 5 | TaskAction CRD 與 Executor | 每個 action 是一個 CR，operator 用 condition 與 reason 表達狀態 | [taskaction-crd-and-executor.md](./taskaction-crd-and-executor.md) |
| 6 | Plugin 體系 | `taskType` 找到 plugin，plugin 產生 Pod 或其他 CR | [plugins.md](./plugins.md) |
| 7 | 契約優先與多語言產碼 | proto 是唯一事實來源，`make gen` 產四種語言 | [contract-and-codegen.md](./contract-and-codegen.md) |
| 8 | 統一 manager 與資料面分離 | 一個二進位裝八個 component；資料不經控制面 | [manager-and-services.md](./manager-and-services.md)、[control-plane-vs-data-plane.md](./control-plane-vs-data-plane.md) |

建議順序就是表格順序。1 到 4 是資料模型，5 到 6 是執行，7 到 8 是工程結構。

## 證據

`flyteidl2/` 各套件名稱、`manager/README.md`、Flyte 1 的名詞來自 `master` 分支的目錄名。
