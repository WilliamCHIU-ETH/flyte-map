# Step 1 探索報告：flyteorg/flyte（Flyte 2 後端）

> 進度：Step 1 完成、Step 2 完成（見 step2-五問.md）、Step 3 圖表規劃待本人審閱後開始。
> 快照：`flyteorg/flyte` 分支 `main` @ `f7628a55fd913ff0f126259802379d79ff4f0908`，最新 release `v2.0.48`（2026-09-03），Go 1.26.5，module `github.com/flyteorg/flyte/v2`。
> 本報告只有文字，沒有圖。所有「證據」欄位為 repo 內路徑；標記「推論」者為跨檔案拼出的理解，尚未逐行驗證。

## 1. 一段話說明

Flyte 2 是一個 Kubernetes 原生的工作流與 AI 任務執行平台的**後端**。使用者在 Python SDK（另一個 repo `flyteorg/flyte-sdk`）裡把函式標成 task，`flyte.run()` 會把它送到這個後端；後端把每一次執行記成一棵 **Run → Action 樹**，用一個統一的 `flyte-manager` 二進位提供約 15 個 gRPC/Connect 服務，並由一個 Kubernetes operator（`executor`）把每個 action 變成 `TaskAction` 自訂資源，再透過 plugin 產生 Pod、Ray、Spark 等實際工作負載。這個 repo 同時是所有 API 的契約來源：`flyteidl2/` 的 protobuf 用 buf 產出 Go、TypeScript、Python、Rust 四種 client。Flyte 1 仍在 `master` 分支維護，兩者結構完全不同。

## 2. 新手必須先懂的 8 個核心概念

1. **Task、TaskSpec、TaskTemplate**：可部署的最小單位。`TaskTemplate.type` 是字串（如 `container`、`spark`、`ray`），決定由哪個 plugin 執行；`target` 是 `container`、`k8s_pod` 或 `sql`；`custom` 是給 plugin 用的自由 Struct。Task 以 org、project、domain、name、version 五段識別。證據：`flyteidl2/core/tasks.proto`、`flyteidl2/task/task_definition.proto`、`flyteidl2/task/task_service.proto`（DeployTask、ListVersions）。
2. **Run 與 Action 樹**：一次執行是一個 Run，Run 的內容是一棵 Action 樹，根節點慣稱 `a0`。Action 有三種：`TaskAction`（跑 task）、`TraceAction`（記錄本地 worker 內非確定性的事，帶來確定性）、`ConditionAction`（人機互動或外部訊號，會停在 PAUSED 等 Signal）。證據：`flyteidl2/workflow/run_definition.proto`、`flyteidl2/workflow/run_service.proto`（SignalEvent）。
3. **ActionPhase 狀態機**：`QUEUED → WAITING_FOR_RESOURCES → INITIALIZING → RUNNING → {SUCCEEDED, FAILED, ABORTED, TIMED_OUT}`；條件動作走 `PAUSED → {SUCCEEDED, TIMED_OUT, ABORTED}`；復原執行時直接記為 `RECOVERED`。證據：`flyteidl2/common/phase.proto`。
4. **沒有 DAG 編譯，子 action 由 SDK 端動態排入**：`flyteidl2/workflow/` 裡沒有任何 workflow 定義或 compiler 的 proto，只有 run 與 action。`ActionsService.Enqueue` 帶 `parent_action_name`，`WatchForUpdates` 讓父 action 的程式訂閱所有子 action 的狀態。也就是父 task 的 Python 程式在自己的 Pod 裡當 controller，邊跑邊決定下一個子 action。證據：`flyteidl2/actions/actions_service.proto`、`manager/README.md` 的「How It Works」第 4 點提到 sdk controller、`flyte-sdk` 有 `src/flyte/_internal/controllers` 與 `rs_controller/`。標記：**推論**，SDK 側的實作細節不在本 repo。
5. **TaskAction CRD 與 Executor**：每個要跑的 action 是一個 Kubernetes 自訂資源 `TaskAction`，spec 內含序列化的 `TaskTemplate`、`taskType`、`inputUri`、`runOutputBase`、`cacheKey`。Executor 是 controller-runtime operator，用三個 condition（`Progressing`、`Succeeded`、`Failed`）加 reason（Queued、Initializing、Executing、Completed、RetryableFailure、PermanentFailure、Aborted、PluginNotFound、InvalidSpec、Paused、Signaled、TimedOut）表達狀態；終態後不再 reconcile；用 finalizer `flyte.org/plugin-finalizer` 保證刪除時先 Abort plugin 資源。證據：`executor/api/v1/taskaction_types.go`、`executor/pkg/controller/taskaction_controller.go`、`executor/config/crd/bases/`。
6. **Plugin 體系沿用 Flyte 1 的 pluginmachinery**：`flyteplugins/go/tasks/pluginmachinery/` 定義 plugin 介面、Phase、PhaseInfo、catalog（快取）與 flytek8s（Pod 建構）。目前載入的 plugin：`pod`、`clustered`（JobSet）、`ray`、`spark`、`dask`、kubeflow 的 `mpi`、`pytorch`、`tensorflow`、`core/sleep`，以及 `webapi/connector`（把 task 轉交給外部 connector 服務）。證據：`executor/plugins/loader.go`、`executor/setup.go`、`flyteplugins/go/tasks/plugins/`。
7. **契約優先與多語言產碼**：`flyteidl2/*.proto` 是唯一事實來源，`make gen` 用 buf 產出 `gen/{go,ts,python,rust}` 並一起 commit；CI 的 `check-generate` 會驗證產物與 proto 一致；`flyteidl2/` 目錄改動需要 `@flyteorg/flyteidl2-contributors` 團隊核可。證據：`buf.yaml`、`buf.gen.*.yaml`、`.github/workflows/check-generate.yml`、`CODEOWNERS`。
8. **統一 manager 與可拆分部署**：`flyte-manager` 一個程序跑全部服務，`--component` 旗標可只跑 `runs`、`actions`、`events`、`secret`、`cache`、`app`、`dataproxy`、`executor` 其中之一，對應 Helm 的 `flyte-binary`（單一二進位）與 `flyte-core`（分散式）兩種 chart。所有服務共用一個 PostgreSQL，Connect 服務都掛在 8090 埠。證據：`manager/cmd/main.go`、`manager/cmd/components.go`、`charts/flyte-binary/`、`charts/flyte-core/`。

## 3. 端到端流程：從 `flyte.run()` 到 Pod 結束

1. SDK 先用 `TaskService.DeployTask` 註冊 task（或以 inline `task_spec` 直接送），再呼叫 `RunService.CreateRun`。證據：`flyteidl2/workflow/run_service.proto` 的 `CreateRunRequest` 有 `task_id`、`task_spec`、`trigger_name` 三選一。
2. `RunService.CreateRun` 取回 TaskSpec、依 org、domain、project 的 Settings 合併 RunSpec（queue、labels、env、service account 等）、把 inputs 存到物件儲存、寫入 `tasks`、`task_specs`、`actions` 三張表，最後呼叫 `ActionsService.Enqueue` 排入根 action。證據：`runs/service/run_service.go` 的 `CreateRun`、`persistRunModel`；`runs/migrations/sql/20260408110000_init_schema.sql`。
3. `ActionsService.Enqueue` 在 Kubernetes 建立 `TaskAction` CR。證據：`actions/service/actions_service.go`、`actions/k8s/client.go`。
4. Executor 的 `Reconcile` 依 `actionType` 分流：`condition` 沒有 plugin 也沒有 Pod，等 Signal 或 timeout；`task` 先加 finalizer，依 `taskType` 從 PluginRegistry 找 plugin，找不到就記 `PluginNotFound` 終態。證據：`executor/pkg/controller/taskaction_controller.go` 的 `Reconcile`、`reconcileTask`、`reconcileCondition`。
5. Plugin 產生實際資源。`pod` plugin 用 flytek8s 組 Pod；raw container 會加 `flyte-copilot` 的 init container（下載 inputs）與 sidecar（等主程序結束後上傳 outputs）。資料走 `DataProxyService` 發的 signed URL 與 `RunSpec.raw_data_storage.raw_data_prefix`，不經 API server。證據：`flytecopilot/README.md`、`flyteplugins/go/tasks/pluginmachinery/flytek8s/copilot.go`、`dataproxy/README.md`。
6. 每次 phase 變化寫回 CR status（含 `PhaseHistory`），並透過 `InternalRunService.RecordActionEvents` 或 `EventsProxyService.Record` 記到 `action_events` 表。證據：`flyteidl2/workflow/internal_run_service.proto`、`flyteidl2/workflow/events_proxy_service.proto`、`executor/pkg/controller/event_batcher.go`。
7. `ActionsService` 用 shared informer 監看 CR，`WatchForUpdates` 先訂閱、再送目前快照、再送一個 sentinel、之後持續推送，保證至少一次送達。父 task 的 SDK controller 收到子 action 終態後決定下一步，再 `Enqueue` 新的子 action，重複 3 到 7。證據：`actions/service/actions_service.go` 的 `WatchForUpdates`；`flyteidl2/workflow/state_service.proto` 的 `ControlMessage.sentinel` 註解。
8. CLI 與 UI 透過 `RunService.WatchRunDetails`、`WatchActions` 從資料庫讀樹狀進度；`runs/service/run_state_manager.go` 在記憶體維護 action 樹與各 phase 計數。Abort 走 `AbortRun` 寫 DB 後由 `AbortReconciler` 背景終止 Pod。證據：`runs/service/run_service.go`、`runs/service/abort_reconciler.go`。
9. Executor 的 `garbage_collector.go` 在終態後清理 CR。快取命中時透過 `CacheService`（`cache_service_outputs`、`cache_service_reservations` 兩表）省掉執行。證據：`executor/pkg/controller/garbage_collector.go`、`cache_service/migrations/sql/`。

## 4. 主要入口點

| 想做什麼 | 從哪裡進 |
|---|---|
| 讀懂整體架構 | `manager/README.md`、`docs/BACKEND_README.md`、`executor/README.md` |
| 跑起一套本地環境 | `make devbox-build`、`make devbox-run FLYTE_DEV=true`、`make -C manager run`；映像檔在 `docker/devbox-bundled/`，chart 在 `charts/flyte-devbox/`（內含 k3d、knative-serving、rustfs、docker-registry） |
| 服務的程式入口 | `manager/cmd/main.go`；各服務各有 `cmd/main.go` 可獨立啟動 |
| 改 API | `flyteidl2/<領域>/*.proto`，然後 `make gen` |
| 加或改 task plugin | `flyteplugins/go/tasks/plugins/`，在 `executor/plugins/loader.go` 或 `executor/setup.go` 註冊 |
| 排程與觸發 | `runs/scheduler/`、`runs/schedule/cron.go`、`flyteidl2/trigger/` |
| 部署 | `charts/flyte-binary`、`charts/flyte-core`、`charts/flyteconnector` |
| CI 在檢查什麼 | `.github/workflows/go-tests.yml`、`check-generate.yml`（產碼、go tidy、mocks）、`pr-quality.yml`（anti-slop）、`devbox.yml`（用 flyte-sdk examples 做功能測試）、`flyte-binary-v2.yml`（映像檔） |
| 觀測 | `monitoring/dashboards/flyte-execution.json`、`make devbox-monitoring` |
| 提案流程 | `docs/rfcs/`，目前只有一份 Settings Service RFC |

## 5. 這個專案強制的規則與不變量

- Proto 相容性：不改欄位號碼、不改型別、移除欄位用 `reserved`。`RunSpec` 與 `TaskAction` 裡的 `cluster` 已被 reserved，改用 `queue` 概念。證據：`CONTRIBUTING.md`、`flyteidl2/task/run.proto`。
- 產出的程式碼要一起 commit，CI 會比對；PR 必須 `git commit -s` 簽 DCO；commit 用 conventional commits。證據：`CONTRIBUTING.md`、`.github/dco.yml`。
- 從 fork 開的 PR 不跑完整 CI：映像檔建置被略過，`check-generate` 用備援映像檔。維護者要把綠燈當「沒跑」而不是「通過」。證據：`CONTRIBUTING.md` 的「CI Checks on Pull Requests from Forks」。
- `flyteidl2/` 需要另一個團隊核可；其餘預設 `@flyteorg/flyte2-contributors`。證據：`CODEOWNERS`。
- `TaskAction` 進入 `Succeeded` 或 `Failed` 後永不再 reconcile。證據：`executor/api/v1/taskaction_types.go` 的註解。
- 狀態串流是至少一次送達，並以 sentinel 區分「現況快照」與「新更新」。證據：`flyteidl2/workflow/state_service.proto`、`flyteidl2/actions/actions_service.proto`。
- Settings 沿 instance → domain → project 繼承，越具體者勝出；`runs/service/settings_*.go` 與 `20260812100000_add_settings_table.sql` 顯示服務已部分落地。證據：`docs/rfcs/20260804_settings_service.md`。
- `ActionsService` 是設計上要取代 `StateService` 與 `QueueService` 的單一介面；後兩者仍在 proto 中。證據：`flyteidl2/actions/actions_service.proto` 開頭註解。
- `TrackedRunService` 只記錄在平台外自行執行的 run，平台不排程也不介入。證據：`flyteidl2/workflow/tracked_run_service.proto`。
- Flyte 1 的 `flyteadmin`、`flytepropeller`、`datacatalog`、`flytectl` 都不在 `main`；沿用的是 `flytestdlib`、`flyteplugins` 的 pluginmachinery、`flytecopilot`。詞彙對照：Flyte 1 的 workflow、execution、node 對應 Flyte 2 的 task、run、action。

## 6. 貢獻入口的現況（快照當日）

| 項目 | 數字 |
|---|---|
| 開放 PR | 70 |
| 開放 issue 標 `flyte2` | 28 |
| 開放 issue 標 `good first issue` 且 `flyte2` | 11 |
| 開放 issue 標 `help wanted` | 4 |

`flyte2` 的 good first issue 幾乎都是可觀測性：executor、actions、runs 的 Prometheus 與 OTEL 指標，另有一個 proto 打字錯誤 `funtion_name`。社群入口是 `slack.flyte.org`。近兩週有非核心團隊帳號的 `clustered` plugin commit 被合併，顯示外部 PR 是走得通的。

## 7. 尚未確認的事項（進 Step 3 前需要決定要不要查）

1. SDK 端 controller 到底跑在哪個程序、用哪幾個 RPC，只能從 flyte-sdk 確認。
2. `queue` 如何對應到叢集：`ClusterService.SelectCluster` 與 `dataproxy/service/cluster_service.go` 存在，但選擇邏輯未讀。
3. `EventsProxyService` 提到可轉送到 Redis stream，何時走 Redis 未讀。
4. App Service 推測是用 Knative 部署使用者的長駐服務（如 FastAPI 模型服務），依據是 `charts/flyte-devbox` 依賴 knative-serving 與 `app_definition.proto` 的 replica、cold start 註解。標記：推論。
5. `WAITING_FOR_RESOURCES` 由誰判定、`RunSpec` 的 run 級並行上限由哪個排程器執行，未讀。
