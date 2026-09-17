# 旅程 1：跑起 devbox，看一個 run 從頭到尾

> 這一頁回答：怎麼在本機把整套後端跑起來、建立一個 run，並在三個地方看到它的痕跡。圖沿用 [devbox 拓樸](../03-how-it-runs/devbox-topology.md)。

## 事前準備

`CONTRIBUTING.md` 列的需求：Buf CLI、Go 1.26.5 以上、Node.js 與 npm、Python 3.10 以上與 `uv`、Rust 工具鏈（只在動 Rust 產物時需要）、Docker。第一次 `make gen` 前先 `make docker-pull`。

## 步驟

| 步 | 指令或動作 | 你應該看到 |
|---|---|---|
| 1 | `make devbox-build` | Docker 建出 devbox 映像檔 |
| 2 | `make devbox-run FLYTE_DEV=true` | k3d 叢集啟動，kubeconfig 被寫入，`kubectl get nodes` 有回應 |
| 3 | `make -C manager run` | 日誌顯示連上 PostgreSQL、跑 migration、各服務啟動、開始 reconcile |
| 4 | `curl localhost:8090/healthz` 與 `curl localhost:8081/healthz` | 兩個都回健康 |
| 5 | 另開終端機，安裝 SDK `uv pip install flyte`，把 `README.md` 的 hello world 存成 `hello.py`，依 union.ai 的 devbox 文件把 endpoint 指到本機，執行 | 終端機印出 run 的名稱與結果 |
| 6 | `kubectl get taskactions -n flyte -w` | 看到 `a0` 與子 action 的 CR 出現、condition 變化、終態後被 GC |
| 7 | `psql -h localhost -p 30001 -U postgres -d runs` 後 `SELECT name, phase, state FROM actions;` | 每個 action 一列，phase 已是終態 |
| 8 | `make devbox-stop` | 叢集停止 |

第 5 步的 endpoint 設定方式在 `flyte-sdk` 與 union.ai 文件，本地圖不涵蓋；`runs/test/scripts/create_run.sh` 是不經 SDK、直接打 API 的參考。

## 三個地方看痕跡

| 地方 | 看什麼 | 對應概念 |
|---|---|---|
| manager 日誌 | `CreateRun`、`Enqueue`、`Reconcile` 的訊息 | [端到端流程](../03-how-it-runs/create-run-to-pod.md) |
| `kubectl` | `TaskAction` 的 conditions 與 reason | [TaskAction 與 Executor](../02-core-concepts/taskaction-crd-and-executor.md) |
| PostgreSQL | `actions`、`action_events` 表 | [統一 manager](../02-core-concepts/manager-and-services.md) |

## 卡住時

`manager/README.md` 的 Troubleshooting 段落列了三種常見錯誤：拿不到 kubeconfig、資料表不存在（重啟 manager 讓 migration 跑）、埠被占用（改 `manager/config.yaml`）。

## 做完這條旅程你應該能

把 [端到端流程](../03-how-it-runs/create-run-to-pod.md) 的九步，對應到你在日誌、`kubectl`、資料庫裡看到的東西。

## 證據

`CONTRIBUTING.md`、`Makefile`、`manager/README.md`、`manager/config.yaml`、`runs/test/scripts/`、`README.md`。

<!-- nav -->
---

上一頁 [04 · CI 檢查什麼、誰來核](../04-code-terrain/ci-and-owners.md) ｜ [總覽](../README.md) ｜ 下一頁 [05 · 旅程 2：從 issue 到檔案](./from-issue-to-code.md)
