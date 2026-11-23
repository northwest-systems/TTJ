# TTJ 1.1.0 仕様変更概要・データモデル差分・状態遷移

本文書は、TTJ `1.0.0` から `1.1.0`（`protocol: "TTJ/1.1.0"`）への仕様改定における変更概要（提出物 A）、データモデル差分（提出物 C）、および状態遷移の整理（提出物 D）をまとめたものである。規範的な作業規則は [規格本文](../skills/ttj/references/specification.md) および [JSON参照形式](../skills/ttj/references/json-format.md) を正本とする。

---

## A. 仕様変更概要（1.0.0 → 1.1.0）

### A.1 変更の背景と目的

TTJ `1.0.0` は最小限の記入で作業と知識を継続できる一方、複数 Session や自動化基盤と併用する運用において以下の課題が顕在化していた。

1. **責任主体と実行単位の未分離**：`assignee` に人・LLM・LLM セッションが混在でき、同一 Worker の複数 Session 間の実行権境界や無断実行の拒否条件が曖昧だった。
2. **共同作業・Ticket 内担当交代の複雑化**：複数担当による同一 Job 実行や Ticket 途中での `assignee` 変更を許容すると、所有責任・実行権失効・履歴境界が複雑になる。
3. **進行状態と業務判定の混同**：レビュー差し戻しやテスト失敗など「Job 自体は完了したが業務結果が否定的」なケースを `status` と `result` に明確分離する規則が弱かった。
4. **実行権・生存判定・巻き取り規則の不足**：Heartbeat 途絶と停止確認済みの区別、同時巻き取りの排他、復帰した旧 Session（stale Run）の更新拒否を `Execution Claim` と `generation` フェンシングで明確に保証する必要があった。
5. **Handoff の責務肥大化**：`Ticket.handoffs` が短期申し送り・Ticket 間参照・課題記録を兼ねていたため、Task 単位の揮発性状況スロットと、永続的な Ticket 参照（`refs`）・中止理由（`cancel_reason`）を分離する必要があった。

### A.2 追加・変更・廃止した要件一覧

| 区分 | 項目 | TTJ 1.0.0 | TTJ 1.1.0 | 変更理由 |
|---|---|---|---|---|
| 変更 | プロトコル版 | `protocol: "TTJ/1.0.0"` | `protocol: "TTJ/1.1.0"` | 非互換の変更（`assignee`・`Ticket.handoffs` 廃止、`Worker`/`Session`/`Execution Claim` 追加）を含むマイナー改定（合意済み 1.1.0 方針） |
| 変更 | フィールド命名方針 | 1単語推奨 | 1単語推奨＋意味のある連結名（`owner_id`, `worker_id`, `task_id`, `ticket_id`, `session_id`, `current_ticket_id`, `active_session_id`, `current_job_id`, `cancel_reason`）を統一採用。既存 `project`/`task`/`job`/`target` は維持 | 参照先と識別子の意味を曖昧なく表現し、仕様・JSON・SKILL 間の表記揺れを防ぐため |
| 追加 | `Worker` | 未定義（`assignee` の任意識別値） | 正式定義（不変 `id`・変更可能 `name`）。種別分類・`owner_type` 禁止 | 作業の責任主体を一意化しつつ、人間・Bot・AI 非依存を徹底するため |
| 追加 | `Session` | 未定義 | 正式定義（`id`・`worker_id`・`started`・`ended`） | 責任主体（Worker）と実行単位（Session）を分離し、同一 Worker の複数 Session を個別識別するため |
| 廃止/置換 | 担当モデル | `Ticket.assignee`・`Job.assignee`（配列・継承）、共同作業、同一 Ticket の担当交代 | **廃止**。`Ticket.owner_id`（単一・不変の `Worker.id`）に統一。別 Worker 作業・追加依頼は別 Ticket | 責任境界を一元化し、排他・巻き取り・監査を単純明快にするため |
| 変更 | `Job` | `instructions`・`criteria`・`assignee`・任意 `result` | `instructions`・`criteria` を維持、`assignee` を廃止、`status`（進行）と `result`（業務判定・中断・エラー）を明確分離。独立 `Request`・子 Job・種別分類・待機制御構造を禁止 | Job を機械的・証拠ベースで完了判定できる最小単位に保つため |
| 変更 | `Task` | 目標・`criteria`・`deadline` | 長寿命目標管理を強化。Ticket 増加で Task を増殖させないこと、完了 Ticket 数のみで達成率を決めないことを明記。揮発性 `Handoff` 単一スロットを所属させる | 修正・再実行による Ticket 増加を自然な運用として受け止めつつ Task 単位で成果を統括するため |
| 変更 | `Ticket` 参照と終了 | `Ticket.handoffs` に元 Ticket ID を手書き | `Ticket.handoffs` を**廃止**。永続的な情報共有参照 `refs`（`depends` と明確区別）および `cancel_reason`・`artifacts` を導入。次 Ticket 未作成でもアーカイブ可能と明記 | 揮発性 Handoff と永続履歴・参照を分離し、重複作業と文脈断絶を防ぐため |
| 拡張 | `Run` | `id`・`job`・`actor`（配列）・`started`・`ended`・`result`・`generation`・`activity`・`observed` | 実行試行の正本として拡張：`ticket_id`・`owner_id`・`session_id`・`generation`・`wait`（待機制御情報）を保持。独立 `Execution`/`Attempt` オブジェクトは導入しない | 不要なオブジェクトや状態機械を増やさず、待機制御と単一 Session 実行を正本化するため |
| 追加 | `Execution Claim` | 概念的言及（Lease 推奨、`generation` 参照） | 実行権の正本として正式定義（`task_id`・`ticket_id`・`owner_id`・`session_id`・`generation`・`state`）。原子的巻き取り手順と生存判定マトリクスを規定 | 同一作業系列の最大1 Session 実行・同時巻き取り抑止・旧 `generation` 拒否を厳密化するため |
| 変更 | `Handoff` | `Ticket.handoffs`（文字列配列） | **Task 所属の揮発性単一スロット**へ再設計。閲覧権と実行権を厳格分離、消失しても正式履歴を維持、無関係 Ticket の並列実行を遮らない | 複数 Session からの安全な状況把握と、正式履歴からの責務分離を両立するため |
| 追加 | Ticket 増加対策 | 未記載 | テンプレート・標準 Job 列・後続 Ticket 生成支援・成果物/中止理由参照・系列表示・終了 Ticket 折りたたみ・Task 進捗集約・現在 Handoff 表示を仕様上の位置付けとして明記 | 単一 Owner・一方向 Ticket 運用での起票・管理負荷を最小化するため |
| 維持 | 既存コア要件 | 4階層・Job 列固定・終端再開禁止・循環 `depends` 禁止・`Task.deadline`・`paused` の意味・`Project.knowledge`（`active`/`supersed`）・特定基盤非依存 | すべて維持 | TTJ 1.0.0 の堅牢な基盤原則との一貫性を保つため |

### A.3 既存 TTJ 1.0.0 との矛盾解決記録

1. **`Ticket.handoffs` 廃止と課題対応の相互参照**
   - 1.0.0 では「専用の関係フィールドを禁止し、`Ticket.handoffs` に元 Ticket ID を書く」規則だった。
   - 1.1.0 では `Ticket.handoffs` を廃止し、`Handoff` を Task 所属の**揮発性**スロットへ移したため、永続的な関連（修正対象・派生元・再実行元・前 Ticket 成果物/中止理由）を揮発性 Handoff に置くと消失時に文脈が失われる矛盾が生じる。
   - **解決**：情報共有専用の永続フィールド `Ticket.refs` を新設し、実行依存 `depends` と明確に区別した。`refs.kind` は表示・説明用の任意ラベルに留め、業務ロジックや状態遷移の分岐を禁止することで 1.0.0 の「不要な分類による複雑化防止」の精神と両立させた。
2. **共同作業・担当交代の廃止と複数者レビューの扱い**
   - 1.0.0 では 1 つの Job に複数 `assignee` を指定して共同レビューし、同一 Ticket 内で担当交代できた。
   - 1.1.0 では単一不変 `Ticket.owner_id` と Worker/Session 分離を優先し、旧 `assignee`・共同作業・同一 Ticket の Owner 変更を廃止した。
   - **解決**：複数 Worker が独立作業・レビューを行う場合は Worker ごとに別 Ticket を作成し並列実行できる（UC-09, UC-24）。同一 Worker の別 Session 引き継ぎのみ、停止・安全確認と原子的な `Execution Claim` 更新を条件に同一 Ticket 継続を許可し、別 Worker による巻き取りは旧 Ticket の `owner_id` を変えず新 Ticket を作成する規則で整合させた。
3. **`Run` 拡張 vs 独立 `Execution` / `Attempt` オブジェクト**
   - 原典ドラフトでは `Execution` / `Attempt` の語が混在していたが、1.0.0 には既に `Run`（Job の1回の実行試行）が存在した。
   - **解決**：仕様上 `Run` と `Execution`/`Attempt` を分離すべき意味的差異がないため、既存 `Run` を実行試行の正本として拡張（`ticket_id`, `owner_id`, `session_id`, `generation`, `wait`）し、実行権の正本を `Execution Claim` に分離する最小構成を採用した。
4. **`Job` の `instructions` / `criteria` と `Project.knowledge` の `supersed` 綴り**
   - 原典ドラフト案の `context` / `done_when` 置換は採らず、1.0.0 の `instructions` / `criteria` を維持した。
   - `Project.knowledge.status` は 1.0.0 仕様・JSON 例・SKILL すべてで一貫して使われていた正確な綴り `active` / `supersed` を維持した。
5. **`paused` と `waiting` / `blocked` および生存判定の関係**
   - 1.0.0 の `paused`（明示的な判断による保留。再開には明示解除＋残存課題の再評価が必要。Task/Job の `paused → completed` 禁止）を維持した上で、`status`（作業状態）と `Run.activity`（実行観測）を直交整理した。
   - **解決**：`paused` であっても、稼働中（`alive`）または状態不明（`unknown`）の Session から無断巻き取りはできず、`stopped` かつ安全条件成立時のみ巻き取りを許可し、再開には `paused` の明示解除を要する規則で矛盾なく統合した。

### A.4 既存実装（TTJCore 等）への影響

- **破壊的変更（スキーマ・バリデーション）**：
  - `protocol` の受理版を `"TTJ/1.1.0"` へ更新。
  - `Worker`・`Session`・`Execution Claim` レコードの追加と参照整合性検証（`Ticket.owner_id → Worker.id`、`Session.worker_id → Worker.id`、`Run.session_id → Session.id`）。
  - `Ticket.assignee`・`Job.assignee`・`Ticket.handoffs` の廃止と、`Ticket.owner_id`（必須・不変）・`Ticket.refs`・`Ticket.cancel_reason`・`Ticket.artifacts` の追加。
  - `Run.actor: array<string>` から `Run.owner_id: string` + `Run.session_id: string` + `Run.ticket_id: string` + `Run.wait` への変更。
  - 監査イベントの主体表現を `worker_id`・`session_id` へ統一。
  - `Handoff` を Ticket 配列から Task 所属単一スロット（再生成・破棄可能）へ移行。
- **排他・実行制御層への影響**：
  - すべての状態遷移・Job 結果確定・Handoff 更新 API に `session_id` と `generation` の照合ガードを追加し、旧 `generation` および未保持 Session を拒否する。
  - 巻き取り操作に `Run.activity == "stopped"` 確認・安全条件フラグ確認・原子的 CAS（Compare-And-Swap）による `generation` インクリメントを実装する。
  - 外部副作用を持つ処理では、実行ゲートウェイ側でも失効済み `generation` を遮断する連携口を用意する。

### A.5 残課題（次期検討・実装側決定事項）

1. **安全条件（副作用残存なし）の検証インターフェース**：副作用確認・環境復旧完了の具体的証明方式（チェックリスト・外部ヘルスチェック結果・署名付き確認記録等）は運用・Adapter 実装側で具体化する。
2. **Ticket テンプレートおよび標準 Job 列の交換形式**：テンプレート仕様の JSON スキーマ標準化が必要になった場合、後続マイナー版で任意拡張として検討する。
3. **1.0.0 データの移行ツール**：既存 1.0.0 レコード（複数 `assignee` を含む Ticket や `Ticket.handoffs`）を 1.1.0 へ機械変換する場合の分割・移行ルールは、移行対象ストレージ実装ごとに定義する。

---

## C. データモデル差分

### C.1 エンティティ別 1.0.0 → 1.1.0 差分一覧

| エンティティ | 1.0.0 構成 | 1.1.0 構成（追加・変更・削除） | 役割の整理 |
|---|---|---|---|
| **Worker** | （未定義） | **新設**：`id`（必須・不変）, `name`（必須・変更可）。種別・`owner_type` 禁止 | 作業の責任主体。認証・環境設定は外部 Adapter 側 |
| **Session** | （未定義） | **新設**：`id`（必須）, `worker_id`（必須）, `started`（任意）, `ended`（任意） | 実際の作業実行を識別する単位。同一 Worker が複数保持可 |
| **Project** | `id`, `name`, `knowledge`（`id`, `status`: `active`/`supersed`, `content`, `basis`, `supersedes`） | **変更なし**（`active`/`supersed` 綴り完全維持） | 継続領域と長期知識の正本 |
| **Task** | `id`, `project`, `objective`, `criteria`, `deadline`, `status`, `resume` | **維持＋責務明確化**：長寿命成果目標。揮発性単一スロット `Handoff` が所属（別レコード時は `Handoff.task_id` で結合） | 複数 Ticket の統括・結果集約・最終達成判定 |
| **Ticket** | `id`, `task`, `objective`, `jobs`, `assignee`, `depends`, `handoffs`, `status`, `resume` | **追加**：`owner_id`（必須・不変）, `refs`（任意：情報共有参照）, `artifacts`（任意）, `cancel_reason`（任意、`canceled` 時必須記録）<br>**廃止**：`assignee`, `handoffs` | 単一 Worker 担当・一方向進行の短命作業単位 |
| **Job** | `id`, `objective`, `criteria`, `instructions`, `assignee`, `status`, `result`, `resume` | **維持**：`id`, `objective`, `criteria`, `instructions`, `status`, `resume`<br>**明確化**：`result`（`status` と分離した業務判定・エラー結果）, `artifacts`（任意）<br>**廃止**：`assignee` | `criteria` で完了判定できる最小実行単位。複雑な構造を持たない |
| **Run** | `id`, `job`, `actor[]`, `started`, `ended`, `result`, `generation`, `activity`, `observed` | **追加**：`ticket_id`（必須）, `owner_id`（必須）, `session_id`（必須）, `wait`（任意：`job_id`, `target`, `condition`, `since`, `resume_point`, `timeout`）<br>**廃止**：`actor[]`<br>**必須化**：`generation` | Job の1回の実行試行（Execution / Attempt）および待機・観測の正本 |
| **Execution Claim** | （未定義；Lease 推奨のみ） | **新設**：`task_id`（必須）, `ticket_id`（必須）, `owner_id`（必須）, `session_id`（必須・nullable）, `generation`（必須・単調増加整数）, `state`（必須：`unassigned`/`active`/`released`/`revoked`）, `acquired`（任意）, `revoked`（任意） | Ticket（作業系列）単位の実行権の正本（`generation` フェンシング） |
| **Handoff** | `Ticket.handoffs: array<string>` | **全面再設計（Task 所属揮発性単一スロット）**：`task_id`（必須）, `current_ticket_id`, `owner_id`, `active_session_id`, `current_job_id`, `activity`, `context`, `stop_position`, `blockers[]`, `notes[]`, `updated` | 閲覧専用の作業文脈キャッシュ（揮発性・正本でない・実行権なし） |

### C.2 `Run` と `Execution` / `Attempt` の対応整理

TTJ 1.1.0 では、独立した `Execution` オブジェクトおよび `Attempt` オブジェクトを導入しない。

| 概念 | TTJ 1.1.0 での対応先 | 設計根拠 |
|---|---|---|
| 実行試行（Attempt / Execution） | **`Run`** | 1回の Job 実行・再試行・再起動ごとに1件の `Run` を作成する。待機制御（`wait`）とプロセス生存観測（`activity`, `observed`）も同一試行に属するため `Run` に統合するのが自然 |
| 実行権・排他・フェンシング | **`Execution Claim`** | 実行権は複数回の `Run` や巻き取りを跨いで作業系列（Ticket）単位で管理する必要があるため、`Run` から分離して正本化し、`generation` を各 `Run` に刻む |
| 作業文脈の申し送り | **`Handoff`** | 実行履歴（永続）と現在状況キャッシュ（揮発）のライフサイクルが異なるため分離 |

---

## D. 状態遷移

### D.1 作業状態（`status`）の遷移表（Task / Ticket / Job）

| 現在状態 | 許可される遷移先 | ガード条件・主な規則 |
|---|---|---|
| `todo` | `doing`, `waiting`, `paused`, `blocked`, `canceled` | 初期値。`doing` 時は開始記録を同時確定し、Job 開始時は Ticket と `todo` の Task も同時に `doing` にする |
| `doing` | `waiting`, `paused`, `blocked`, `completed`, `canceled` | 有効な `Execution Claim`（最新 `generation`）を持つ Session のみ更新可 |
| `waiting` | `todo`, `doing`, `paused`, `blocked`, `completed`, `canceled` | 計画内の外部結果待ち（対象や Worker 種別で分岐しない）。解除後は現在の未完了 Job のみから一方向続行。待機結果の証拠で `criteria` 成立なら `completed` 可 |
| `paused` | `todo`, `doing`, `waiting`, `blocked`, `canceled` | 明示的な保留。再開には明示解除と残存課題の再評価が必要。保留前状態の無条件復元禁止。Task/Job の `paused → completed` 禁止 |
| `blocked` | `todo`, `doing`, `waiting`, `paused`, `canceled` | 計画外の継続不能課題。実課題の解消確認で復帰（別 Ticket 完了のみでの自動解除禁止）。Task/Job の `blocked → completed` 禁止 |
| `completed` | **なし（終端）** | 再開禁止。修正・再実行は新 Ticket で対応 |
| `canceled` | **なし（終端）** | 再開禁止。中止理由（`cancel_reason` 等）と実行権失効を同時確定。Ticket 自身の中止は全 Job 終端による `completed` 集約より優先 |

- **Ticket の集約例外**：Ticket 自身が `canceled` でなければ、全 Job が `completed` または `canceled` になった時点で（未着手・保留中・ブロック中からでも）`completed` へ集約する。

### D.2 `Job.status` と `Job.result` の分離例

| シナリオ | `Job.status` | `Job.result` | 後続の扱い |
|---|---|---|---|
| 通常実装 Job が `criteria` 達成 | `completed` | `passed`（または成果概要） | 次の Job へ進む |
| レビュー Job 依頼送信直後（未回答） | `waiting` | （未確定） | `Run.wait` で待機。送信だけでは完了しない |
| レビュー Job が差し戻し証拠を受領 | `completed` | `changes_requested` | 完了済み Job は巻き戻さず、修正用新 Ticket を作成 |
| テスト Job が全項目を実行し失敗を確認 | `completed` | `failed` | テスト実行 Job 自体は完了。修正・再テストは新 Ticket |
| 複合スクリプトが異常終了し継続不能 | `canceled` | `error` | Ticket も `cancel_reason` 付きで `canceled`。安全確認後に新 Ticket で再実行 |

### D.3 `Run.activity`（生存観測）の遷移と停止確認

| 現在値 | 許可される遷移先 | 規則 |
|---|---|---|
| `unstarted` | `alive`, `unknown`, `stopped` | 起動前。起動後に `unstarted` へ戻すことは禁止 |
| `alive` | `unknown`, `stopped` | 生存観測有効期間が切れた場合（Heartbeat 途絶・監視タイムアウト等）は `stopped` ではなく必ず **`unknown`** へ遷移する |
| `unknown` | `alive`, `stopped` | 生存再確認で `alive`、信頼できる停止確認で `stopped` |
| `stopped` | **なし（同一 Run の終端）** | `stopped → alive`・`stopped → unknown` 禁止。再起動は新しい `Run` を作成する |

### D.4 `Execution Claim` の遷移と `generation` フェンシング

```text
unassigned (session=null, gen=0)
   │  [新規取得: CAS gen 0→1]
   ▼
active (session=S1, gen=1)
   ├── [正常完了・解放: gen 1→2] ──→ released (session=null, gen=2)
   ├── [中止・別Worker巻き取りで失効: CAS gen 1→2] ──→ revoked (session=null, gen=2)
   └── [同一Workerの別Session S2へ安全巻き取り: CAS gen 1→2] ──→ active (session=S2, gen=2)
```

- **stale Run / 旧 `generation` 拒否**：
  1. 状態遷移・Job 完了・結果記録・Handoff 更新要求は `(ticket_id, session_id, generation)` を提示する。
  2. TTJ Core は現行 `Execution Claim` の `state == "active"` かつ `claim.session_id == session_id` かつ `claim.generation == generation` を検証する。
  3. 1つでも不一致なら**無条件に拒否**し、状態・結果・Handoff を一切変更しない（必要に応じて無効試行として監査記録のみ残す）。
  4. 外部副作用を伴う操作では、ゲートウェイ・実行環境側でも旧 `generation` を拒否する。

### D.5 巻き取り可否マトリクスと手順

| 対象 Ticket / Session の状態 | Handoff 参照 | 巻き取り可否 | 条件・補足 |
|---|---|---|---|
| 未割当（`Claim: unassigned` かつ稼働 Run なし） | 可 | **可** | 終端 Ticket は不可。`paused` は明示解除が必要 |
| 稼働中（`activity: alive`） | 可 | **不可** | 稼働中 Session からの奪取禁止（UC-17） |
| `waiting` かつ稼働中（`status: waiting` & `activity: alive`） | 可 | **不可** | 待機中でも Session 生存中は奪取禁止 |
| 停止確認済み（`activity: stopped`、`waiting` 中含む） | 可 | **安全条件成立時のみ可** | 副作用残存の疑いがあれば安全確認・復旧完了まで保留（UC-18, UC-25） |
| 状態不明（`activity: unknown`、Heartbeat 途絶含む） | 可 | **不可** | 停止未確認のまま巻き取り禁止（UC-19） |

**原子的巻き取りフロー（同時要求は最大1件成功：UC-20）**：
1. Handoff と現行 `Ticket` / `Run` / `Execution Claim` を参照する。
2. 現行 `Run.activity` を確認：`alive` または `unknown` なら即拒否。
3. `stopped` 確認かつ副作用残存なし等の安全条件成立を確認（未確認なら拒否・保留）。
4. 現行 `Execution Claim` に対し、期待 `generation = g` を条件とする原子的 CAS を実行する：
   - **別 Worker への巻き取り（または障害で新 Ticket 再実行）**：旧 Claim を `state = "revoked"`, `session_id = null`, `generation = g + 1` に原子更新（競合する2件目以降は `generation` 不一致で即失敗）。必要なら旧 Ticket を `cancel_reason` 付きで `canceled` とし、新 Worker を `owner_id` とする新 Ticket（`refs` に旧 Ticket を記録）を作成して新 Claim（`generation = 1`）を取得し実行開始する。旧 Ticket の `owner_id` は変更しない。
   - **同一 Worker の別 Session による安全な再接続継続**：旧 Claim を `session_id = S_new`, `generation = g + 1`, `state = "active"` に原子更新し、新 `Run`（`generation = g + 1`）で現在の未完了 Job から一方向に続行する。

### D.6 Handoff の更新・消失・並列 Ticket との関係

- **更新**：有効な `Execution Claim` を保持する Session、または正規の管理・再生成処理のみが Task の単一 `Handoff` スロットを更新できる。旧 `generation` からの更新は拒否する。
- **消失・破棄**：`Handoff` が消失しても、正本である `Task`・`Ticket`・`Job`・`cancel_reason`・`artifacts`・`Run`・`Execution Claim`・監査履歴はすべて維持され、正本データから `Handoff` を再生成できる（UC-22）。
- **並列実行**：同一 Task 内に独立した複数 Ticket（例：Ticket A と Ticket B）が並列に `Execution Claim` を保持し稼働する場合でも、Task の `Handoff` スロットが1つであることを理由に実行を直列化・ブロックしない（UC-24）。
