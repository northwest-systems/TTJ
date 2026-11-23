# TTJ JSON参照形式 v1.1.0

- [定義] 以下は項目ごとの記入方法と例であり、ラベルはその行だけに適用する。
- [必須] 作業規則は [規格本文](specification.md) に従う。
- [必須] `protocol: string`：独立した JSON 文書のルートに `"TTJ/1.1.0"` を記載する。
- [定義] `protocol` の値は規格名と版を1項目で識別する。
- [定義] 同じ文書の子レコードは同じ版として扱い、各レコードへの転記は不要。
- [任意] `protocol` は生成側で補完し、以下の項目一覧では共通の記載を繰り返さない。
- [任意] 各レコードを別ファイル・別形式で保存する。
- [例] ID・名称・時刻・成果物参照はすべて架空の値。
- [定義] `objective` は Task・Ticket・Job すべてで「何を達成したいか」を表す。
- [定義] `criteria` は Task・Job の完了を確認する条件の一覧。
- [定義] Task・Ticket・Job の `status` は `todo`・`doing`・`waiting`・`paused`・`blocked`・`completed`・`canceled` の7値。
- [必須] 新規作成時の作業状態は、未指定でも `todo`。
- [定義] 任意の一覧を省略した新規レコードは、その一覧を空として扱う。
- [必須] 更新で省略した項目は既存値を維持し、空にする場合は明示する。
- [定義] 既存の参照名 `project`・`task`・`job`・`target` を維持しつつ、新規の ID 参照名は `owner_id`・`worker_id`・`task_id`・`ticket_id`・`session_id`・`active_run_id`・`run_seq`・`current_ticket_id`・`active_session_id`・`current_job_id`・`source_revision` に統一する。
- [推奨] フィールド名はできるだけ1単語とし、意味を明確にする連結名を規格全体で統一して使う。

## 1. Worker

- [必須] `id: string`：Worker を一意に識別する不変の識別値。
- [必須] `name: string`：変更可能な表示名。
- [禁止] `owner_type` や `human`・`bot`・`agent`・`ai`・`service` 等の種別分類を必須項目として追加する。
- [定義] 認証情報や実行環境設定は外部 Adapter 側で管理する。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "worker-alpha",
  "name": "Alpha"
}
```

## 2. Session

- [必須] `id: string`：Session を一意に識別する値。
- [必須] `worker_id: string`：この Session が属する `Worker.id`。
- [任意] `started: string`：開始時刻（タイムゾーン付き ISO 8601）。
- [任意] `ended: string`：終了時刻（終了済みの場合）。
- [定義] 同じ Worker が複数 Session を持て、各 Session を個別に識別する。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "session-alpha-01",
  "worker_id": "worker-alpha",
  "started": "2026-01-01T01:00:00Z"
}
```

## 3. Project

- [必須] `id: string`：Project を一意に参照する識別値。
- [必須] `name: string`：何の領域か分かる名前。
- [推奨] ID にも対象を判別できる短い識別値を使う。
- [任意] `knowledge: array<object>`：長期知識がある場合だけ記載する。
- [任意] リポジトリ等の追加情報は必要な場合だけ保持する。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "study-meetup",
  "name": "勉強会の運営"
}
```

## 4. Task

- [必須] `id: string`：Task の識別値。
- [必須] `project: string`：所属する Project の ID。
- [必須] `objective: string`：長寿命の成果目標。
- [必須] `criteria: array<string>`：Task 全体の完了を確認する条件を1件以上書く。
- [任意] `status: string`：作業状態（`todo`・`doing`・`waiting`・`paused`・`blocked`・`completed`・`canceled`。新規作成時の未指定は `todo`）。Run を持たない Task の計画内待機にも `waiting` を用いる。
- [任意] `resume: string`：永続正本（`depends`・`Run.wait`・管理履歴）から再開・解除条件が一意に分からない場合に1文で記載する（揮発性 `Handoff` への記載を理由に省略しない）。
- [任意] `deadline: string`：期限がある Task だけに完了期限を書く。
- [必須] `deadline` を記載する場合は、タイムゾーン付きの ISO 8601 日時にする。
- [定義] `deadline` を省略した Task は期限未設定として扱う。
- [禁止] 期限を Ticket・Job へ重複記入させる。
- [定義] 期限を過ぎても、それだけで作業状態は変更しない。
- [定義] 修正・再実行により Ticket が増えても、同一 Task 配下で管理する。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "prepare-invitation",
  "project": "study-meetup",
  "objective": "勉強会の案内を用意する",
  "criteria": [
    "日時・会場・申込方法が正しい案内文が確定している"
  ],
  "deadline": "2026-01-08T18:00:00+09:00"
}
```

## 5. Ticket

- [必須] `id: string`：Ticket の識別値。
- [必須] `task: string`：所属する Task の ID。
- [必須] `owner_id: string`：存在する単一の `Worker.id`。確定後の変更は禁止。
- [必須] `objective: string`：達成したい目標。
- [必須] `jobs: array<string>`：Job の ID を実行順に1件以上並べる。確定後は固定。
- [任意] `status: string`：作業状態（`todo`・`doing`・`waiting`・`paused`・`blocked`・`completed`・`canceled`。新規作成時の未指定は `todo`）。
- [任意] `depends: array<string>`：実行依存（先に `completed` になる必要がある前提 Ticket の ID 一覧）。循環依存は禁止。
- [任意] `refs: array<object>`：情報共有用の参照一覧（実行依存 `depends` とは明確に区別する）。
- [任意] `artifacts: array<string>`：成果物・証拠の参照一覧。
- [必須] `cancel_reason: string`（条件付き必須）：Ticket を `status: "canceled"` にする場合は中止理由の記録を必須とする（非 `canceled` 時は省略可）。
- [任意] `resume: string`：永続正本（`depends`・`Run.wait`・管理履歴）から再開・障害解除条件が一意に分からない場合に1文で記載する（揮発性 `Handoff` への記載を理由に省略しない）。
- [禁止] 旧 `assignee`、複数 Owner、Ticket 独自の `criteria`、旧 `Ticket.handoffs` を使用する。
- [必須] Ticket と全 Job の定義は正式作成時に一括確定する。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "create-invitation",
  "task": "prepare-invitation",
  "owner_id": "worker-alpha",
  "objective": "案内文の初稿を作成し確認準備を整える",
  "jobs": [
    "write-invitation",
    "self-check-invitation"
  ]
}
```

## 6. Job と status / result 分離

- [必須] `id: string`：Job の識別値。
- [必須] `objective: string`：達成したい目標。
- [必須] `criteria: array<string>`：Job の完了可否を確認可能な証拠で判断できる条件を1件以上書く。
- [任意] `instructions: string`：目標だけでは手順が伝わらない場合だけ補う（v1.0.0 の項目名を維持）。
- [任意] `status: string`：進行状態（`todo`・`doing`・`waiting`・`paused`・`blocked`・`completed`・`canceled`。未指定時は `todo`）。
- [任意] `result: string`：`status` と分離された確認結果・業務判定（例：`passed`・`approved`・`changes_requested`・`failed`・`interrupted`・`error` 等）。
- [任意] `artifacts: array<string>`：成果物・確認証拠の参照一覧。
- [任意] `resume: string`：永続正本（`Run.wait`・管理履歴）から再開・障害解除条件が一意に分からない場合に1文で記載する（揮発性 `Handoff` への記載を理由に省略しない）。
- [定義] 親と順序は Ticket の `jobs` から分かるため、再記入しない。
- [禁止] Job ごとの個別 Owner、複雑な分岐・子 Job・待機制御詳細、独立 `Request` オブジェクト、種別分類を追加する。
- [例] 次は作成時の通常 Job。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "write-invitation",
  "objective": "必要事項を揃えた案内文を作る",
  "instructions": "前回の開催要領と最新の会場情報を照合して下書きを作成する。",
  "criteria": [
    "日時・会場・申込方法が記載された案内文ドラフトが保存されている"
  ]
}
```

- [例] 次は別 Worker（`worker-beta`）が担当するレビュー Ticket 内のレビュー Job が、差し戻し判定で完了した例。`status`（完了）と `result`（差し戻し）を分離して表現する。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "review-invitation-job",
  "objective": "案内文の記載内容を確認しレビュー結果を返す",
  "criteria": [
    "元の開催情報との照合結果と判定が記録されている"
  ],
  "status": "completed",
  "result": "changes_requested",
  "artifacts": [
    "review-notes/create-invitation-r1.md"
  ]
}
```

- [例] 次はテスト実行 Job が実行自体を完了し、テスト結果として失敗を記録した例。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "run-release-checks",
  "objective": "公開前検証を実行し結果を記録する",
  "criteria": [
    "全検証コマンドの実行ログと合否判定が保存されている"
  ],
  "status": "completed",
  "result": "failed",
  "artifacts": [
    "logs/release-checks-01.log"
  ]
}
```

## 7. 終了判定・中止と新 Ticket による再実行

- [必須] Job は確認可能な証拠に基づき `criteria` をすべて満たしたら `completed`、中止を判断したら `canceled` にする。Job を単独で `canceled` にする場合は、中止確定と同時に結合中 Run の試行権限を即座に無効化・閉鎖し（`active_run_id = null`, `run_seq + 1`）、停止要求を発行する。
- [必須] 後続 Job を開始するには、先行 Job がすべて `completed` または `canceled` であることに加え、[規格本文](specification.md) 第6節・第8.2節の開始条件をすべて満たす必要がある：前提 Ticket が `completed`、対象 Ticket および現在 Job が `todo` または `doing`（`paused`・`blocked`・`waiting`・終端でないこと）、親 Task が `paused`・終端でないこと（親 Task が別 Ticket 起因で `waiting`・`blocked` であっても独立した実行可能 Ticket・Job は開始できる）、有効な `active` `Execution Claim` の保持、実行系列前提（`expected_run_seq`）の一致、および先行 Job に対する停止確認または安全条件の確認。たとえば先行 Job を `canceled` にしても、停止未確認・安全未確認のまま、または対象 Ticket が `paused` 中であれば後続 Job は開始できない。
- [必須] Ticket は全 Job が `completed` または `canceled` になれば `completed` にする。
- [必須] Ticket 自身がキャンセル済みなら `canceled` を維持し、中止理由（`cancel_reason`）を必須保持する。
- [定義] 次 Ticket が未作成でも、終了済み Ticket はアーカイブできる。
- [例] 次は複合スクリプトの異常終了により Ticket を `canceled` にした抜粋。

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "deploy-staging-01",
    "task": "prepare-invitation",
    "owner_id": "worker-alpha",
    "status": "canceled",
    "cancel_reason": "ステージング反映スクリプトがタイムアウトし環境整合を確認できなかったため中止。復旧確認後に新Ticketで再実行する。",
    "artifacts": [
      "logs/deploy-staging-01.log"
    ]
  },
  "jobs": [
    {
      "id": "prepare-package",
      "status": "completed",
      "result": "passed"
    },
    {
      "id": "apply-staging",
      "status": "canceled",
      "result": "error"
    }
  ]
}
```

## 8. 実行依存（depends）と情報共有参照（refs）

- [任意] `depends: array<string>` に、先に `completed` になる必要がある前提 Ticket の ID を指定する。
- [必須] `depends` に循環依存・自己依存・重複参照を含めない（表示状態が `waiting`・`paused`・`blocked`・停止/移譲中のいずれであっても、非終端 Ticket・Job が保持する未解決 `Run.wait` と混在する循環も禁止。解消済み・解除済みの `Run.wait` は履歴として残しつつ依存グラフの辺からは除外する）。
- [任意] `refs: array<object>` に、前 Ticket の成果物・結果・中止理由、派生元・修正対象・再実行元などを情報共有専用として記録する。
- [必須] `refs` の各要素は `ticket_id: string` を持ち、任意で `kind: string`（例：`derived_from`・`fixes`・`rerun_of`・`related`）と `note: string` を付けられる。
- [禁止] `refs` を実行依存や状態遷移のゲート条件として扱う。
- [例] 次は差し戻し結果と中止理由を参照して修正・再実行する新 Ticket の例。`depends` は前提 Ticket の完了待ち、`refs` は情報共有参照として分離している。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "revise-invitation-02",
  "task": "prepare-invitation",
  "owner_id": "worker-alpha",
  "objective": "レビュー指摘を反映して案内文を修正する",
  "jobs": [
    "apply-review-fixes",
    "verify-invitation-links"
  ],
  "depends": [
    "review-invitation-01"
  ],
  "refs": [
    {
      "ticket_id": "create-invitation",
      "kind": "fixes",
      "note": "初稿ドラフト成果物を引き継ぐ"
    },
    {
      "ticket_id": "review-invitation-01",
      "kind": "derived_from",
      "note": "changes_requested の指摘メモを参照"
    }
  ],
  "status": "todo"
}
```

- [任意] `resume: string` は、Task・Ticket・Job において、永続正本（`depends`・`Run.wait`・管理履歴）だけでは再開・障害解除条件が一意に分からない場合に1文で記載する（揮発性 `Handoff` のみに再開条件を残すことは禁止）。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "publish-invitation",
  "status": "waiting",
  "resume": "会場管理者から最終使用許可の回答を受け取る"
}
```

## 9. Run（実行試行と待機制御）

- [定義] `Run` は Job の1回の実行試行を記録する正本。
- [必須] `id: string`：実行試行を識別する ID。
- [必須] `ticket_id: string`：所属する Ticket の ID。
- [必須] `job: string`：実行した Job の ID。
- [必須] `owner_id: string`：当該 Ticket の `Worker.id`。
- [必須] `session_id: string`：実行を担当する単一の `Session.id`。
- [必須] `generation: integer`：`Execution Claim` のフェンシング世代番号（正の整数）。
- [必須] `started: string`（条件付き必須）：実作業の開始を確定した Run では開始時刻（ISO 8601）を必須記録する。起動前予約（`activity: "unstarted"`）のまま、または実作業を開始せずに取り消して `stopped` 等で終了閉鎖した Run には開始事実がないため、`started` を省略できる。
- [任意] `ended: string`：終了時刻（ISO 8601。終了確定時に時刻を確認できる場合は記録する）。
- [必須] `result: string`（条件付き必須）：試行の結果（例：`succeeded`・`failed`・`interrupted`・`canceled`・`error` 等）。実行中は省略可、**Run の終了確定時は必須**とする。
- [任意] `activity: string`：実行観測状態（`unstarted`・`alive`・`stopped`・`unknown`）。
- [任意] `observed: string`：最終観測時刻（ISO 8601）。
- [任意] `wait: object`：`waiting` 中の待機・実行制御情報（`job_id`・`target`・`condition`・`since`・`resume_point`・`timeout`・待機解消/解除時の `resolved`）。Job 側ではなく Run 側で保持する。
- [例] 次は複合スクリプト実行中の Job が外部操作の結果を `waiting` で待機している Run の例（初回取得済み Claim の `generation: 2`）。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "run-staging-apply-01",
  "ticket_id": "deploy-staging-01",
  "job": "apply-staging",
  "owner_id": "worker-alpha",
  "session_id": "session-alpha-01",
  "generation": 2,
  "started": "2026-01-01T01:00:00Z",
  "activity": "alive",
  "observed": "2026-01-01T01:05:00Z",
  "wait": {
    "job_id": "apply-staging",
    "target": "external-approval:staging-slot-7",
    "condition": "メンテナンス枠の開始承認を受け取る",
    "since": "2026-01-01T01:04:30Z",
    "resume_point": "step-3-apply-migration",
    "timeout": "2026-01-01T02:00:00Z"
  }
}
```

- [例] 次は停止確認済みの終了 Run（終了確定時に `result` を必須記録し、この時点で `run-write-invitation-01` の試行権限は閉じる）。Heartbeat 途絶だけでは `unknown` であり、独立監視等による停止確認後にのみ `stopped` とする。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "run-write-invitation-01",
  "ticket_id": "create-invitation",
  "job": "write-invitation",
  "owner_id": "worker-alpha",
  "session_id": "session-alpha-01",
  "generation": 2,
  "started": "2026-01-01T01:00:00Z",
  "ended": "2026-01-01T01:20:00Z",
  "activity": "stopped",
  "observed": "2026-01-01T01:20:00Z",
  "result": "succeeded"
}
```

- [例] 次は起動前予約（`unstarted`）を実作業開始前に取り消して閉鎖した Run（`unstarted → stopped`）。実作業を開始していないため `started` を持たず、終了 `result` のみを記録する。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "run-staging-apply-00",
  "ticket_id": "deploy-staging-01",
  "job": "apply-staging",
  "owner_id": "worker-alpha",
  "session_id": "session-alpha-01",
  "generation": 2,
  "ended": "2026-01-01T00:55:00Z",
  "activity": "stopped",
  "observed": "2026-01-01T00:55:00Z",
  "result": "canceled"
}
```

## 10. Execution Claim（実行権の正本）

- [定義] `Execution Claim` は Ticket 単位（一意キー：`ticket_id`）の実行権の正本であり、`generation` によるフェンシング、現行試行 `active_run_id` の結合、および実行系列版 `run_seq` を持つ。
- [必須] `task_id: string`：所属 Task の ID。
- [必須] `ticket_id: string`：対象 Ticket の ID（Claim の一意キー）。
- [必須] `owner_id: string`：当該 Ticket の `Worker.id`。
- [必須] `session_id: string | null`：現在実行権を保持する単一の生存する適格 `Session.id`（`state` が `active` 以外のときは必ず `null`）。
- [必須] `active_run_id: string | null`：現在有効な単一試行の `Run.id`（未実行・試行終了・`state` が `active` 以外のときは必ず `null`）。
- [必須] `generation: integer`：初期値 `1`（`unassigned` 作成時）から、状態切替（取得・再取得・移譲・解放・失効）のたびに厳密に `+1` 単調増加する正の整数（確定済み世代は巻き戻さない）。
- [必須] `run_seq: integer`：初期値 `0` から、Run の開始結合・即時無効化・安全閉鎖・終了確定のたびに単調に `+1` 増加する非負整数（巻き取り切替および並行再試行の古い前提検出に用いる）。
- [必須] `state: string`：実行権状態（`unassigned`・`active`・`released`・`revoked`）。
- [任意] `acquired: string`：取得時刻（ISO 8601）。
- [任意] `revoked: string`：失効時刻（ISO 8601）。
- [任意] `takeover: object`：単一の原子的確定による新 Ticket 巻き取りの確定済み結果記録（同一操作再送の冪等返却および主体・パラメータ照合に用いる：`operation_id`・`claimant_session_id`・`target_owner_id`・`successor_ticket_id`）。未確定の途中状態は公開しない。
- [例] 次は `unassigned`（`generation: 1`, `run_seq: 0`）から初回取得し、`run-write-invitation-01` を実行中の `Execution Claim`（`generation: 2`, `run_seq: 1`）。

```json
{
  "protocol": "TTJ/1.1.0",
  "task_id": "prepare-invitation",
  "ticket_id": "create-invitation",
  "owner_id": "worker-alpha",
  "session_id": "session-alpha-01",
  "active_run_id": "run-write-invitation-01",
  "generation": 2,
  "run_seq": 1,
  "state": "active",
  "acquired": "2026-01-01T01:00:00Z"
}
```

- [例] 次は停止確認および安全確認後に、単一の原子的確定による新 Ticket 巻き取りで旧実行権を失効・確定させた状態（`generation: 2 → 3`、`session_id: null`、`active_run_id: null`）。以降の `generation: 2` からの更新および旧 Ticket での再取得は拒否される。

```json
{
  "protocol": "TTJ/1.1.0",
  "task_id": "prepare-invitation",
  "ticket_id": "deploy-staging-01",
  "owner_id": "worker-alpha",
  "session_id": null,
  "active_run_id": null,
  "generation": 3,
  "run_seq": 2,
  "state": "revoked",
  "revoked": "2026-01-01T01:40:00Z",
  "takeover": {
    "operation_id": "op-takeover-staging-01",
    "claimant_session_id": "session-beta-01",
    "target_owner_id": "worker-beta",
    "successor_ticket_id": "deploy-staging-02"
  }
}
```

## 11. Handoff（Task 所属の揮発性単一スロット）

- [定義] `Handoff` は `Task` ごとに最大1つ持つ揮発性の引き継ぎ・状況スロット。
- [必須] `task_id: string`：所属 Task の ID。
- [必須] `revision: integer`：揮発スロット側の書込連番（同一スロット・同一 `source_revision` 内で単調増加する正の整数）。
- [必須] `source_revision: integer`：揮発スロットの消失をまたいで鮮度を比較できる永続正本側の単調確定版（正の整数）。さらに同一 `source_revision` であっても、スロット消失・再生成前に準備された旧書込みは新スロットを上書きできない（[規格本文](specification.md) 第9.1節）。
- [任意] `current_ticket_id: string | null`：現在対象または直近の Ticket ID。
- [任意] `owner_id: string | null`：対象 Ticket の `Worker.id`。
- [任意] `active_session_id: string | null`：対象 Ticket の実行権を持つ、または直近の `Session.id`。
- [任意] `current_job_id: string | null`：現在 Job の ID。
- [任意] `source_generation: integer`：表示対象 `current_ticket_id` の `Execution Claim` 世代。
- [任意] `activity: string`：実行観測の要約（`unstarted`・`alive`・`stopped`・`unknown`）。
- [任意] `context: string`：現在の作業文脈・申し送り・次の操作（再開に不可欠な条件は正本にも必ず残す）。
- [任意] `stop_position: string`：停止位置の要約。
- [任意] `blockers: array<string>`：現在把握しているブロッカー一覧。
- [任意] `notes: array<string>`：補足情報。
- [任意] `updated: string`：最終更新時刻（ISO 8601）。
- [禁止] Handoff を正式履歴・成果物・再開条件の正本とする、閲覧権限を実行権と同一視する、Handoff にコマンド実行機能を持たせる。
- [禁止] Handoff が単一スロットであることを理由に、Task 内の無関係な Ticket を直列化する。

```json
{
  "protocol": "TTJ/1.1.0",
  "task_id": "prepare-invitation",
  "revision": 5,
  "source_revision": 12,
  "current_ticket_id": "revise-invitation-02",
  "owner_id": "worker-alpha",
  "active_session_id": "session-alpha-02",
  "current_job_id": "apply-review-fixes",
  "source_generation": 2,
  "activity": "alive",
  "context": "レビュー指摘3件のうち日時表記と申込URLを修正済み。残り1件の会場アクセス表記を反映中。",
  "stop_position": "会場アクセス節の照合前",
  "blockers": [],
  "notes": [
    "最新ドラフトは drafts/invitation-v2.md に保存済み。"
  ],
  "updated": "2026-01-01T03:10:00Z"
}
```

## 12. 長期知識（Project.knowledge）

- [任意] Project の `knowledge` に再利用する知見を残す。
- [必須] 記録する場合は `content: string` に残す内容、`basis: string` に根拠・判断理由・出典をまとめる。
- [必須] 元の Task・Ticket がなくても内容と判断経緯を理解できる。
- [必須] 知識は `id` で識別し、採用状態と変更履歴を保持する。
- [定義] 知識の `status` は `active`・`supersed` の2値（v1.0.0 の綴りを維持）。
- [必須] 新規記録は未指定でも `active` とし、通常は `active` だけを参照する。
- [定義] 将来の予定も、その予定を現在の知識として保持するため `active`。
- [必須] 採用中の旧知識を置換する場合は `supersedes` に同じ Project の旧知識の ID を書く。
- [必須] 置換時は旧記録の `supersed` と新記録の `active` を同時確定する。
- [例] 次は過去の記録、現在使う記録、現在の予定を表した例。

```json
{
  "protocol": "TTJ/1.1.0",
  "knowledge": [
    {
      "id": "invitation-short-form",
      "status": "supersed",
      "content": "申込先を中心に短い案内を作る",
      "basis": "初回の案内作成時に、読む量を減らす方針として採用した。"
    },
    {
      "id": "invitation-required-details",
      "status": "active",
      "content": "案内には日時・会場・申込方法をすべて記載する",
      "basis": "初回案内後に日時と会場の質問が相次いだため、必要情報を案内だけで確認できる方針へ変更した。",
      "supersedes": "invitation-short-form"
    },
    {
      "id": "invitation-automation-plan",
      "status": "active",
      "content": "次回は案内配信の自動化を検討する予定。",
      "basis": "準備作業を減らすため、次回の検討事項として記録する。効果と運用負担は今後確認する。"
    }
  ]
}
```

## 13. 管理記録（監査イベント履歴）

- [必須] 現在状態と別に、変更前後・主体・時刻・理由・確定世代・関連実行を追跡する。
- [定義] `id`・`operation` はイベントと一括操作、`target`・`event` は対象の ID と変更の種類。
- [定義] `from`・`to`・`worker_id`・`session_id`・`timestamp`・`reason`・`revision` は変更前後・実行主体・時刻・理由・レコード世代（管理・監視・証拠取り込み操作では当該認可主体を識別する）。
- [定義] `run`・`generation` は関連する Run と `Execution Claim` のフェンシング世代。
- [任意] 管理情報は更新手段で補完し、同等の履歴形式を使う。
- [禁止] 補完できない事実を捏造する、または各階層へ手入力させる。

```json
{
  "protocol": "TTJ/1.1.0",
  "id": "evt-write-invitation-started",
  "operation": "start-write-invitation",
  "target": "write-invitation",
  "event": "status.changed",
  "from": "todo",
  "to": "doing",
  "worker_id": "worker-alpha",
  "session_id": "session-alpha-01",
  "timestamp": "2026-01-01T01:00:00Z",
  "reason": "開始条件と Execution Claim（generation=2）を確認した",
  "revision": 2,
  "run": "run-write-invitation-01",
  "generation": 2
}
```
