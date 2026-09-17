# 哪些 label 適合新手

> 這一頁回答：issue tracker 裡上百個 label，新手該先看哪幾個、避開哪幾個。數字為快照當日（2026-09-17）的開放數，只供規模感。

## 先看這些

| label | 開放數 | 意思 | 給新手的建議 |
|---|---|---|---|
| `good first issue` | 15 | 維護者認為適合入門 | 其中 11 個同時標 `flyte2`，多為指標與可觀測性 |
| `flyte2` | 28 | 屬於 Flyte 2，即 `main` 分支 | 本地圖範圍內的都在這裡 |
| `help wanted` | 4 | 維護者希望外部協助 | 少但明確 |
| `bug` | 22 | 缺陷 | 兩代混在一起，先確認有沒有 `flyte2` |

## 這些代表流程狀態，不是題目

| label | 意思 |
|---|---|
| `needs-rebase` | 分支落後，先 rebase |
| `review-needed`、`lgtm` | 審查狀態 |
| `needs-decision`、`needs investigation`、`needs repro steps` | 還沒定案，新手不宜直接動手 |
| `do-not-merge`、`blocked` | 有前置條件 |
| `Code agent slop` | 維護者標記低品質 AI 產出的 PR，自我警惕用 |
| `lf-cla: no` | 自動化標記，與 DCO 相關 |

## 這些是 Flyte 1 的地盤

`flyteadmin`、`flytepropeller`、`flytectl`、`flyteidl`（沒有 2）、`flytekit`、`flytesnacks`、`datacatalog`、`propeller`、`local flytekit`。看到這些就走 [哪個分支或 repo](./which-branch-or-repo.md) 的決策樹。

## 三個實用習慣

1. 用 `gh issue list -R flyteorg/flyte --label flyte2 --label "good first issue"` 每週看一次。
2. 挑之前先看 issue 的建立日期與最後留言，太舊的先問還要不要做。
3. 認領時留言說你打算怎麼做，讓維護者早點糾正方向。

## 證據

快照當日 `gh api repos/flyteorg/flyte/labels` 與 `gh issue list` 的結果、`.github/labeler.yml`、`.github/workflows/pr-quality.yml`。

<!-- nav -->
---

上一頁 [06 · proto 還是 Go](./proto-or-go.md) ｜ [總覽](../README.md) ｜ 讀完了
