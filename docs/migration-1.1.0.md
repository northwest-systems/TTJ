# TTJ 1.1.0 仕様変更概要・データモデル差分・状態遷移

本文書は、TTJ `1.0.0` から `1.1.0`（`protocol: "TTJ/1.1.0"`）への仕様改定における変更概要（提出物 A）、データモデル差分（提出物 C）、および状態遷移の整理（提出物 D）をまとめたものである。規範的な作業規則は [規格本文](../skills/ttj/references/specification.md) および [JSON参照形式](../skills/ttj/references/json-format.md) を正本とする。

---

## A. 仕様変更概要（1.0.0 → 1.1.0）

### A.1 変更の背景と目的

TTJ `1.0.0` は最小限の記入で作業と知識を継続できる一方、複数 Session や自動化基盤と併用する運用において以下の課題が顕在化していた。

1. **責任主体と実行単位の未分離**：`assignee` に人・LLM・LLM セッションが混在でき、同一 Worker の複数 Session 間の実行権境界や無断実行の拒否条件が曖昧だった。
2. **共同作業・Ticket 内担当交代の複雑化**：複数担当による同一 Job 実行や Ticket 途中での `assignee` 変更を許容すると、所有責任・実行権失効・履歴境界が複雑になる。
3. **進行状態と業務判定の混同**：レビュー差し戻しやテスト失敗など「Job 自体は完了したが業務結果が否定的」なケースを `status` と `result` に明確分離する規則が弱かった。
4. **実行権・現行試行結合・生存判定・巻き取り規則の不足**：Heartbeat 途絶と停止確認済みの区別、同時巻き取りおよび途中停止クラッシュ・予約勝者永久離脱時の決定性、切替直前の新 Run 介入による陳腐化安全確認の排除、同世代の閉じた旧 Run や復帰した旧 Session の更新拒否、副作用防止のための外部フェンシングを `Execution Claim` と `generation`/`run_seq` フェンシングで厳密に保証する必要があった。
5. **Handoff の責務肥大化**：`Ticket.handoffs` が短期申し送り・Ticket 間参照・課題記録を兼ねていたため、Task 単位の揮発性状況スロット（消失横断鮮度管理付き）と、永続的な Ticket 参照（`refs`）・中止理由（`cancel_reason`）・再開条件（`resume` / `Run.wait` / 管理履歴）を分離する必要があった。

### A.2 追加・変更・廃止した要件一覧

| 区分 | 項目 | TTJ 1.0.0 | TTJ 1.1.0 | 変更理由 |
|---|---|---|---|---|
| 変更 | プロトコル版 | `protocol: "TTJ/1.0.0"` | `protocol: "TTJ/1.1.0"` | 非互換の変更（`assignee`・`Ticket.handoffs` 廃止、`Worker`/`Session`/`Execution Claim` 追加）を含むマイナー改定（合意済み 1.1.0 方針） |
| 変更 | フィールド命名方針 | 1単語推奨 | 1単語推奨＋意味のある連結名（`owner_id`, `worker_id`, `task_id`, `ticket_id`, `session_id`, `active_run_id`, `run_seq`, `current_ticket_id`, `active_session_id`, `current_job_id`, `source_revision`, `cancel_reason`）を統一採用。既存 `project`/`task`/`job`/`target` は維持 | 参照先と識別子の意味を曖昧なく表現し、仕様・JSON・SKILL 間の表記揺れを防ぐため |
| 追加 | `Worker` | 未定義（`assignee` の任意識別値） | 正式定義（不変 `id`・変更可能 `name`）。種別分類・`owner_type` 禁止 | 作業の責任主体を一意化しつつ、人間・Bot・AI 非依存を徹底するため |
| 追加 | `Session` | 未定義 | 正式定義（`id`・`worker_id`・`started`・`ended`） | 責任主体（Worker）と実行単位（Session）を分離し、同一 Worker の複数 Session を個別識別するため |
| 廃止/置換 | 担当モデル | `Ticket.assignee`・`Job.assignee`（配列・継承）、共同作業、同一 Ticket の担当交代 | **廃止**。`Ticket.owner_id`（単一・不変の `Worker.id`）に統一。別 Worker 作業・追加依頼は別 Ticket | 責任境界を一元化し、排他・巻き取り・監査を単純明快にするため |
| 変更 | `Job` | `instructions`・`criteria`・`assignee`・任意 `result`・`resume` | `instructions`・`criteria`・`resume` を維持、`assignee` を廃止、`status`（進行）と `result`（業務判定・中断・エラー）を明確分離。独立 `Request`・子 Job・種別分類・待機制御構造を禁止 | Job を機械的・証拠ベースで完了判定できる最小単位に保つため |
| 変更 | `Task` | 目標・`criteria`・`deadline`・`status`（`waiting` 含む）・`resume` | 長寿命目標管理を強化。Run を持たない Task レベルの計画内待機（`waiting`）と `resume` を維持。Ticket 増加で Task を増殖させないこと、完了 Ticket 数のみで達成率を決めないことを明記。揮発性 `Handoff` 単一スロットを所属させる | 修正・再実行による Ticket 増加を自然な運用として受け止めつつ Task 単位で成果を統括するため |
| 変更 | `Ticket` 参照と終了 | `Ticket.handoffs` に元 Ticket ID を手書き | `Ticket.handoffs` を**廃止**。永続的な情報共有参照 `refs`（`depends` と明確区別）および `cancel_reason`（`canceled` 時必須）・`artifacts` を導入。次 Ticket 未作成でもアーカイブ可能と明記。完了依存グラフは `depends` と未解決 `Run.wait` のみで定義 | 揮発性 Handoff と永続履歴・参照を分離し、重複作業と文脈断絶を防ぐため |
| 拡張 | `Run` | `id`・`job`・`actor`（配列）・`started`・`ended`・`result`・`generation`・`activity`・`observed` | 実行試行の正本として拡張：`ticket_id`・`owner_id`・`session_id`・`generation`・`wait`（解消時 `resolved` 含む）を保持。`started`（実作業開始確定時必須、`unstarted` や未起動閉鎖時は省略可）と終了時 `result` を条件付き必須化。独立 `Execution`/`Attempt` オブジェクトは導入しない | 不要なオブジェクトや状態機械を増やさず、待機制御・単一 Session 実行・試行権限閉鎖を正本化するため |
| 追加 | `Execution Claim` | 概念的言及（Lease 推奨、`generation` 参照） | Ticket 単位（一意キー：`ticket_id`）の実行権正本として正式定義（`task_id`・`ticket_id`・`owner_id`・`session_id`・`active_run_id`・`generation`（初期値 `1`、切替ごと `+1`）・`run_seq`（初期値 `0`、試行結合/閉鎖ごと `+1`）・`state`（`unassigned → revoked`・`released → revoked` 含む）・`takeover`（`status`: `reserved`/`committed`））。外部フェンシングを準拠必須要件とし、開始前/開始後の照合分離と同一 Claim 内安全閉鎖を規定 | 同一作業系列の最大1 Session・最大1有効 Run、切替時陳腐化防止、同時巻き取り抑止、勝者永久離脱復旧、旧 `generation`/旧 Run 拒否を厳密化するため |
| 変更 | `Handoff` | `Ticket.handoffs`（文字列配列） | **Task 所属の揮発性単一スロット**へ再設計（揮発連番 `revision` に加え、消失横断比較用の永続正本版 `source_revision` を導入）。閲覧権と実行権を厳格分離、管理再生成の認可を明文化、再開条件は必ず永続正本へ残し、無関係 Ticket の並列実行を遮らない | 複数 Session からの安全な状況把握と、消失をまたぐ鮮度保証・正本からの責務分離を両立するため |
| 追加 | Ticket 増加対策 | 未記載 | テンプレート・標準 Job 列・後続 Ticket 生成支援・成果物/中止理由参照・系列表示・終了 Ticket 折りたたみ・Task 進捗集約・現在 Handoff 表示を準拠必須要件（`[必須]`）として明記（交換形式・UI 実装方法は選択可） | 単一 Owner・一方向 Ticket 運用での起票・管理負荷を最小化するため |
| 維持 | 既存 1.0.0 コア規則 | 4階層・Job 列固定・終端再開禁止・循環 `depends` 禁止・`Task.deadline`・Task レベル `waiting`・`paused` の意味・同一計画の再試行および後続 Job 開始時の「停止または安全条件」・中断/失敗時の記録と根拠のない再試行禁止・`Project.knowledge`（`active`/`supersed`）・特定基盤非依存 | すべて維持（巻き取りのみ原典 1.1.0 要件どおり「全過去 Run の停止確認かつ安全条件」の両方必須） | TTJ 1.0.0 の堅牢な基盤原則との一貫性と後方互換の明確さを保つため |

### A.3 既存 TTJ 1.0.0 との矛盾解決および設計判断の記録

1. **巻き取り設計の選択（単一原子確定と復旧可能予約の統一契約）**
   - **比較**：(案 A) 単一トランザクションの原子的確定のみを必須とする設計、(案 B) 単一原子確定を標準としつつ、分割確定ストレージ向けに `Claim.takeover` 永続予約（`status: "reserved" | "committed"`）による復旧可能手続も同一の外部観測契約で許容する設計。
   - **選択と判断理由**：**案 B（単一原子確定と永続予約復旧の統一契約）**を採用した。理由は、(1) 現仕様（`specification.md` §8.4・`json-format.md` §10・`UC-S02`）が既に `Claim.takeover` を備えており仕様変更の影響が最小であること、(2) 単一原子確定では `takeover` が直ちに `status: "committed"` として確定するため同一 `operation_id` 再送の冪等返却と別要求の衝突拒否を同じ契約で記述できること、(3) 分割確定環境でも予約時点からの旧 Ticket 再取得永久禁止と、予約勝者永久離脱時の認可済み復旧経路（終了済み勝者への新 Claim 誤結合を禁じ `unassigned(gen=1)` を維持）により単一原子確定と完全に同一の安全性・決定性を保証できること、の3点である。
2. **1.0.0 規則の維持：同一計画の再試行・後続 Job 開始（「停止 *または* 安全条件」）の実現経路**
   - 1.0.0 規格（L244, L246）が認めた「前の実行の停止 **または** 安全条件」を、Claim を失効させずに行えるよう **同一 Claim 内での試行権限の安全閉鎖（Run Safe Closure）** を定義した。同一 Session の旧試行 `R1` が `unknown` のままでも、`R1` を永久閉鎖・外部フェンシングで無効化し残存副作用がない安全条件を満たせば、`claim.active_run_id` を `null` へ解放し `claim.run_seq` を `+1` して同じ `active` Claim で新 `Run` `R2` を結合できる（Job 単独中止後の後続 Job 開始も同じ）。一方、別 Session・別 Worker への**巻き取り**は原典 1.1.0 要件どおり「全過去 Run の `stopped` 確認 **かつ** 安全条件」の両方必須を維持する。
3. **1.0.0 規則の維持：Run を持たない Task レベルの `waiting` と親 Task 状態からの独立性**
   - 1.0.0 規格（L107–122）の Task レベル計画内待機 `waiting` を維持し、管理操作・認可済み証拠取り込み経路による入退・完了を明文化した。また親 Task が別 Ticket 起因で `waiting`・`blocked` であっても、独立した実行可能 Ticket・Job は開始可能（親 Task の `paused`・終端のみ開始拒否）であることを正本・JSON・SKILL すべてで一致させた。
4. **`Ticket.handoffs` 廃止と永続相互参照・再開条件の正本保存**
   - 永続的な関連参照は情報共有専用の `Ticket.refs` へ分離し、再開・障害解除・再試行に不可欠な条件と証拠は永続正本（`depends`・`Run.wait`・`Run.result`・`Ticket.cancel_reason`・`resume`・管理履歴）へ必須保存とした。さらに `Handoff.source_revision` により、揮発スロット消失・再生成をまたぐ古いスナップショットの遅延適用を確実に拒否できるようにした。
5. **完了依存グラフの辺と未解決待機の定義**
   - 完了依存グラフに含める辺を「非終端 Ticket の `depends`」と「現在未解決の `Run.wait`（停止・移譲中に引き継がれている未解決待機を含む）」に限定し、解消済み・解除済みの `Run.wait` は履歴に保持しつつ辺から外すことで、待機解消後の逆向き登録を許可し、かつ逆向き競合登録が双方同時成功しない原子的な循環検証境界を定義した。
6. **`Job` の `instructions` / `criteria` と `Project.knowledge` の `supersed` 綴り**
   - 1.0.0 の `instructions` / `criteria` および `Project.knowledge.status` の正確な綴り `active` / `supersed` を完全維持した。

### A.4 既存実装（TTJCore 等）への影響

- **破壊的変更（スキーマ・バリデーション）**：
  - `protocol` の受理版を `"TTJ/1.1.0"` へ更新。
  - `Worker`・`Session`・`Execution Claim`（一意キー：`ticket_id`、`active_run_id`・`generation` 初期値 `1`・`run_seq` 初期値 `0`・`takeover`（`status`: `reserved`/`committed`）含む）の追加と参照整合性検証。
  - `Ticket.assignee`・`Job.assignee`・`Ticket.handoffs` の廃止と、`Ticket.owner_id`（必須・不変）・`Ticket.refs`・`Ticket.cancel_reason`（`canceled` 時必須）・`Ticket.artifacts` の追加。
  - `Run.actor: array<string>` から `Run.owner_id` + `Run.session_id` + `Run.ticket_id` + `Run.wait`（`resolved` 含む）への変更、`started`（実作業開始確定時必須、未起動予約・未起動閉鎖時省略可）および終了時 `result` の条件付き必須化。
  - `Handoff` を Ticket 配列から Task 所属単一スロット（`revision`・永続正本基準版 `source_revision`・`source_generation` 付き）へ移行。
- **排他・実行制御・外部フェンシング層への影響**：
  - Run 開始前照合（`active_run_id == null` 検証＋原子的結合＋`run_seq +1`）と開始後照合（`active_run_id == run.id` 非 `null` 一致）を分離実装する。
  - 同一 Claim 内での試行権限安全閉鎖（`unknown` 試行の永久閉鎖＋外部フェンシング＋`active_run_id = null`＋`run_seq +1`）を実装し、巻き取りと新規 Run 開始・閉鎖を同一競合境界（`expected_generation` + `expected_run_seq`）で排他する。
  - 新 Ticket 移管巻き取りで「同一 `operation_id` 再送冪等判定 → 既存予約/確定衝突判定 → 前提/競合検証・確定」の順序および勝者永久離脱時の認可済み復旧（終了済み勝者へ誤結合せず `unassigned(gen=1)` 維持）を実装する。
  - 独立監視の停止証拠記録経路、管理操作（`unassigned → revoked`・二重失効抑止・Task 計画内 `waiting` 含む）、依存評価（未解決待機辺のみ＋逆向き競合の原子的検証）、Run 終了後 `waiting`/`doing` Job への認可済み証拠取り込み経路を実装する。
  - 副作用を防ぐ必要がある操作では、外部ゲートウェイ／効果確定側でも失効済み権限（旧 `generation`・失効 Claim・閉じた同世代 `Run.id`）を必須拒否し、失効前開始済み操作が残存し得る場合は安全確認完了まで後続実行を保留する。

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
| **Task** | `id`, `project`, `objective`, `criteria`, `deadline`, `status`, `resume` | **維持＋責務明確化**：長寿命成果目標。`status`（Run を持たない Task レベルの計画内 `waiting` 含む）・`resume` 維持。揮発性単一スロット `Handoff` が所属（別レコード時は `Handoff.task_id` で結合） | 複数 Ticket の統括・結果集約・最終達成判定 |
| **Ticket** | `id`, `task`, `objective`, `jobs`, `assignee`, `depends`, `handoffs`, `status`, `resume` | **追加**：`owner_id`（必須・不変）, `refs`（任意：情報共有参照）, `artifacts`（任意）, `cancel_reason`（**条件付き必須：`canceled` 時必須**）<br>**維持**：`status`, `depends`, `resume`<br>**廃止**：`assignee`, `handoffs` | 単一 Worker 担当・一方向進行の短命作業単位 |
| **Job** | `id`, `objective`, `criteria`, `instructions`, `assignee`, `status`, `result`, `resume` | **維持**：`id`, `objective`, `criteria`, `instructions`, `status`, `resume`<br>**明確化**：`result`（`status` と分離した業務判定・エラー結果）, `artifacts`（任意）<br>**廃止**：`assignee` | `criteria` で完了判定できる最小実行単位。複雑な構造を持たない |
| **Run** | `id`, `job`, `actor[]`, `started`, `ended`, `result`, `generation`, `activity`, `observed` | **追加**：`ticket_id`（必須）, `owner_id`（必須）, `session_id`（必須）, `wait`（任意：`job_id`, `target`, `condition`, `since`, `resume_point`, `timeout`, `resolved`）<br>**廃止**：`actor[]`<br>**明確化**：`generation`（必須・正整数）, `started`（**条件付き必須：実作業開始確定時必須、`unstarted` 予約または未起動閉鎖 `unstarted → stopped` では省略可**）, `result`（**条件付き必須：終了確定時必須**） | Job の1回の実行試行（Execution / Attempt）および待機・観測の正本 |
| **Execution Claim** | （未定義；Lease 推奨のみ） | **新設（一意キー：`ticket_id`）**：`task_id`（必須）, `ticket_id`（必須）, `owner_id`（必須）, `session_id`（必須・`active` 以外は `null`）, `active_run_id`（必須・有効試行なしまたは非 `active` 時は `null`）, `generation`（必須・初期値 `1`、切替1回ごとに `+1`）, `run_seq`（必須・初期値 `0`、試行結合/閉鎖ごと `+1`）, `state`（必須：`unassigned`/`active`/`released`/`revoked`）, `acquired`（任意）, `revoked`（任意）, `takeover`（任意：`operation_id`, `claimant_session_id`, `target_owner_id`, `successor_ticket_id`, `status`: `reserved`/`committed`） | Ticket 単位の実行権・現行試行結合・実行系列版の正本（`generation` / `run_seq` フェンシング） |
| **Handoff** | `Ticket.handoffs: array<string>` | **全面再設計（Task 所属揮発性単一スロット）**：`task_id`（必須）, `revision`（必須・揮発スロット書込連番）, `source_revision`（必須・消失横断比較用の永続正本単調確定版）, `current_ticket_id`, `owner_id`, `active_session_id`, `current_job_id`, `source_generation`, `activity`, `context`, `stop_position`, `blockers[]`, `notes[]`, `updated` | 閲覧専用の作業文脈キャッシュ（揮発性・正本でない・実行権なし） |

### C.2 `Run` と `Execution` / `Attempt` の対応整理

TTJ 1.1.0 では、独立した `Execution` オブジェクトおよび `Attempt` オブジェクトを導入しない。

| 概念 | TTJ 1.1.0 での対応先 | 設計根拠 |
|---|---|---|
| 実行試行（Attempt / Execution） | **`Run`** | 1回の Job 実行・再試行・再起動ごとに1件の `Run` を作成する。待機制御（`wait`）とプロセス生存観測（`activity`, `observed`）も同一試行に属するため `Run` に統合し、終了確定または安全閉鎖時点でその `Run.id` の試行権限を閉じる |
| 実行権・現行試行結合・フェンシング | **`Execution Claim`** | 実行権は Ticket（`ticket_id`）単位で複数回の `Run` や巻き取りを跨いで管理する必要があるため、`Run` から分離して正本化し、`generation`・`active_run_id`・`run_seq` で二重起動・同世代旧 Run・旧 Session 復帰・切替前陳腐化をすべて遮断する |
| 作業文脈の申し送り | **`Handoff`** | 実行履歴・再開条件（永続正本）と現在状況キャッシュ（揮発）のライフサイクルが異なるため分離し、`source_revision` で消失をまたぐ鮮度比較を保証する |

---

## D. 状態遷移

### D.1 作業状態（`status`）の遷移表（Task / Ticket / Job）

| 現在状態 | 許可される遷移先 | ガード条件・主な規則 |
|---|---|---|
| `todo` | `doing`, `waiting`, `paused`, `blocked`, `canceled` | 初期値。`doing` 時は開始記録を同時確定し、Job 開始時は Ticket と `todo` の Task も同時に `doing` にする。親 Task が `waiting`/`blocked` でも独立 Ticket・Job の開始可。未割当 Ticket の前提待ちは依存評価経路で `waiting` 遷移可 |
| `doing` | `waiting`, `paused`, `blocked`, `completed`, `canceled` | 結合済み Run からの実行更新は開始後照合（`active_run_id == run.id` 非 `null` 一致）を要求。Run 終了後（`active_run_id == null`）の `doing` Job は認可済み証拠取り込み経路で完了可 |
| `waiting` | `todo`, `doing`, `paused`, `blocked`, `completed`, `canceled` | Task（Run なし含む）・Ticket・Job における計画内の外部結果・前提待ち（対象や Worker 種別で分岐しない）。解除後は現在の未完了 Job のみから一方向続行。待機結果の確認証拠で `criteria`（Task は完了条件全体）成立なら `completed` 可 |
| `paused` | `todo`, `doing`, `waiting`, `blocked`, `canceled` | 明示的な保留。解除前の実行再開を禁止。明示解除時は保持していた未解消の `blocked`・`waiting`・前提依存を先に再評価し、残存制約があれば `blocked` / `waiting` へ遷移。全制約解消時のみ未着手なら `todo`・着手済みなら `doing` へ復帰（保留前状態の無条件復元禁止）。Task/Job の `paused → completed` 禁止 |
| `blocked` | `todo`, `doing`, `waiting`, `paused`, `canceled` | 計画外の継続不能課題。解除条件は必ず永続正本（`resume`・管理履歴等）に残し、権限保持主体または現行 Claim 保持 Session が実課題の解消を確認して復帰（別 Ticket 完了のみでの自動解除禁止）。Task/Job の `blocked → completed` 禁止 |
| `completed` | **なし（終端）** | 再開禁止。Ticket 完了時は `active` な Claim を解放・失効（既解放/既失効なら二重変更なし）。修正・再実行は新 Ticket で対応 |
| `canceled` | **なし（終端）** | 再開禁止。Ticket は中止理由（`cancel_reason`）必須記録と未失効 Claim（`unassigned`/`active`/`released`）の `revoked` 失効を同時確定（既 `revoked` 済みは二重失効・追加世代加算なし）。Ticket 自身の中止は全 Job 終端による `completed` 集約より優先 |

- **Ticket の集約例外**：Ticket 自身が `canceled` でなければ、全 Job が `completed` または `canceled` になった時点で（未着手・保留中・ブロック中からでも）`completed` へ集約する。

### D.2 操作種別ごとの認可経路（開始前 / 開始後 / 管理・評価・証拠取り込み・独立監視）

| 操作種別 | 主な遷移・操作 | 認可・前提条件 | 確定境界・共通の保護 |
|---|---|---|---|
| **実行操作（開始前照合）** | 新規 Run 開始・結合（初回開始、同一計画通常再試行、Job 単独中止後の後続 Job 開始） | `Session.worker_id == Ticket.owner_id`、`claim.state == "active"`、`claim.session_id == session_id`、`claim.generation == generation`、対象現在 Job、開始ガード（対象 Ticket/Job が `todo`/`doing`、前提 `depends` 全 `completed`、親 Task が `paused`/終端でない）、**`claim.active_run_id == null`**（または安全閉鎖で解放済み） | 原子的結合で `claim.active_run_id = R_new`, `claim.run_seq +1`。拒否時は正本無変更。並行開始は最大1件成功 |
| **実行操作（開始後照合）** | 結合済み Run による進行更新・`waiting` 移行・結果確定・Session からの Handoff 更新 | 上記の認証済み Session・`active` Claim・対象現在 Job に加え、**`claim.active_run_id == run.id`（非 `null` 一致）**（Handoff 更新時は永続正本 `source_revision` 鮮度も照合） | 不一致（未結合・閉じた同世代旧 Run・旧 `generation`・別 Session）は拒否し正本無変更 |
| **試行権限の安全閉鎖** | 同一 `active` Claim 内の通常再試行・Job 単独中止後続開始で、`unknown` の旧 Run `R1` を閉鎖 | `active` Claim 保持 Session（または Job 単独中止の認可主体）。`R1` 永久閉鎖・終了 `result` 記録・外部フェンシングによる無効化・残存副作用なし確認 | Claim 失効なしで `claim.active_run_id = null`, `claim.run_seq +1` を原子確定（巻き取りには適用不可） |
| **管理操作** | 権限保持運用者による Task/Ticket/Job の `paused`・明示解除・`canceled`（`unassigned`/`released` Ticket 中止含む）・障害解消確認後 `blocked` 復帰・依存変更・範囲変更・予約勝者離脱時の巻き取り復旧確定・管理 Handoff 再生成 | 運用で定義された権限保持主体の認可。未失効 Claim は中止時に `revoked`（`generation +1`）、既 `revoked` 済みは二重失効なし。管理 Handoff 再生成は永続正本最新版（最新 `source_revision`）のみ読取・実行権非付与 | 終端復活禁止・監査記録必須・拒否時正本無変更 |
| **Task 計画内待機操作** | Run を持たない Task の外部承認・計画イベント待ち（`waiting`）および承認確認後の解除・完了 | 権限保持運用者（または解除/完了時の認可済み証拠取り込み主体）。解除条件を正本 `Task.resume` または管理履歴に必須保存 | 終端復活禁止・監査記録必須 |
| **依存評価操作** | 未割当 Ticket の `depends` 評価による `todo ↔ waiting`、前提 Ticket 中止による `blocked` | 参照先 Ticket の確定状態に基づく決定的評価（Claim 不要） | 終端復活禁止・監査記録必須 |
| **証拠取り込み操作** | Run 終了後（`active_run_id == null`）に到着した検証済み外部応答・確認証拠による現在 Job（`waiting` または `doing`）の `completed`（`result` 確定）または待機解除 | 現行 `active` Claim を保持する認証済み Session、または検証済み証拠を取り込む認可済み主体。対象 Ticket が `paused`/`blocked`/終端でないこと・現在 Job の `criteria`/待機条件を照合 | 閉じた旧 `Run.id` 名義・旧 `generation` からの直接更新は拒否。終端復活禁止・監査記録必須 |
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
| `unstarted` | `alive`, `unknown`, `stopped` | 起動前予約。実作業未開始のまま `unstarted → stopped` で取り消し閉鎖した Run は `started` を持たず終了 `result` のみ持つ。起動後に `unstarted` へ戻すことは禁止 |
| `alive` | `unknown`, `stopped` | 生存観測には有効期間があり、最終観測時刻 `observed` から有効期間を超過した場合（Heartbeat 途絶・監視タイムアウト等）は実効値が必ず **`unknown`** になる |
| `unknown` | `alive`, `stopped` | 生存再確認で `alive`、独立監視等による確実な停止確認で `stopped` |
| `stopped` | **なし（同一 Run の終端観測値）** | 当該 Run の実行主体および配下の実行コンテキストが終了し、新規の副作用操作を開始できないことを保証。時間経過で `unknown` に戻らず、`stopped → alive`・`stopped → unknown` 禁止。再起動は新しい `Run` を作成する（停止前に開始された外部操作・残存副作用の安全確認は別途要する） |

### D.5 `Execution Claim` の状態遷移・世代規則・外部フェンシング

- **キー**：`ticket_id`（Ticket ごとに1レコード。独立 Ticket は別キーで並列実行可）。
- **`generation` と `run_seq` 規則**：Ticket 正式作成時の論理的な初期 Claim（`state: "unassigned"`, `session_id: null`, `active_run_id: null`）で **`generation = 1`, `run_seq = 0`**。状態切替1回ごとに `generation` を厳密に **`+1`**、Run 結合・安全閉鎖・終了ごとに `run_seq` を **`+1`** する。
- **`session_id` / `active_run_id` の `null` 規則**：`state` が `unassigned`・`released`・`revoked` のときは必ず `session_id = null` かつ `active_run_id = null`。`active` のときのみ非 `null` の `session_id` を持ち、有効な試行の実行中のみ単一の `active_run_id` を持つ。

```text
unassigned (session=null, run=null, gen=1, run_seq=0)
   ├── [初回取得: CAS gen 1→2, 非終端・過去実行残存なし] ──→ active (session=S1, run=null|R_cur, gen=g, run_seq=s)
   └── [未取得Ticketの管理中止: gen 1→2] ──────────────────→ revoked (session=null, run=null, gen=2) ※永久終端

active (session=S1, run=null|R_cur, gen=g, run_seq=s)
   ├── [正常解放 / Ticket completed: gen g→g+1] ──────→ released (session=null, run=null, gen=g+1)
   │                                                          ├── [再取得: CAS gen +1, 非終端・過去Run全stopped・安全条件成立] → active
   │                                                          └── [管理中止 / 新Ticket巻き取り: CAS gen +1] ──────────────────→ revoked ※永久終端
   │
   ├── [中止 / 強制失効 / 新Ticket巻き取り予約・確定: CAS gen g→g+1] ─→ revoked (session=null, run=null, gen=g+1)
   │                                                          └── [再取得: CAS gen +1, 非終端かつ巻き取り予約/移管未済・全stopped・安全条件成立のみ] → active
   │                                                          ※ 終端Ticketまたは巻き取り予約(reserved)/確定(committed)済みなら revoked は永久終端
   │
   └── [同一Worker別Session S2へ安全移譲: CAS (gen=g, run_seq=s) → gen=g+1] ─→ active (session=S2, run=null, gen=g+1, run_seq=s)
```

- **同世代旧 Run / stale Run / 旧 `generation` / 外部副作用の拒否**：
  1. 開始前照合（`active_run_id == null`＋原子的結合＋`run_seq +1`）と開始後照合（`active_run_id == run.id` 非 `null` 一致）を分離し、同世代でも閉じた旧 `R1` は `R1 != active_run_id` で即拒否される。
  2. 副作用を防ぐ必要がある操作では、**効果の確定側（外部ゲートウェイ・実行環境・リソース更新口）でも失効済み権限（旧 `generation`・失効済み Claim・閉じた同世代 `Run.id`）を必須拒否**し、失効前に開始済みの外部操作が残存し得る場合は安全確認完了まで後続実行を保留する。

### D.6 巻き取り可否マトリクス・3操作の区別・途中停止・勝者離脱復旧

**3 操作の明確な区別**：
1. **生存中 Run への監視再接続**：`Run.activity == "alive"` の同一 Run に再接続（新 Run 作成なし・`generation` / `run_seq` 変更なし）。
2. **停止確認後の同一 Worker 別 Session への Claim 移譲（再開）**：全過去 Run が `stopped`・`active_run_id == null`・安全条件成立かつ `(expected_generation, expected_run_seq)` 一致を同一競合境界で検証して CAS 移譲（`generation +1`）し、待機があれば待機条件・再開位置を引き継いで再評価、`paused`/`blocked` は各制約を維持、待機・保留・障害がなく実行可能（`todo`/`doing`）なら新 Run を結合して続行。
3. **業務上の再実行（新 Ticket）**：別 Worker への巻き取り、差し戻し・テスト失敗後の修正、異常終了・Session 障害で継続不能な場合のやり直しとして新 Ticket を作成。

**巻き取り可否マトリクス**（過去 Run が未停止のまま残存する場合は、Claim が `revoked` 等で `session_id: null` であっても「未割当」とはみなさず稼働中・状態不明ガードを優先）：

| 対象 Ticket / 過去 Run の状態 | Handoff 参照 | 巻き取り / 取得可否 | 条件・補足 |
|---|---|---|---|
| 未割当（再取得可能な非 `active` Claim、**かつ過去 Run に `alive`/`unknown`/安全未確認なし**） | 可 | **可** | 終端 Ticket・巻き取り予約/移管済み Ticket は不可。`paused` は明示解除が必要 |
| 稼働中（`alive` の Run が残存。Claim 失効済み含む） | 可 | **不可** | 稼働中 Session からの奪取・競合開始を禁止（UC-17） |
| `waiting` かつ稼働中（`status: waiting` & `activity: alive`） | 可 | **不可** | 待機中でも Session 生存中は奪取禁止 |
| 停止確認済み（全過去 Run が `stopped` かつ `active_run_id == null`、`waiting` 中含む） | 可 | **安全条件成立かつ `(expected_generation, expected_run_seq)` 一致時のみ可** | 確認後に同世代で別 Run が開始・終了し `run_seq` が変化した場合は拒否。失効前開始済み操作・副作用残存の疑いがあれば保留（UC-18, UC-25, UC-S02, UC-S03） |
| 状態不明（`unknown` の Run が残存。Heartbeat 途絶・Claim 失効後停止未確認含む） | 可 | **不可** | 停止未確認のまま巻き取り禁止（UC-19） |

**新 Ticket 巻き取りの照合順序と復旧フロー（UC-20, UC-S02）**：
1. **同一 `operation_id` 再送の最優先判定**：旧 Claim に同一 `operation_id`・同一実質パラメータの `takeover` があれば、元の `expected_generation` が古くても世代不一致とせず、`committed` なら既存結果を冪等返却、`reserved` なら未完の確定を完了して返却（パラメータ相違は拒否）。
2. **既存予約・確定済み衝突の判定**：別の `operation_id` の `takeover`（`reserved` または `committed`）が存在する場合、提示された世代が現行世代と一致していても「既存予約／確定済み衝突」として拒否（正本無変更）。
3. **前提・競合境界検証と確定／予約**：全過去 Run `stopped`・`active_run_id == null`・安全条件成立・`generation == expected_generation`・`run_seq == expected_run_seq` を同一競合境界で検証し、旧 Claim を `revoked`（`generation +1`、予約時点から旧 Ticket 再取得永久禁止）とし、未終端旧 Ticket を `canceled`（`cancel_reason` 必須・`owner_id` 不変・旧 Claim 二重失効なし）、後続 Ticket `successor_ticket_id` を一意作成、`takeover.status = "committed"` とする。勝者 Session が生存する適格 Session なら後続 Claim を `unassigned(gen=1) → active(gen=2)` とし、勝者が終了済み（永久離脱）の場合は認可済み復旧主体が同一予約を確定し、後続 Claim を終了済み勝者へ誤結合せず **`unassigned(gen=1)` のまま維持**する。

### D.7 Handoff の消失横断鮮度比較・管理再生成・並列 Ticket

- **Session 更新と消失横断鮮度比較**：表示対象 `current_ticket_id` の `active` Claim（結合中は現行 `active_run_id` も一致）と、揮発スロット消失をまたいで比較可能な永続正本側の単調確定版 `source_revision`（永続正本最新および現行スロット以上、同一 `source_revision` 内では `revision` 単調増加）を検証する。消失前の旧スナップショットがより大きい `revision` を付けて遅延到着しても `source_revision` 比較で確実に拒否する（UC-22）。
- **管理再生成の認可**：Claim を持たない正規の管理再生成は、権限を持つ管理・システム主体として永続正本の最新確定版（最新 `source_revision`）を原子的に読み取って閲覧キャッシュのみを再構築する（実行権非付与・正本無変更・古い `source_revision` は拒否）。
- **並列実行非阻害**：同一 Task 内に独立した複数 Ticket が別キー（別 `ticket_id`）の `Execution Claim` で並列稼働する場合でも、Task の `Handoff` スロットが1つであることを理由に実行を直列化・ブロックしない（UC-24）。

### D.8 完了依存グラフの辺と逆向き競合登録の確定境界

- **辺の定義**：完了依存グラフに含める辺は「非終端 Ticket の `depends`」と「現在未解決の `Run.wait`（停止・移譲中で引き継がれている未解決待機を含む）」のみ。解消済み・解除済みの `Run.wait`（`resolved` 記録等）は履歴に残しつつ辺から外すため、待機解消後の逆向き登録は許可される。
- **競合確定境界**：循環を閉じ得る逆向き登録（`depends` と `Run.wait` の混在を含む）が並行発行された場合、原子的な循環検証・確定境界により高々一方のみ成功し、他方は循環依存として拒否される（UC-11）。
