# 旅程 3：改一個 proto 欄位並重新產碼

> 這一頁回答：改 `flyteidl2/` 裡的一個欄位，從編輯到 PR 要做哪七件事。圖沿用 [契約與產碼](../02-core-concepts/contract-and-codegen.md)。

## 七步

| 步 | 做什麼 | 注意 |
|---|---|---|
| 1 | 編輯 `flyteidl2/<套件>/<檔>.proto` | 新增欄位用新號碼；移除用 `reserved 號碼` 與 `reserved "名稱"`；不改型別；每個欄位要有註解 |
| 2 | `make buf-format` 與 `make buf-lint` | lint 不過 CI 會擋 |
| 3 | `make docker-pull` 一次，然後 `make gen` | 或 `make gen-local` 用本機 buf、go、cargo、uv |
| 4 | 驗證四種產物 | Go：`cd gen/go && go build ./... && go test ./...`；TypeScript：`cd gen/ts && npm install`；Python：`cd gen/python && uv lock`；Rust：`make build-crate` |
| 5 | 若改了 Go 介面，`make mocks` | CI 的 `check-mocks` 會比對 |
| 6 | 把 proto 與 `gen/` 的變更一起 commit，`git commit -s` | 兩者分開 commit 會讓 `check-generate` 在中間狀態失敗 |
| 7 | 開 PR，說明相容性影響 | `flyteidl2/` 需要 `@flyteorg/flyteidl2-contributors` 核可 |

## 相容性的判斷

問自己一個問題：舊版 client 讀到新版訊息、或新版 client 讀到舊版訊息，會不會壞？

- 只新增可選欄位：安全。
- 改名但不改號碼：wire 相容，但 JSON 名稱與產碼的識別字都會變，下游程式碼要跟著改。issue #7558 的 `funtion_name` 就屬這類。
- 改號碼或型別：不相容，`CONTRIBUTING.md` 明文禁止。
- 刪欄位：必須 `reserved`，否則號碼被重用會靜默出錯。

## 跨語言的注意

`flyteidl2/gen_utils/` 裡有各語言的產碼設定；Python 產物會發到 PyPI 的 `flyteidl2`，Rust 到 crates.io 的 `flyte`，npm 到 `@flyteorg/flyte`。改動若影響套件結構，這裡也要看。

## 做完這條旅程你應該能

自己判斷一個 proto 改動屬於哪一級相容性，並在本機把 CI 會做的檢查先跑過一遍。

## 證據

`CONTRIBUTING.md`（Modifying Protocol Buffers、Testing and Verification）、`Makefile`、`docs/BACKEND_README.md`、`.github/workflows/check-generate.yml`、`CODEOWNERS`、`flyteidl2/gen_utils/`。

<!-- nav -->
---

上一頁 [05 · 旅程 2：從 issue 到檔案](./from-issue-to-code.md) ｜ [總覽](../README.md) ｜ 下一頁 [05 · 旅程 4：第一個 PR](./first-pr.md)
