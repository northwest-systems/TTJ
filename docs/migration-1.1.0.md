# TTJ 1.1.0 仕様変更概要・データモデル差分・状態遷移

本文書は、TTJ `1.0.0` から `1.1.0`（`protocol: "TTJ/1.1.0"`）への仕様改定における変更概要（提出物 A）、データモデル差分（提出物 C）、および状態遷移の整理（提出物 D）をまとめたものである。規範的な作業規則は [規格本文](../skills/ttj/references/specification.md) および [JSON参照形式](../skills/ttj/references/json-format.md) を正本とする。

---

## A. 仕様変更概要（1.0.0 → 1.1.0）

### A.1 変更の背景と目的

TTJ `1.0.0` は最小限の記入で作業と知識を継続できる一方、複数 Session や自動化基盤と併用する運用において以下の課題が顕在化していた。

1. **責任主体と実行単位の未分離**：`assignee` に人・LLM・LLM セッションが混在でき、同一 Worker の複数 Session 間の実行権境界や無断実行の拒否条件が曖昧だった。
2. **共同作業・Ticket 内担当交代の複雑化**：複数担当による同一 Job 実行や Ticket 途中での `assignee` 変更を許容すると、所有責任・実行権失効・履歴境界が複雑になる。
3. **進行状態と業務判定の混同**：レビュー差し戻しやテスト失敗など「Job 自体は完了したが業務結果が否定的」なケースを `status` と `result` に明確分離する規則が弱かった。
4. **実行権・現行試行結合・生存判定・巻き取り規則の不足**：Heartbeat 途絶と停止確認済みの区別、同時巻き取りおよび途中停止クラッシュ時の決定性、同世代の閉じた旧 Run や復帰した旧 Session の更新拒否、副作用防止のための外部フェンシングを `Execution Claim` と `generation` フェンシングで厳密に保証する必要があった。
5. **Handoff の責務肥大化**：`Ticket.handoffs` が短期申し送り・Ticket 間参照・課題記録を兼ねていたため、Task 単位の揮発性状況スロットと、永続的な Ticket 参照（`refs`）・中止理由（`cancel_reason`）・再開条件（`resume` / `Run.wait` / 管理履歴）を分離する必要があった。

### A.2 追加・変更・廃止した要件一覧

| 区分 | 項目 | TTJ 1.0.0 | TTJ 1.1.0 | 変更理由 |
|---|---|---|---|---|
| 変更 | プロトコル版 | `protocol: "TTJ/1.0.0"` | `protocol: "TTJ/1.1.0"` | 非互換の変更（`assignee`・`Ticket.handoffs` 廃止、`Worker`/`Session`/`Execution Claim` 追加）を含むマイナー改定（合意済み 1.1.0 方針） |
| 変更 | フィールド命名方針 | 1単語推奨 | 1単語推奨＋意味のある連結名（`owner_id`, `worker_id`, `task_id`, `ticket_id`, `session_id`, `active_run_id`, `current_ticket_id`, `active_session_id`, `current_job_id`, `cancel_reason`）を統一採用。既存 `project`/`task`/`job`/`target` は維持 | 参照先と識別子の意味を曖昧なく表現し、仕様・JSON・SKILL 間の表記揺れを防ぐため |
| 追加 | `Worker` | 未定義（`assignee` の任意識別値） | 正式定義（不変 `id`・変更可能 `name`）。種別分類・`owner_type` 禁止 | 作業の責任主体を一意化しつつ、人間・Bot・AI 非依存を徹底するため |
| 追加 | `Session` | 未定義 | 正式定義（`id`・`worker_id`・`started`・`ended`） | 責任主体（Worker）と実行単位（Session）を分離し、同一 Worker の複数 Session を個別識別するため |
| 廃止/置換 | 担当モデル | `Ticket.assignee`・`Job.assignee`（配列・継承）、共同作業、同一 Ticket の担当交代 | **廃止**。`Ticket.owner_id`（単一・不変の `Worker.id`）に統一。別 Worker 作業・追加依頼は別 Ticket | 責任境界を一元化し、排他・巻き取り・監査を単純明快にするため |
| 変更 | `Job` | `instructions`・`criteria`・`assignee`・任意 `result`・`resume` | `instructions`・`criteria`・`resume` を維持、`assignee` を廃止、`status`（進行）と `result`（業務判定・中断・エラー）を明確分離。独立 `Request`・子 Job・種別分類・待機制御構造を禁止 | Job を機械的・証拠ベースで完了判定できる最小単位に保つため |
| 変更 | `Task` | 目標・`criteria`・`deadline`・`status`（`waiting` 含む）・`resume` | 長寿命目標管理を強化。Task レベルの計画内待機（`waiting`）と `resume` を維持。Ticket 増加で Task を増殖させないこと、完了 Ticket 数のみで達成率を決めないことを明記。揮発性 `Handoff` 単一スロットを所属させる | 修正・再実行による Ticket 増加を自然な運用として受け止めつつ Task 単位で成果を統括するため |
| 変更 | `Ticket` 参照と終了 | `Ticket.handoffs` に元 Ticket ID を手書き | `Ticket.handoffs` を**廃止**。永続的な情報共有参照 `refs`（`depends` と明確区別）および `cancel_reason`（`canceled` 時必須）・`artifacts` を導入。次 Ticket 未作成でもアーカイブ可能と明記 | 揮発性 Handoff と永続履歴・参照を分離し、重複作業と文脈断絶を防ぐため |
| 拡張 | `Run` | `id`・`job`・`actor`（配列）・`started`・`ended`・`result`・`generation`・`activity`・`observed` | 実行試行の正本として拡張：`ticket_id`・`owner_id`・`session_id`・`generation`・`wait`（待機制御情報）を保持。`started`（`unstarted` 予約時除く）と終了時 `result` を条件付き必須化。独立 `Execution`/`Attempt` オブジェクトは導入しない | 不要なオブジェクトや状態機械を増やさず、待機制御・単一 Session 実行・試行権限終了時点を正本化するため |
| 追加 | `Execution Claim` | 概念的言及（Lease 推奨、`generation` 参照） | Ticket 単位（一意キー：`ticket_id`）の実行権正本として正式定義（`task_id`・`ticket_id`・`owner_id`・`session_id`・`active_run_id`・`generation`（初期値 `1`、切替ごと `+1`）・`state`・`takeover`）。外部フェンシングを準拠必須要件とし、原子的または永続復旧可能な巻き取り手順を規定 | 同一作業系列の最大1 Session・最大1有効 Run、同時巻き取り抑止、途中クラッシュ復旧、旧 `generation` 拒否を厳密化するため |
| 変更 | `Handoff` | `Ticket.handoffs`（文字列配列） | **Task 所属の揮発性単一スロット**へ再設計（`revision`・`source_generation` による鮮度/単調性制御）。閲覧権と実行権を厳格分離、再開条件は必ず永続正本へ残し、消失しても再開判断と正式履歴を維持、無関係 Ticket の並列実行を遮らない | 複数 Session からの安全な状況把握と、正式履歴・再開判定からの責務分離を両立するため |
| 追加 | Ticket 増加対策 | 未記載 | テンプレート・標準 Job 列・後続 Ticket 生成支援・成果物/中止理由参照・系列表示・終了 Ticket 折りたたみ・Task 進捗集約・現在 Handoff 表示を準拠必須要件（`[必須]`）として明記（交換形式・UI 実装方法は選択可） | 単一 Owner・一方向 Ticket 運用での起票・管理負荷を最小化するため |
| 維持 | 既存 1.0.0 コア規則 | 4階層・Job 列固定・終端再開禁止・循環 `depends` 禁止・`Task.deadline`・Task レベル `waiting`・`paused` の意味・同一計画の再試行および後続 Job 開始時の「停止または安全条件」・中断/失敗時の記録と根拠のない再試行禁止・`Project.knowledge`（`active`/`supersed`）・特定基盤非依存 | すべて維持（巻き取りのみ原典 1.1.0 要件どおり「停止確認かつ安全条件」の両方必須） | TTJ 1.0.0 の堅牢な基盤原則との一貫性と後方互換の明確さを保つため |

### A.3 既存 TTJ 1.0.0 との矛盾解決および互換規則の確認記録

1. **`Ticket.handoffs` 廃止と永続相互参照・再開条件の正本保存**
   - 1.0.0 では「専用の関係フィールドを禁止し、`Ticket.handoffs` に元 Ticket ID や課題・再開確認を書く」規則だった。
   - 1.1.0 では `Ticket.handoffs` を廃止し、`Handoff` を Task 所属の**揮発性**スロットへ移したため、永続的な関連や再開・障害解除条件を揮発性 Handoff だけに置くと消失・再起動後に再開判定が不能となる矛盾が生じる。
   - **解決**：(a) 永続的な関連参照は情報共有専用の `Ticket.refs` へ分離（`depends` と完全区別し、`refs.kind` による業務分岐禁止）、(b) 再開・再試行・障害解除に不可欠な条件と証拠は必ず永続正本（`depends`・`Run.wait`・`Run.result`・`Ticket.cancel_reason`・対象レコードの `resume`・管理履歴）へ保存し、揮発性 Handoff の記載を理由とした `resume` 省略を禁止した。(c) 1.0.0 の「中断・引き継ぎ前と重要な失敗時の更新」「再開時の実状態照合」「根拠のない失敗再試行の禁止」は永続正本＋Handoff 更新として維持した。
2. **共同作業・担当交代の廃止と複数者レビュー・引き継ぎの扱い**
   - 1.0.0 では 1 つの Job に複数 `assignee` を指定して共同レビューし、同一 Ticket 内で担当交代できた。
   - 1.1.0 では単一不変 `Ticket.owner_id` と Worker/Session 分離を優先し、旧 `assignee`・共同作業・同一 Ticket の Owner 変更を廃止した。
   - **解決**：別 Worker の作業・レビュー・巻き取りは旧 Ticket の `owner_id` を変えず新 Ticket とし（未終端旧 Ticket からの巻き取り移管時は旧 Ticket の `canceled` 終端化と旧 Claim の再取得永久禁止を必須とし、二重実行を構造的に防止）、同一 Worker の別 Session 引き継ぎのみ停止・安全確認＋原子的 `Execution Claim` 更新で整合させた。
3. **1.0.0 規則の維持：同一計画の再試行・後続 Job 開始（「停止 *または* 安全条件」） vs 巻き取り（「停止 *かつ* 安全条件」）**
   - 1.0.0 規格（L244, L246）は、同一計画内の通常再試行および後続 Job の開始について「前の実行の停止 **または** 安全条件」を認めていた（たとえば旧試行の停止通知が取得できない環境でも、外部フェンシング等により旧試行の副作用・更新を確実に無効化できる安全条件が成立していれば開始可能）。
   - **解決**：1.0.0 互換規則を**変更せず維持**する。すなわち、同一 `active` Claim による同一計画内の通常再試行および後続 Job 開始は「停止確認 **または** 副作用重複・旧試行の影響を確実に防げる安全条件」を維持し、一方で別 Session/別 Worker への**巻き取り**については原典 1.1.0 要件（UC-17/18/19/25）どおり「過去 Run の全停止確認（`stopped`）**かつ** 安全条件成立」の両方必須を維持する。
4. **1.0.0 規則の維持：Task レベルの計画内待機（`waiting`）と非実行経路（管理・評価・証拠取り込み）**
   - 1.0.0 規格（L107–122）に存在した Task レベルの `waiting` を復元し、さらに Claim 未発行時の前提待ち評価・権限保持運用者による保留/中止・Run 終了後の検証済み外部証拠取り込みを「管理・評価・証拠取り込み操作」として実行操作（Claim 必須）から明確分離し、双方に認可・終端保護・監査記録を適用した。
5. **`Run` 拡張 vs 独立 `Execution` / `Attempt` オブジェクト**
   - 原典ドラフトでは `Execution` / `Attempt` の語が混在していたが、1.0.0 には既に `Run`（Job の1回の実行試行）が存在した。
   - **解決**：既存 `Run` を実行試行の正本として拡張し、`Execution Claim` に現行試行 `active_run_id` を一意結合することで、不要なオブジェクトや状態機械を増やさずに同世代旧 Run の拒否・並行再試行の排他・試行終了時の権限閉鎖（1.0.0 L247 の継承）を実現した。
6. **`Job` の `instructions` / `criteria` と `Project.knowledge` の `supersed` 綴り**
   - 1.0.0 の `instructions` / `criteria` および `Project.knowledge.status` の正確な綴り `active` / `supersed` を完全維持した。

### A.4 既存実装（TTJCore 等）への影響

- **破壊的変更（スキーマ・バリデーション）**：
  - `protocol` の受理版を `"TTJ/1.1.0"` へ更新。
  - `Worker`・`Session`・`Execution Claim`（一意キー：`ticket_id`、`active_run_id`・`generation` 初期値 `1`・`takeover` 含む）の追加と参照整合性検証（`Ticket.owner_id → Worker.id`、`Session.worker_id → Worker.id`、`Run.session_id → Session.id`）。
  - `Ticket.assignee`・`Job.assignee`・`Ticket.handoffs` の廃止と、`Ticket.owner_id`（必須・不変）・`Ticket.refs`・`Ticket.cancel_reason`（`canceled` 時必須）・`Ticket.artifacts` の追加。
  - `Run.actor: array<string>` から `Run.owner_id` + `Run.session_id` + `Run.ticket_id` + `Run.wait` への変更、`started`（`unstarted` 予約時除く）および終了時 `result` の条件付き必須化。
  - `Handoff` を Ticket 配列から Task 所属単一スロット（`revision`・`source_generation` 付き、再生成・破棄可能）へ移行。
- **排他・実行制御・外部フェンシング層への影響**：
  - すべての実行操作 API で認証済み `Session.worker_id == Ticket.owner_id`、`claim.state == "active"`、`claim.session_id`、`claim.generation`、対象 Job、現行 `claim.active_run_id == run_id` を照合し、1つでも不一致なら拒否する。
  - Run 開始時に `claim.active_run_id` を原子的に設定（並行再試行の排他）し、Run 終了確定時に試行権限を即座に閉じる。
  - 独立監視の停止証拠記録経路（`activity: "stopped"` のみ）と業務状態更新経路を分離し、管理・依存評価・証拠取り込みの認可経路を明示実装する。
  - 巻き取り操作に全過去 Run の `stopped` 確認・安全条件確認・原子的 CAS（`generation +1`）および新 Ticket 移管時の原子的確定または `takeover` 予約によるクラッシュ復旧・冪等再送を実装する。
  - 副作用を防ぐ必要がある操作では、外部ゲートウェイ／効果確定側でも失効済み権限（旧 `generation`・失効済み Claim・閉じた `run_id`）を拒否し、失効前開始済み操作が残存し得る場合は安全確認完了まで後続実行を保留する。

### A.5 残課題（次期検討・実装側決定事項）

1. **安全条件（副作用残存なし）の検証インターフェース**：副作用確認・失効前開始済み外部操作の完了確認・環境復旧完了の具体的証明方式（チェックリスト・外部ヘルスチェック・署名付き確認記録等）は運用・Adapter 実装側で具体化する。
2. **Ticket テンプレートおよび標準 Job 列の交換形式**：必須機能としてのテンプレート・標準 Job 列の JSON 交換スキーマ標準化が必要になった場合、後続マイナー版で任意拡張として検討する。
3. **1.0.0 データの移行ツール**：既存 1.0.0 レコード（複数 `assignee` を含む Ticket や `Ticket.handoffs`）を 1.1.0 へ機械変換する場合の分割・移行ルールは、移行対象ストレージ実装ごとに定義する。

---

## C. データモデル差分

### C.1 エンティティ別 1.0.0 → 1.1.0 差分一覧

| エンティティ | 1.0.0 構成 | 1.1.0 構成（追加・変更・削除） | 役割の整理 |
|---|---|---|---|
| **Worker** | （未定義） | **新設**：`id`（必須・不変）, `name`（必須・変更可）。種別・`owner_type` 禁止 | 作業の責任主体。認証・環境設定は外部 Adapter 側 |
| **Session** | （未定義） | **新設**：`id`（必須）, `worker_id`（必須）, `started`（任意）, `ended`（任意） | 実際の作業実行を識別する単位。同一 Worker が複数保持可 |
| **Project** | `id`, `name`, `knowledge`（`id`, `status`: `active`/`supersed`, `content`, `basis`, `supersedes`） | **変更なし**（`active`/`supersed` 綴り完全維持） | 継続領域と長期知識の正本 |
| **Task** | `id`, `project`, `objective`, `criteria`, `deadline`, `status`, `resume` | **維持＋責務明確化**：長寿命成果目標。`status`（Task レベルの計画内 `waiting` 含む）・`resume` 維持。揮発性単一スロット `Handoff` が所属（別レコード時は `Handoff.task_id` で結合） | 複数 Ticket の統括・結果集約・最終達成判定 |
| **Ticket** | `id`, `task`, `objective`, `jobs`, `assignee`, `depends`, `handoffs`, `status`, `resume` | **追加**：`owner_id`（必須・不変）, `refs`（任意：情報共有参照）, `artifacts`（任意）, `cancel_reason`（**条件付き必須：`canceled` 時必須**）<br>**維持**：`status`, `depends`, `resume`<br>**廃止**：`assignee`, `handoffs` | 単一 Worker 担当・一方向進行の短命作業単位 |
| **Job** | `id`, `objective`, `criteria`, `instructions`, `assignee`, `status`, `result`, `resume` | **維持**：`id`, `objective`, `criteria`, `instructions`, `status`, `resume`<br>**明確化**：`result`（`status` と分離した業務判定・エラー結果）, `artifacts`（任意）<br>**廃止**：`assignee` | `criteria` で完了判定できる最小実行単位。複雑な構造を持たない |
| **Run** | `id`, `job`, `actor[]`, `started`, `ended`, `result`, `generation`, `activity`, `observed` | **追加**：`ticket_id`（必須）, `owner_id`（必須）, `session_id`（必須）, `wait`（任意：`job_id`, `target`, `condition`, `since`, `resume_point`, `timeout`）<br>**廃止**：`actor[]`<br>**明確化**：`generation`（必須・正整数）, `started`（**条件付き必須：`activity: "unstarted"` 予約時のみ省略可、開始確定時必須**）, `result`（**条件付き必須：終了確定時必須**） | Job の1回の実行試行（Execution / Attempt）および待機・観測の正本 |
| **Execution Claim** | （未定義；Lease 推奨のみ） | **新設（一意キー：`ticket_id`）**：`task_id`（必須）, `ticket_id`（必須）, `owner_id`（必須）, `session_id`（必須・`active` 以外は `null`）, `active_run_id`（必須・有効試行なしまたは非 `active` 時は `null`）, `generation`（必須・初期値 `1`、切替1回ごとに `+1`）, `state`（必須：`unassigned`/`active`/`released`/`revoked`）, `acquired`（任意）, `revoked`（任意）, `takeover`（任意：`operation_id`, `claimant_session_id`, `target_owner_id`, `successor_ticket_id`） | Ticket 単位の実行権と現行試行結合の正本（`generation` フェンシング） |
| **Handoff** | `Ticket.handoffs: array<string>` | **全面再設計（Task 所属揮発性単一スロット）**：`task_id`（必須）, `revision`（必須・単調増加整数）, `current_ticket_id`, `owner_id`, `active_session_id`, `current_job_id`, `source_generation`, `activity`, `context`, `stop_position`, `blockers[]`, `notes[]`, `updated` | 閲覧専用の作業文脈キャッシュ（揮発性・正本でない・実行権なし） |

### C.2 `Run` と `Execution` / `Attempt` の対応整理

TTJ 1.1.0 では、独立した `Execution` オブジェクトおよび `Attempt` オブジェクトを導入しない。

| 概念 | TTJ 1.1.0 での対応先 | 設計根拠 |
|---|---|---|
| 実行試行（Attempt / Execution） | **`Run`** | 1回の Job 実行・再試行・再起動ごとに1件の `Run` を作成する。待機制御（`wait`）とプロセス生存観測（`activity`, `observed`）も同一試行に属するため `Run` に統合し、終了確定時点でその `Run.id` の試行権限を閉じる |
| 実行権・現行試行結合・フェンシング | **`Execution Claim`** | 実行権は Ticket（`ticket_id`）単位で複数回の `Run` や巻き取りを跨いで管理する必要があるため、`Run` から分離して正本化し、`generation` と `active_run_id` で二重起動・同世代旧 Run・旧 Session 復帰をすべて遮断する |
| 作業文脈の申し送り | **`Handoff`** | 実行履歴・再開条件（永続正本）と現在状況キャッシュ（揮発）のライフサイクルが異なるため分離 |

---

## D. 状態遷移

### D.1 作業状態（`status`）の遷移表（Task / Ticket / Job）

| 現在状態 | 許可される遷移先 | ガード条件・主な規則 |
|---|---|---|
| `todo` | `doing`, `waiting`, `paused`, `blocked`, `canceled` | 初期値。`doing` 時は開始記録を同時確定し、Job 開始時は Ticket と `todo` の Task も同時に `doing` にする。未割当 Ticket の前提待ちは依存評価経路で `waiting` 遷移可 |
| `doing` | `waiting`, `paused`, `blocked`, `completed`, `canceled` | 実行操作では認証済み Session＋`active` Claim（現行 `session_id`・`generation`）＋対象 Job＋現行 `Run.id` を照合 |
| `waiting` | `todo`, `doing`, `paused`, `blocked`, `completed`, `canceled` | Task・Ticket・Job における計画内の外部結果・前提待ち（対象や Worker 種別で分岐しない）。解除後は現在の未完了 Job のみから一方向続行。待機結果の確認証拠で `criteria` 成立なら `completed` 可 |
| `paused` | `todo`, `doing`, `waiting`, `blocked`, `canceled` | 明示的な保留。解除前の実行再開を禁止。明示解除時は保持していた未解消の `blocked`・`waiting`・前提依存を先に再評価し、残存制約があれば `blocked` / `waiting` へ遷移。全制約解消時のみ未着手なら `todo`・着手済みなら `doing` へ復帰（保留前状態の無条件復元禁止）。Task/Job の `paused → completed` 禁止 |
| `blocked` | `todo`, `doing`, `waiting`, `paused`, `canceled` | 計画外の継続不能課題。解除条件は必ず永続正本（`resume`・管理履歴等）に残し、実課題の解消確認で復帰（別 Ticket 完了のみでの自動解除禁止）。Task/Job の `blocked → completed` 禁止 |
| `completed` | **なし（終端）** | 再開禁止。Ticket 完了時は Claim を解放・失効。修正・再実行は新 Ticket で対応 |
| `canceled` | **なし（終端）** | 再開禁止。Ticket は中止理由（`cancel_reason`）必須記録と Claim 失効（`revoked`）を同時確定。Ticket 自身の中止は全 Job 終端による `completed` 集約より優先 |

- **Ticket の集約例外**：Ticket 自身が `canceled` でなければ、全 Job が `completed` または `canceled` になった時点で（未着手・保留中・ブロック中からでも）`completed` へ集約する。

### D.2 操作種別ごとの認可経路（実行操作 vs 管理・評価・証拠取り込み）

| 操作種別 | 主な遷移・操作 | 認可条件 | 共通の保護 |
|---|---|---|---|
| **実行操作** | Job 実作業開始（`todo → doing`）、実行中 `waiting` 移行・解除後の続行、Run 起因の結果確定、Session からの Handoff 更新 | `Session.worker_id == Ticket.owner_id`、`claim.state == "active"`、`claim.session_id == session_id`、`claim.generation == generation`、対象現在 Job、現行 `claim.active_run_id == run_id`（Run 開始時は `active_run_id == null` を原子的結合） | 終端復活禁止・監査記録必須・閉じた `Run.id` / 旧 `generation` 拒否 |
| **管理操作** | 権限保持運用者による Task/Ticket/Job の `paused`・明示解除・`canceled`・依存変更・範囲変更（Claim 未発行時含む） | 運用で定義された権限保持主体の認可。対象範囲に `active` Claim がある場合は中止時に `revoked` 失効、保留時に安全な停止要求を同時確定 | 同上 |
| **依存評価操作** | 未割当 Ticket の `depends` 評価による `todo ↔ waiting`、前提 Ticket 中止による `blocked` | 参照先 Ticket の確定状態に基づく決定的評価（Claim 不要） | 同上 |
| **証拠取り込み操作** | Run 終了後に到着した検証済み外部応答・確認証拠による `waiting → completed`（`result` 確定）または待機解除 | 現行 `active` Claim を保持する認証済み Session、または検証済み証拠を取り込む認可済み主体。現在 Job の `criteria`・待機条件と照合して原子的確定 | 同上（旧 Run・旧 `generation` からの送信は拒否） |
| **独立監視記録** | 失効後・終了後を含む対象 Run の `activity: "stopped"`・`observed`・停止証拠の記録 | 実行主体から独立した信頼できる監視機構による専用観測経路 | 業務状態（`status`/`result`/`Handoff`）は一切変更しない。監視主体・時刻を監査記録 |

### D.3 `Job.status` と `Job.result` の分離例

| シナリオ | `Job.status` | `Job.result` | 後続の扱い |
|---|---|---|---|
| 通常実装 Job が `criteria` 達成 | `completed` | `passed`（または成果概要） | 次の Job へ進む |
| レビュー Job 依頼送信直後（未回答） | `waiting` | （未確定） | `Run.wait` で待機。送信だけでは完了しない |
| レビュー Job が差し戻し証拠を受領 | `completed` | `changes_requested` | 完了済み Job は巻き戻さず、修正用新 Ticket を作成 |
| テスト Job が全項目を実行し失敗を確認 | `completed` | `failed` | テスト実行 Job 自体は完了。修正・再テストは新 Ticket |
| 複合スクリプトが異常終了し継続不能 | `canceled` | `error` | Ticket も `cancel_reason` 付きで `canceled`。安全確認後に新 Ticket で再実行 |

### D.4 `Run.activity`（生存観測）の遷移・保証範囲・観測期限

| 現在値 | 許可される遷移先 | 規則 |
|---|---|---|
| `unstarted` | `alive`, `unknown`, `stopped` | 起動前予約。この状態のみ `Run.started` 未設定を許容。起動後に `unstarted` へ戻すことは禁止 |
| `alive` | `unknown`, `stopped` | 生存観測には有効期間があり、最終観測時刻 `observed` から有効期間を超過した場合（Heartbeat 途絶・監視タイムアウト等）は実効値が必ず **`unknown`** になる |
| `unknown` | `alive`, `stopped` | 生存再確認で `alive`、独立監視等による確実な停止確認で `stopped` |
| `stopped` | **なし（同一 Run の終端観測値）** | 当該 Run の実行主体および配下の実行コンテキストが終了し、新規の副作用操作を開始できないことを保証。時間経過で `unknown` に戻らず、`stopped → alive`・`stopped → unknown` 禁止。再起動は新しい `Run` を作成する（ただし停止前に開始された外部操作・残存副作用の安全確認は別途要する） |

### D.5 `Execution Claim` の状態遷移・世代規則・外部フェンシング

- **キー**：`ticket_id`（Ticket ごとに1レコード。独立 Ticket は別キーで並列実行可）。
- **`generation` 規則**：Ticket 作成時の初期状態（`state: "unassigned"`, `session_id: null`, `active_run_id: null`）で **`generation = 1`**（正の整数）。以下の状態切替1回ごとに厳密に **`+1`** 進める。
- **`session_id` / `active_run_id` の `null` 規則**：`state` が `unassigned`・`released`・`revoked` のときは必ず `session_id = null` かつ `active_run_id = null`。`active` のときのみ非 `null` の `session_id` を持ち、有効な試行の実行中のみ単一の `active_run_id` を持つ（Run 終了確定時に `active_run_id = null` へ戻し試行権限を閉じる）。

```text
unassigned (session=null, run=null, gen=1)
   │  [初回取得: CAS gen 1→2, 非終端・過去実行残存なし]
   ▼
active (session=S1, run=null|R_cur, gen=g)
   ├── [正常解放 / Ticket completed: gen g→g+1] ──────→ released (session=null, run=null, gen=g+1)
   │                                                          │ [再取得: CAS gen +1, 非終端・過去Run全stopped・安全条件成立]
   │                                                          └──→ active (session=S_next, run=null, gen=g+2)
   │
   ├── [中止 / 強制失効 / 新Ticket巻き取り: CAS gen g→g+1] ─→ revoked (session=null, run=null, gen=g+1)
   │                                                          │ [再取得: CAS gen +1, 非終端かつ後続Ticket未移管・全stopped・安全条件成立のみ]
   │                                                          └──→ active (session=S_same_owner, run=null, gen=g+2)
   │                                                          ※ 終端Ticketまたは後続Ticket移管済みなら revoked は永久終端（再取得禁止）
   │
   └── [同一Worker別Session S2へ安全移譲: CAS gen g→g+1] ─→ active (session=S2, run=null, gen=g+1)
```

- **同世代旧 Run / stale Run / 旧 `generation` / 外部副作用の拒否**：
  1. 実行操作では認証済み `Session.worker_id == Ticket.owner_id`、`claim.state == "active"`、`claim.session_id == session_id`、`claim.generation == generation`、対象現在 Job、および現行 `claim.active_run_id == run_id` をすべて照合する。同世代でも終了済みの旧 `R1` は `R1 != active_run_id`（または閉じた試行）で即拒否される。
  2. 同一 Job への並行再試行要求は `claim.active_run_id` の原子的設定（`null → R_new`）で排他し、最大1つの Run のみ起動する。
  3. 副作用を防ぐ必要がある操作では、**効果の確定側（外部ゲートウェイ・実行環境・リソース更新口）でも失効済み権限（旧 `generation`・失効済み Claim・閉じた `run_id`）を拒否**することを準拠必須とし、失効前に開始済みの外部操作が残存し得る場合は安全確認完了まで後続実行を保留する。

### D.6 巻き取り可否マトリクス・3操作の区別・途中停止耐性

**3 操作の明確な区別**：
1. **生存中 Run への監視再接続**：`Run.activity == "alive"` の同一 Run に再接続（新 Run 作成なし・`generation` 変更なし）。
2. **停止確認後の同一 Worker 別 Session への Claim 移譲（再開）**：全過去 Run が `stopped` かつ安全条件成立後に CAS（`generation +1`）で同一 Ticket の Claim を移譲し、直前 Run の `wait`・再開位置・正本 `resume` を引き継いで状態を再評価してから続行。
3. **業務上の再実行（新 Ticket）**：別 Worker への巻き取り、差し戻し・テスト失敗後の修正、異常終了・Session 障害で継続不能な場合のやり直しとして新 Ticket を作成。

**巻き取り可否マトリクス**（過去 Run が未停止のまま残存する場合は、Claim が `revoked` 等で `session_id: null` であっても「未割当」とはみなさず稼働中・状態不明ガードを優先）：

| 対象 Ticket / 過去 Run の状態 | Handoff 参照 | 巻き取り / 取得可否 | 条件・補足 |
|---|---|---|---|
| 未割当（再取得可能な非 `active` Claim、**かつ過去 Run に `alive`/`unknown`/安全未確認なし**） | 可 | **可** | 終端 Ticket・後続移管済み Ticket は不可。`paused` は明示解除が必要 |
| 稼働中（`alive` の Run が残存。Claim 失効済み含む） | 可 | **不可** | 稼働中 Session からの奪取・競合開始を禁止（UC-17） |
| `waiting` かつ稼働中（`status: waiting` & `activity: alive`） | 可 | **不可** | 待機中でも Session 生存中は奪取禁止 |
| 停止確認済み（全過去 Run が `stopped`、`waiting` 中含む） | 可 | **安全条件成立時のみ可** | 失効前開始済み操作・副作用残存の疑いがあれば安全確認・復旧完了まで保留（UC-18, UC-25, UC-S03） |
| 状態不明（`unknown` の Run が残存。Heartbeat 途絶・Claim 失効後停止未確認含む） | 可 | **不可** | 停止未確認のまま巻き取り禁止（UC-19） |

**原子的／永続復旧可能な巻き取りフロー（同時要求は最大1件成功・クラッシュ耐性：UC-20, UC-S02）**：
1. 正本（Ticket・全過去 Run・Execution Claim・管理履歴）および Handoff を参照する。
2. 過去 Run に `alive` または `unknown` が1件でも残存していれば即拒否（Claim・Ticket 無変更）。
3. 全過去 Run が `stopped` かつ安全条件成立を確認（未確認なら拒否・保留、Claim・Ticket 無変更）。
4. 期待世代 `expected_generation = g` と冪等操作識別子 `operation_id` で原子的 CAS を実行する：
   - **新 Ticket へ移す巻き取り（別 Worker 移管・障害再実行）**：
     - 単一の原子的確定、または旧 Claim の CAS（`g → g + 1`, `state: "revoked"`, `session_id: null`, `active_run_id: null`, 永続予約 `takeover: { operation_id, claimant_session_id, target_owner_id, successor_ticket_id }` 記録）により、(a) 旧 Claim の再取得永久禁止、(b) 未終端旧 Ticket `T` の `status: "canceled"`（`cancel_reason` 必須・`owner_id` 不変）終端化、(c) 新 Worker を `owner_id` とする後続 Ticket `T_new`（`refs` に旧 Ticket `T`）の一意作成と新 Claim 取得、を確定する。
     - CAS 直後に勝者 `S2` がクラッシュしても、新世代を読んだ別 Session `S3` は確定済み `takeover` 予約と世代不一致で拒否され、`S2` の同一 `operation_id` 再送または復旧処理が `T_new` を厳密に1件だけ冪等に確定・返却する。
   - **同一 Worker 別 Session への安全移譲（Claim 移譲）**：
     - 旧 Claim を `state: "active"`, `session_id: S_new`, `active_run_id: null`, `generation: g + 1` に原子更新する。
     - 直前 Run の `wait`・再開位置・正本 `resume` を引き継いで状態を再評価し、`paused` 中なら明示解除まで実行を再開せず、残る `waiting`・`blocked` を保持する。待機解除で `doing` へ遷移した場合のみ新 Run を開始し、現在の未完了 Job のみから一方向に続行する。

### D.7 Handoff の更新照合・鮮度単調性・消失耐性

- **照合と単調性**：Session による更新は、表示対象 `current_ticket_id` の `active` Claim（`session_id`・最新 `generation`）と一致し、かつ Task 単位の `Handoff.revision`（および同一 Ticket 更新時は `source_generation`）が単調増加する場合のみ受理する。遅延更新や古いスナップショットからの再生成で現行表示を上書きしない。
- **消失・再起動耐性**：再開・障害解除・再試行に必要な条件は必ず永続正本（`depends`・`Run.wait`・`Run.result`・`Ticket.cancel_reason`・`resume`・管理履歴）に保存するため、`Handoff` 消失および再起動後も正本のみから再開判断と `Handoff` 再生成が可能である（UC-22）。
- **並列実行非阻害**：同一 Task 内に独立した複数 Ticket が別キー（別 `ticket_id`）の `Execution Claim` で並列稼働する場合でも、Task の `Handoff` スロットが1つであることを理由に実行を直列化・ブロックしない（UC-24）。
