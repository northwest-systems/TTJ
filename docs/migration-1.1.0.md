# TTJ 1.1.0 変更点（1.0.0 → 1.1.0）

TTJ `1.0.0` から `1.1.0`（`protocol: "TTJ/1.1.0"`）への変更点を「追加」「削除」「変更」の3区分でまとめる。作業規則の正本は [規格本文](../skills/ttj/references/specification.md) および [JSON参照形式](../skills/ttj/references/json-format.md)、場面別の JSON パターンは [代表ユースケースとサンプルパターン](../skills/ttj/references/use-cases.md) を参照。

## 追加

- `Worker`（`wkr-`）と `Session`（`ssn-`）の分離
- 状態 `paused` と待機情報 `Run.wait`
- 結果フィールド（`Job.result`, `Run.result`）と中止理由 `cancel_reason`
- Task 所属の揮発性 `Handoff`（Task 完了時に消去、恒久知見のみ `Project.knowledge` へ昇格）
- `Ticket.exclusions`（その Ticket でやらないこと・除外事項を明記する任意配列）
- 派生参照 `Ticket.refs` と終了済み旧 Ticket のアーカイブ折りたたみ
- ユースケース 10 件のサンプルパターン集（[skills/ttj/references/use-cases.md](../skills/ttj/references/use-cases.md)）

## 削除

- Job 内の複雑な試行状態管理（`Run` に分離）
- Task 完了後も残る恒久 Handoff

## 変更

- プロトコル表記：`"TTJ/1.0.0"` → `"TTJ/1.1.0"`
- ID 形式の統一：重複を避けるため自動採番とし、全7種（`pj-`, `tsk-`, `tkt-`, `job-`, `run-`, `wkr-`, `ssn-`）を `<接頭辞>-<英数字6桁または8桁>-<インクリメント（ゼロ埋めなし）>` 形式に統一
- 配列キーへの統一：単数／複数によるキー名変化を廃止し、空 `[]`・1件 `[ {...} ]`・複数件 `[ {...}, {...} ]` すべてを許容する配列キー（`projects`, `workers`, `sessions`, `tasks`, `tickets`, `jobs`, `runs` 等）に統一
- 単一 Owner 原則（Project / Task の状態の最終確定者を単一 Owner に固定）
- Job の単純化（進行状態 `status` と結果 `result` の分離）
- Run の正本化と `Job` 内側への配置：`Run` を `Job` の内側（`jobs[].runs[]`）に配置して親 ID（`ticket_id`, `job`, `worker_id`, `session_id`, `wait.job_id`）の重複記載を廃止し、`started` と終了確定時の `result` / `ended` および `wait` を持つ実行試行の正本へ変更
- 再開不可・差し戻し時のルール（既存 Ticket のセッション持ち主や確定済み Job 列を書き換えず、`refs`・`depends` 付きの新規 Ticket を作り直す方式に統一）
