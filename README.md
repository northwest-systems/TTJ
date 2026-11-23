# TTJ

**Task–Ticket–Job 管理規格 · v1.1.0**

- [定義] **目的**：担当や環境が変わっても、作業・申し送り・知識を安全に継続する。
  - [必須] Worker 種別（人間・Bot・AI 等）で業務ロジックや状態遷移を分岐させず、特定 Agent・Bot・オーケストレーターへ依存しない。
  - [必須] 必要な情報を一度だけ書き、不要な分類や状態機械を増やさない。

```text
Worker（責任主体：id: wkr-* / name）── Session（実行単位：id: ssn-* / worker_id）
                                            │
Project（id: pj-* / 名前を持つ継続領域・長期知識 knowledge）
└── Task（id: tsk-* / 長寿命の成果目標・criteria・揮発性 Handoff 単一スロット）
    ├── Ticket A（id: tkt-* / worker_id: wkr-* / session_id: ssn-*）─┐
    ├── Ticket B（id: tkt-* / worker_id: wkr-* / session_id: ssn-*）─┴─→ Ticket C（depends で完了待ち / refs で情報共有参照）
    └── Ticket（担当セッション固定・確定後は固定の Job 列・一方向進行）
        └── Job 1 → Job 2 → …（id: job-* / status と result を分離）
            └── Run（id: run-* / 実行試行の正本・waiting 時の待機制御）
```

- [必須] **規格の識別**：独立した JSON 文書のルートに `protocol: "TTJ/1.1.0"` を持つ。
  - [定義] 子レコードは同じ版を共有し、転記は不要。
- [必須] **ID 接頭辞**：Project・Task・Ticket・Job・Run・Worker・Session の `id` は、それぞれ `pj-`・`tsk-`・`tkt-`・`job-`・`run-`・`wkr-`・`ssn-` の接頭辞で始める。
- [必須] **Worker と Session**：作業の責任主体（`Worker`：`id`（接頭辞 `wkr-`）・`name`）と実行単位（`Session`：`id`（接頭辞 `ssn-`）・`worker_id`）を分離する。
  - [禁止] `owner_type` の必須化や、Worker 種別による状態遷移の分岐。
  - [必須] 単一 Owner 原則により Project / Task の状態の最終確定者を単一 Owner に固定する。
- [必須] **Project**：`id`（接頭辞 `pj-`）と意味の分かる `name` を持つ。
  - [任意] リポジトリを伴わない作業にも使う。
- [定義] **目標**：`objective` は Task・Ticket・Job で同じ意味の「達成したいこと」。
- [必須] **Task**：`id`（接頭辞 `tsk-`）・`project`・`objective`・`criteria` を持つ長寿命の成果目標。
  - [任意] 期限がある場合だけ `deadline` をタイムゾーン付き ISO 8601 日時で記載する。
  - [禁止] 修正・再実行による Ticket 増加で Task を増殖させる、または完了 Ticket 数のみで達成率を決める。
- [必須] **Ticket**：`id`（接頭辞 `tkt-`）・`task`・`worker_id`・`session_id`・`objective`・`jobs` を持つ、単一 Worker・単一 Session 担当・一方向進行の作業単位。
  - [禁止] 複数担当、Job ごとの個別担当、同一 Ticket の `worker_id`・`session_id` 変更、Ticket 独自の `criteria`、確定後の Job 列変更、終端 Ticket の再開。
  - [必須] 担当 Session が再開不可になった場合は、旧 Ticket を中止（`canceled` と `cancel_reason` を記録）して閉じ、同一 Task 配下に `refs` 付きの新規 Ticket を起票して新しい Session（`ssn-`）で稼働させる。
  - [必須] 全 Job が `completed` または `canceled` になれば `completed` に集約し、Ticket 自身がキャンセル済みなら `canceled` と `cancel_reason` を維持する。
  - [定義] 次 Ticket が未作成でも終了済み Ticket をアーカイブできる。
- [必須] **Job**：`id`（接頭辞 `job-`）・`objective`・`criteria`（任意で `instructions`）を持つ最小実行単位。
  - [必須] 進行状態 `status` と業務結果 `result`（`approved`・`changes_requested`・`passed`・`failed` 等）を分離する。
  - [禁止] 独立した `Request` オブジェクト、子 Job、複雑なワークフロー構造、種別ごとの Job 分類。
- [任意] **実行依存と情報共有参照**：実行依存 `depends`（循環禁止）と、成果物・中止理由・派生関係等を共有する `refs` を明確に区別する。
- [定義] **Run**：`Run`（`id` 接頭辞 `run-`）は実行試行の正本として `worker_id`・`session_id`・開始時刻（`started`）・待機制御（`wait`）・試行結果（`result`）を保持する（実行中は `result`・`ended` が無く、終了確定時に `result` と確認できれば `ended` を記録する）。
  - [必須] 同一 Ticket・同一 Session 内での Job 再試行は、失敗した旧 Run を閉じて新しい Run を追加・実行する。
  - [必須] 前の実行が停止（または残存副作用がない安全条件）していることを確認した上で旧 Run と旧 Ticket を閉鎖した場合のみ、新規 Ticket への引き継ぎを許可する。
- [定義] **揮発性 Handoff と長期知識**：短期の引き継ぎは Task 所属の揮発性単一スロット `Handoff`（Task 完了時に消去）、長期知識は Project の `knowledge`（`active`・`supersed`）に分ける。
  - [必須] Handoff 消失でも正式履歴を失わず、無関係な Ticket の並列実行を遮らない。
- [定義] **[規格本文](skills/ttj/references/specification.md)**：TTJ v1.1.0 作業規則の正本。
- [定義] **[JSON参照形式](skills/ttj/references/json-format.md)**：項目ごとの記入方法と例。
- [定義] **[代表ユースケースとサンプルパターン（UC-01〜UC-10）](skills/ttj/references/use-cases.md)**：代表的な10場面における状態遷移と JSON サンプルパターン。
- [定義] **[TTJ Skill](skills/ttj/SKILL.md)**：LLM が規格を適用する手順。
  - [任意] `skills/ttj` 全体を利用環境の Skill 配置先へコピーする。
  - [必須] コピーする場合は参照文書も含める。
  - [禁止] Skill 単独で永続化・排他・外部副作用防止を保証したと称する。
- [定義] **[v1.1.0 変更点](docs/migration-1.1.0.md)**：TTJ 1.0.0 から 1.1.0 への追加・削除・変更点の一覧。
- [定義] **ライセンス**：[Apache-2.0](LICENSE)。
