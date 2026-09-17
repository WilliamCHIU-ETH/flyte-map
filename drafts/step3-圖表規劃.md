# Step 3 圖表規劃：flyte-map 的頁面與 22 張圖

> 進度：Step 1、2 已定案（核心概念不變；未確認事項列入邊界）。本檔是 Step 3 交付物，待本人批准後才進 Step 4 畫圖寫文。
> 規則：一張圖只回答一個問題；每張圖 15 個節點以內；每頁的每一節最多一張圖；圖型只用 GitHub 原生支援的 mermaid（flowchart、sequenceDiagram、stateDiagram-v2）。
> 檔名採英文 kebab-case、內容繁中，理由是連結不會被百分號編碼、換專案時骨架可直接複製。

## 一、頁面骨架（25 個 Markdown）

```
flyte-map/
├── README.md                              快照標記、閱讀路線、讀完能做到什麼、地圖邊界   [R1]
├── 00-what-is-flyte2.md                   一段話、它不是什麼、五問速答、全景圖         [A1]
├── 01-background/
│   └── README.md                          6 個外部概念，每個一段加連結                 [B1]
├── 02-core-concepts/
│   ├── README.md                          術語表（Flyte 1 對 Flyte 2）、8 概念總表     （只有表）
│   ├── run-and-action-tree.md             概念 1、2                                    [C1]
│   ├── action-phase.md                    概念 3                                       [C2]
│   ├── no-dag-dynamic-enqueue.md          概念 4                                       [C3]
│   ├── taskaction-crd-and-executor.md     概念 5                                       [C4]
│   ├── plugins.md                         概念 6                                       [C5]
│   ├── manager-and-services.md            概念 8                                       [C6]
│   ├── contract-and-codegen.md            概念 7                                       [C7]
│   └── control-plane-vs-data-plane.md     資料面（Step 1 概念 8 的後半）               [C8]
├── 03-how-it-runs/
│   ├── create-run-to-pod.md               端到端 9 步                                  [D1]
│   ├── parent-child-watch.md              父子 action 的 Watch 與 Enqueue；Condition    [D2][D3]
│   ├── failure-retry-abort.md             失敗、重試、超時、中止                       [D4]
│   └── devbox-topology.md                 本機環境有哪些盒子                           [D5]
├── 04-code-terrain/
│   ├── directory-map.md                   目錄對概念                                   [E1]
│   └── ci-and-owners.md                   CI 檢查什麼、CODEOWNERS 誰核                 [E2]
├── 05-contribution-paths/
│   ├── run-devbox-and-watch-a-run.md      旅程 1，沿用 D5                              （不新畫）
│   ├── from-issue-to-code.md              旅程 2                                       [F1]
│   ├── change-a-proto-field.md            旅程 3，沿用 C7                              （不新畫）
│   └── first-pr.md                        旅程 4                                       [F2]
└── 06-decision-trees/
    ├── which-branch-or-repo.md            Flyte 1 或 2、flyte 或 flyte-sdk             [G1]
    ├── proto-or-go.md                     要動 proto 還是只動 Go                       [G2]
    └── labels-for-newcomers.md            哪些 label 適合新手                          （只有表）
```

## 二、22 張圖的清單

格式：**編號 頁面**｜回答的唯一問題｜圖型｜節點（數量）｜證據

### README 與 00

- **R1 README**｜這張地圖該照什麼順序讀？｜flowchart LR｜00 → 01 → 02 → 03 → 04 → 05 → 06，加一條「已有 K8s 背景可跳過 01」的虛線（8）｜本規劃
- **A1 00**｜Flyte 2 由哪些部分組成、誰跟誰說話？｜flowchart LR，三個 subgraph｜使用者側：flyte-sdk、CLI、UI；控制面：flyte-manager、PostgreSQL；資料面：K8s API、TaskAction CR、Executor、Pod、物件儲存（12）｜`manager/README.md`、`docs/BACKEND_README.md`

### 01 背景知識

- **B1 01**｜Kubernetes operator 的 reconcile loop 是怎麼轉的？｜flowchart TD｜使用者建立 CR → API server 存 etcd → informer 通知 → controller Reconcile → 比對期望與現況 → 建或改子資源 → 寫回 status → 回到監看；標註「TaskAction 就是這裡的 CR」（8）｜`executor/README.md`、controller-runtime 一般知識

### 02 核心概念

- **C1 run-and-action-tree**｜一次執行在資料上長什麼樣？｜flowchart TD｜Run → a0 根 TaskAction → 子 TaskAction × 2 → 其中一個再生 TraceAction 與 ConditionAction；旁註 org/project/domain/name 識別、parent 與 group 欄位（10）｜`flyteidl2/workflow/run_definition.proto`
- **C2 action-phase**｜一個 action 會經過哪些狀態，哪些是終態？｜stateDiagram-v2｜QUEUED、WAITING_FOR_RESOURCES、INITIALIZING、RUNNING、SUCCEEDED、FAILED、ABORTED、TIMED_OUT、PAUSED、RECOVERED，終態用 [*] 標示（11）｜`flyteidl2/common/phase.proto`
- **C3 no-dag-dynamic-enqueue**｜流程的形狀由誰決定？｜flowchart LR，左右兩個 subgraph｜左：Flyte 1 的 SDK 編譯 DAG → 註冊 → propeller 照圖執行；右：Flyte 2 的父 task 在 Pod 內執行 Python → Enqueue 子 action → Watch 結果 → 再決定下一步（10）｜`flyteidl2/actions/actions_service.proto`、`manager/README.md`
- **C4 taskaction-crd-and-executor**｜Executor 收到一個 TaskAction 後怎麼分流？｜flowchart TD｜取 CR → 有 deletionTimestamp？→ Abort 並移除 finalizer；否則看 actionType → condition：等 Signal 或 timeout；task：加 finalizer → 查 PluginRegistry → 找不到記 PluginNotFound 終態 → 找到就 Handle → 把 PhaseInfo 對應到 condition 與 reason → 寫 status（12）；condition 與 reason 的完整對照另做表｜`executor/pkg/controller/taskaction_controller.go`、`executor/api/v1/taskaction_types.go`
- **C5 plugins**｜taskType 字串如何找到執行者？｜flowchart LR｜TaskTemplate.type → PluginRegistry → 已載入 plugin：pod、clustered、ray、spark、dask、mpi、pytorch、tensorflow、sleep、connector → 各自產生的 K8s 資源（Pod、JobSet、RayJob、SparkApplication、Kubeflow Job、外部 connector 服務）（14）｜`executor/plugins/loader.go`、`executor/setup.go`
- **C6 manager-and-services**｜一個二進位裡裝了哪些服務、怎麼拆開部署？｜flowchart TD｜flyte-manager → runs、actions、events、secret、cache、app、dataproxy、executor 八個 component；共用 PostgreSQL；共用 8090 埠；旁註 `--component` 旗標與 flyte-binary、flyte-core 兩種 chart（12）｜`manager/cmd/components.go`、`charts/`
- **C7 contract-and-codegen**｜proto 改一行之後會流到哪裡？｜flowchart LR｜flyteidl2/*.proto → buf format、lint → buf generate → gen/go、gen/ts、gen/python、gen/rust → mocks → go mod tidy → commit → CI check-generate 比對（9）｜`Makefile`、`buf.gen.*.yaml`、`.github/workflows/check-generate.yml`
- **C8 control-plane-vs-data-plane**｜資料怎麼進出 Pod 而不經過 API server？｜flowchart LR｜SDK 向 DataProxy 要 signed URL → 直接上傳 inputs 到物件儲存；Pod 的 copilot init container 下載 inputs → 主容器執行 → copilot sidecar 上傳 outputs；CacheService 記 output 位置與 reservation；控制面只持有 URI（11）｜`dataproxy/README.md`、`flytecopilot/README.md`、`cache_service/migrations/sql/`

### 03 運作流程

- **D1 create-run-to-pod**｜`flyte.run()` 之後依序發生什麼？｜sequenceDiagram｜參與者：SDK、RunService、PostgreSQL、DataProxy、ActionsService、K8s API、Executor、Pod（8 參與者、約 14 個訊息）；訊息對應 Step 1 第 3 節的 9 步｜`runs/service/run_service.go`、`actions/k8s/client.go`
- **D2 parent-child-watch**｜父 action 如何得知子 action 完成並排下一個？｜sequenceDiagram｜參與者：父 Pod 內的 SDK controller、ActionsService、informer、K8s API、子 TaskAction（5）；訊息：Subscribe → 送現況快照 → 送 sentinel → 持續推送 → 父 Enqueue 下一個｜`actions/service/actions_service.go` 的 WatchForUpdates
- **D3 parent-child-watch 第二節**｜PAUSED 的 Condition 怎麼被喚醒？｜sequenceDiagram｜參與者：父 Pod、ActionsService、Executor、CLI 或 UI、Webhook 目標（5）；訊息：Enqueue condition → Executor 記 Paused、可選 POST webhook → 使用者 SignalEvent → Executor 記 Signaled → 父收到 SUCCEEDED 與 value｜`flyteidl2/workflow/run_definition.proto` 的 ConditionAction、`run_service.go` 的 SignalEvent
- **D4 failure-retry-abort**｜出錯時 Executor 怎麼決定重試、失敗還是超時？｜flowchart TD｜plugin 回報 PhaseInfo → RetryableFailure？→ 系統錯誤次數是否超過 maxSystemFailures → 重置 plugin 資源重跑 或 記 PermanentFailure；attempt 超過 maxRuntime → Abort plugin → 記 TimedOut；收到刪除 → handleAbortAndFinalize（12）｜`taskaction_controller.go` 的 reconcileTimedOutAttempt、resetPluginResource、finalizePermanentFailure
- **D5 devbox-topology**｜本機開發環境跑起來有哪些盒子？｜flowchart TD｜筆電上的 flyte-manager（FLYTE_DEV=true）→ k3d 叢集內：TaskAction CRD、Knative Serving、RustFS、docker-registry、PostgreSQL（30001）、使用者 Pod；旁註 make devbox-build、devbox-run、devbox-monitoring（10）｜`CONTRIBUTING.md`、`charts/flyte-devbox/Chart.yaml`、`docker/devbox-bundled/`

### 04 程式碼地形

- **E1 directory-map**｜哪個資料夾對應哪個概念？｜flowchart LR，四個 subgraph｜契約：flyteidl2、gen；控制面：manager、runs、actions、events、secret、cache_service、app、dataproxy；資料面：executor、flyteplugins、flytecopilot；共用與部署：flytestdlib、charts、docker、.github（14）｜`git ls-tree` 快照
- **E2 ci-and-owners**｜一個 PR 會被哪些檢查跑過，fork 少了什麼？｜flowchart LR｜PR 開啟 → go-tests、check-generate（產碼、tidy、mocks）、pr-quality、devbox 功能測試、flyte-binary-v2 映像檔 → CODEOWNERS 核可 → 合併；fork PR 的映像檔與完整產碼檢查標成灰色「略過」（12）｜`.github/workflows/`、`CONTRIBUTING.md`、`CODEOWNERS`

### 05 貢獻路徑

- **F1 from-issue-to-code**｜從一個 issue 到該改的檔案怎麼走？｜flowchart TD｜讀 label（flyte2、good first issue）→ 判斷屬於哪一層（契約、控制面、資料面）→ 對應目錄 → 找 service 或 controller 檔 → 找同名 _test.go → 找 CI 中會跑到它的 workflow（10）｜E1 加 Step 1 第 6 節
- **F2 first-pr**｜第一個 PR 從 fork 到合併要過哪幾關？｜flowchart LR｜fork → 建 feature 分支 → 改碼 → make gen 或 make buf-lint → git commit -s → 填 PR template 四段與 label → CI（fork 略過映像檔）→ 維護者本地驗證 → CODEOWNERS 核可 → squash 合併（12）｜`CONTRIBUTING.md`、`.github/PULL_REQUEST_TEMPLATE.md`

### 06 決策樹

- **G1 which-branch-or-repo**｜這個問題屬於哪個分支或 repo？｜flowchart TD｜問題涉及 Python API 或 CLI？→ flyte-sdk；涉及 flyteadmin、propeller、flytectl 名詞？→ master 分支的 Flyte 1；涉及 TaskAction、manager、flyteidl2？→ main 分支的 Flyte 2；UI → flyteconsole；不確定 → 看 issue 的 flyte2 label（12）｜Step 1 第 5 節詞彙對照
- **G2 proto-or-go**｜要動 proto 還是只動 Go？｜flowchart TD｜改的是對外訊息欄位或 RPC？→ 是：動 flyteidl2、跑 make gen、需 flyteidl2-contributors 核可、注意欄位號碼與 reserved；否：只動 Go、跑 go test、check-generate 仍會驗 mocks（9）｜`CONTRIBUTING.md`、`CODEOWNERS`

## 三、只有表格、不畫圖的內容

| 頁面 | 表格 |
|---|---|
| 00 | 它不是什麼（4 列）；五問速答（5 列） |
| 02/README | 術語表：Flyte 1 名詞、Flyte 2 名詞、一句解釋（約 12 列）；8 概念總表 |
| 02/taskaction-crd-and-executor | condition 類型 × reason 的完整對照（12 列） |
| 04/ci-and-owners | workflow 名稱、觸發條件、檢查什麼、fork 是否會跑（9 列） |
| 06/labels-for-newcomers | label、意義、快照當日開放數（約 8 列） |

## 四、README 的地圖邊界（草案）

依本人決定，Step 1 第 7 節的未確認事項全部列入邊界，不在地圖內展開：

1. 不涵蓋 Flyte 1（`master` 分支）與 flyteadmin、flytepropeller 的內部。
2. 不涵蓋 flyte-sdk 的內部，包含 SDK 端 controller 的實作與它用到的 RPC 細節。
3. 不涵蓋 union.ai 的商業功能與託管服務。
4. 不是 Kubernetes、Go、protobuf 的教學；01 只給一段話與外部連結。
5. 不是營運手冊：不談生產環境的規模調校、多叢集的 queue 對應、EventsProxy 的 Redis 路徑、App Service 的 Knative 細節、run 級並行上限的排程器。
6. 與 Airflow、Argo、Prefect、Temporal 的定位比較未經核對，只在 00 以一句話帶過並標明為假設。

## 五、需要本人批准的事項

1. 22 張圖比 Step 2 前估的 15 到 20 張多 2 張。若要刪，建議順序：R1（改成編號清單）、D3（併入 D2 的文字）、G2（改成表格）。
2. 檔名英文、內容繁中的做法。
3. 邊界第 6 條的處理方式：一句話帶過並標假設，或完全不提。
