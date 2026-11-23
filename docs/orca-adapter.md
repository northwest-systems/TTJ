# Orca 補足設計・Adapter 設計（非規範）

> [!IMPORTANT]
> 本文書は `skills/ttj/` の**外**に配置する外部 Adapter 向けの補足設計資料（提出物 F）である。**TTJ Core（規格本文・JSON 参照形式・Skill および Core 公開契約）には Orca 固有の型・ID・状態遷移・接続処理を一切組み込まない。** TTJ Core は Orca が存在しない環境でも単独で完全に成立する。

---

## 1. 境界原則（TTJ Core 非依存の保証）

TTJ 1.1.0 の禁止事項 #18 に従い、責務を次の2層に厳格分離する。

| 層 | 責務 | 禁止事項 |
|---|---|---|
| **TTJ Core** | Project / Task / Ticket / Job / Run / Worker（ID 接頭辞 `pj-` / `tsk-` / `tkt-` / `job-` / `run-` / `wkr-`） / Session / Execution Claim / Handoff / Knowledge のデータモデル、公開契約、状態遷移、不変条件、`generation`/`run_seq` フェンシング | Orca 専用 Run ID・Orchestration Task ID・Session 型・Bot 種別・Orca 固有状態遷移・API 接続処理をコア仕様やコア公開契約へ組み込むこと |
| **Orca Adapter（外部層）** | TTJ 概念と Orca 概念の対応付け（`ExternalBinding`）、認証・実行環境設定の保持、Dispatch 起動・独立監視、ゲートウェイ側での旧 `generation`・閉じた `Run.id` 失効強制、Handoff 参照 UI 連携 | TTJ の業務完了判定や巻き取り可否判定をバイパスすること、Orca の稼働表示のみで安全巻き取りを許可すること |

---

## 2. 概念対応関係（TTJ Core ↔ Orca）

| TTJ 1.1.0 概念 | Orca 側の対応概念 | 補足・注意点 |
|---|---|---|
| `Task` | Orca `Run`（長寿命の目標実行単位） | 1つの長寿命目標に対し複数 Ticket がぶら下がる。Ticket 増加で Orca `Run`（TTJ `Task`）を増殖させない。Run を持たない Task 単位の外部承認待ち（`waiting`）も管理・証拠取り込み経路で扱う |
| `Ticket` | Orca `Orchestration Task` | 単一 Worker が責任を持つ一方向の作業単位。別 Worker への依頼や修正・再実行は新しい `Orchestration Task`（新 Ticket）として起票する |
| `Job` | `Orchestration Task` 内の作業工程（ステップ） | Ticket 確定後は固定列。`status` と `result` を分離し、`criteria` に基づく証拠で完了判定する。Job 単独中止時は確定と同時に結合中 `Run` の試行権限を即時無効化する |
| `Run`（TTJ 実行試行） | Orca `Dispatch` | Job の1回の実行試行に対応。待機制御情報（`wait`、解消時 `resolved`）と観測状態（`activity`, `observed`）を保持する。終了確定・単独中止・安全閉鎖時にその `Run.id` の試行権限を即座に閉じる（`active_run_id = null`, `run_seq +1`） |
| `Worker` | 外部識別子（External Worker ID） | 人間オペレーター・既存 Orca Bot・Orchestration Worker のいずれであっても TTJ 上は同一の `Worker`（`id`: `wkr-*`, `name`）として扱い、`owner_type` 等で業務分岐しない |
| `Session` | Orca `Session` | 同一 Worker に複数 Orca Session が存在し得るが、TTJ 上は個別の `Session.id` として識別し、同一作業系列の実行権を共有させない。終了済み Session には新 `active` Claim を決して付与しない |
| `Execution Claim` | TTJ Core が Ticket 単位（キー：`ticket_id`）で正本を保持 | `generation`（初期値 `1`、切替1回ごとに `+1`、非巻戻し）、現行試行 `active_run_id`、実行系列版 `run_seq`（初期値 `0`、結合/無効化/閉鎖ごとに `+1`）を保持。Orca 側ディスパッチャ／ゲートウェイは失効済み権限からの外部副作用を確定側で必須遮断する |
| `Handoff` | TTJ Core（Task 所属の揮発性単一スロット） | Orca UI・コーディネータ・他 Session は TTJ Core の Handoff（更新 3 経路分離・`source_revision` 鮮度およびスロットインスタンス照合付き）を参照・表示するのみ。Handoff 閲覧は実行権を意味せず、再開に必須な条件は正本へ残す |

---

## 3. 実行フローと完了・更新照合の境界

### 3.1 Dispatch 完了 ≠ 無条件の TTJ 業務完了

Orca の `Dispatch`（TTJ `Run`）がプロセス終了コード 0 やメッセージ送信完了で終了しても、それだけで TTJ の `Job` や `Ticket` を `completed` にしてはならない。

1. **実行開始（開始前照合と原子的試行結合）**：Orca Adapter は TTJ Core に対し、生存する適格 Session（`Session.worker_id == Ticket.owner_id`）を用いて `active` な `Execution Claim`（現行 `session_id`・最新 `generation`）、対象現在 Job、開始ガード（親 Task が `paused`/終端でないこと；別 Ticket 起因の `waiting`/`blocked` は開始可）、空き枠（`active_run_id == null`、または同一計画通常再試行における安全閉鎖による解放）、および**実行系列前提（`expected_run_seq`）の一致**を検証した上で、新しい TTJ `Run` を作成して `claim.active_run_id` へ原子的に結合し（`claim.run_seq +1`）、`Dispatch` を起動する。
2. **待機（`waiting`）**：`Dispatch` 内でレビュー・承認・外部操作・別 Ticket の結果待ちが発生した場合、Job を `waiting` とし、待機制御情報（`job_id`, `target`, `condition`, `since`, `resume_point`, `timeout`）を TTJ `Run.wait` へ記録する（`depends` と非終端の未解決 `Run.wait`（`paused`/`blocked` 保持中含む）を合わせた循環依存を禁止。解消時は `resolved` を記録して辺から外す）。
3. **証拠確認と完了判定（開始後照合と後着証拠経路の分離）**：確認可能な成果物・ログ・判定記録が `Job.criteria` をすべて満たした時点で、次のいずれかの正当な経路で `Job.status = "completed"` および `Job.result`（例：`approved`, `changes_requested`, `passed`, `failed`）を確定する。
   - **結合済み Run からの確定（開始後照合）**：認証済み `Session.worker_id == Ticket.owner_id`、`claim.state == "active"`、`claim.session_id == session_id`、`claim.generation == generation`、対象現在 Job、および**現行 Run の非 `null` 一致（`claim.active_run_id == run.id`）**をすべて照合する（同世代でも閉じた旧 `Run.id` は拒否）。
   - **Run 終了後の後着証拠取り込み（`waiting` / `doing` 共通）**：`Dispatch` 終了により当該 `Run.id` の試行権限が閉じた（`claim.active_run_id == null`）後に、現在 Job が `waiting` または `doing` の状態で確認可能な外部応答・証拠が到着した場合は、旧 `Run` 名義ではなく、現行 `active` Claim を保持する認証済み Session または TTJ Core が認可する証拠取り込み経路（終端保護・現在 Job 照合・監査記録必須）で評価・確定する。
4. **Job 単独中止と修正・再実行**：
   - 特定 Job を単独で `canceled` にする場合は、中止確定と同時に結合中 `Run` の試行権限を即座に無効化・閉鎖（ゲートウェイ効果確定側で即時拒否可能、`active_run_id = null`, `run_seq + 1`）し停止要求を発行した上で、後続 Job 開始前に停止確認または安全条件を別途確認する。
   - `result` が `changes_requested` や `failed` の場合、または `Dispatch` 異常終了で継続不能となり Ticket を `canceled`（`cancel_reason` 必須保存）にした場合、元 Ticket は再開せず、同一 TTJ `Task`（Orca `Run`）配下に新 Ticket（新 `Orchestration Task`、`refs` 付き）を作成して対応する。

---

## 4. Handoff 参照・独立監視・TTJ Core 経由の巻き取り

### 4.1 稼働表示・生存観測と安全巻き取りの分離

- **Orca 側の稼働表示・Heartbeat 途絶だけで安全巻き取りを許可しない**。
  - Orca の UI 上で「オフライン」「無応答」「Heartbeat 途絶」と表示されていても、TTJ Core 上の `Run.activity` は **`unknown`** であり、`stopped`（停止確認済み）ではない。
  - また、中止等で Claim が `revoked`（`session_id: null`）になっていても、過去 Run が `alive` または `unknown` のまま残存している場合は「未割当」とみなさず、停止確認と安全確認のガードを優先する。
  - 巻き取り可否の判定と単一の原子的確定（`(expected_generation, expected_run_seq)` の一致検証による陳腐化安全確認の排除、および親 Task 中止・旧 Job 証拠確定との線形化を含む）は、必ず **TTJ Core の `Execution Claim` と巻き取り契約** を経由して行う。
- **停止確認（`stopped`）と副作用・失効前開始済み操作の安全確認**：
  - 独立した監視機構（Liveness Observer）が、当該 `Dispatch`（`Run`）の実行主体および配下の実行コンテキストが終了し新規副作用を開始できないことを確認した時点で、専用の観測記録経路により `Run.activity = "stopped"` と `observed`・停止証拠を記録する（この専用経路は Claim 失効後・Run 終了後でも使用可能だが、業務状態・結果・Handoff は一切変更しない）。
  - 障害注入や外部リソース変更など副作用の可能性がある作業、および **Claim 失効前・Job 単独中止前に外部側で開始済みだが結果未確認の操作** が残存し得る場合は、停止確認とは別に完了・無効化・環境復旧を含む安全確認が完了するまで、Adapter は後続の巻き取り・後続 Job・再実行 Dispatch を保留する。

### 4.2 既存 Orca Bot と Orchestration Worker の差異への注意

Orca 環境では、常駐型の既存 Orca Bot と、オンデマンドに起動される Orchestration Worker とで Session の寿命・接続方式・プロセス監視方法・認証トークン形式が異なる場合がある。

- **Adapter 側で吸収する差異**：認証方式、Session 起動・停止確認の実装手段、Heartbeat の監視機構、外部ゲートウェイへの `(ticket_id, session_id, generation, run_id)` 伝播方法。
- **TTJ Core へ持ち込まない事項**：既存 Orca Bot か Orchestration Worker かを示す `owner_type` や種別フラグ、種別ごとの Job 状態遷移分岐、種別ごとの完了判定ルール。
- いずれの主体であっても、同一 Worker であっても、**実行権（`active` Claim）を持たない Session からの無断実行は拒否**される。別 Worker へ巻き取る場合は、未確定の中間状態を外部公開しない単一の原子的確定により、旧 Ticket の `owner_id` を変えず `canceled` 終端化・旧 Claim `revoked`（再取得永久禁止・二重失効なし）・後続 Ticket と初期 Claim 確定・再送用永続結果を一体確定する。終了済み Session には新 `active` Claim を決して付与せず、確定済み世代も巻き戻さない。

---

## 5. `ExternalBinding` と Adapter 設計

### 5.1 `ExternalBinding` の役割

TTJ Core のレコードに Orca 固有の ID や認証情報を直接埋め込まず、Adapter 側が管理する `ExternalBinding` テーブル／ドキュメントで TTJ ID（`pj-*` / `tsk-*` / `tkt-*` / `job-*` / `run-*` / `wkr-*` 等）と外部 ID を対応付ける。

```json
{
  "binding_id": "bind-orca-001",
  "adapter": "orca",
  "ttj_entity": "ticket",
  "ttj_id": "tkt-revise-invitation-02",
  "external_entity": "orchestration_task",
  "external_id": "orca-otask-9f31c",
  "metadata": {
    "orca_run_id": "orca-run-42",
    "worker_external_ref": "orca-worker:alpha",
    "endpoint_profile": "local-orchestrator"
  },
  "updated": "2026-01-01T03:00:00Z"
}
```

> 注：上記 JSON は **Orca Adapter 内部の外部バインディング表現**であり、TTJ Core の仕様レコードではない（そのため TTJ の `protocol` を持たない）。

### 5.2 Adapter の主要責務と API 境界

1. **Identity Binding**：Orca の利用者・Bot・Orchestration Worker の外部識別子を TTJ `Worker.id`（`wkr-*`）に、Orca Session を TTJ `Session.id` にマッピングする。認証情報・資格情報は Adapter 側のみで管理する。
2. **Claim & External Fencing Proxy（準拠必須）**：
   - Dispatch 開始前に TTJ Core の開始前照合で `Execution Claim`（`generation`, `active_run_id == null`, `expected_run_seq`）を検証し、新 `Run.id` を `claim.active_run_id` へ原子的に結合する（`run_seq +1`）。
   - 実行中のツール呼び出し・状態更新・外部リソース更新ゲートウェイに `(ticket_id, session_id, generation, job, run_id)` を添付し、**効果の確定側でも失効済み権限（旧 `generation`・失効済み Claim・閉じた同世代 `run_id`・Job 単独中止で即時無効化された `run_id`）を必須拒否**する。
   - 失効前・単独中止前に認可・開始された外部操作が残存し得る場合は、安全確認完了まで後続実行を保留する。
3. **Independent Liveness Observer**：
   - Orca ランタイム・OS プロセス・コンテナ監視から独立した観測結果を収集し、有効期間内の生存は `alive`、Heartbeat 途絶・通信断・有効期間超過は `unknown`、確実な終了確認のみ `stopped`（終端観測値）として TTJ Core の専用観測記録経路へ報告する。
4. **Takeover Coordinator**：
   - 巻き取り要求時は必ず TTJ Core の単一原子確定契約（確定済み同一 `operation_id`・同一主体・同一実質パラメータの再送冪等返却、主体/パラメータ不一致拒否、別要求拒否、全過去 Run の `stopped` 確認・安全確認・`(expected_generation, expected_run_seq)` 検証、親 Task 中止・旧 Job 証拠確定との線形化、終了済み Session への新 `active` Claim 非付与）を呼び出し、途中状態を外部へ公開せずに競合を TTJ Core で解決する。
5. **Read-Only Handoff Projection**：
   - Orca ダッシュボードや他 Session 向けに TTJ Core の Task 揮発性 `Handoff` を読み取り専用で表示する。Handoff ビューから直接作業実行やコマンド送信を行わせず、更新は Run 起因更新・Run 非紐付け Session 更新・正規管理再生成（書込み時点の `source_revision` 最新性検証・実行権非付与）の 3 経路を厳格分離する。

---

## 6. Orca 非利用環境での独立成立性

TTJ Core（`skills/ttj/` 配下の仕様・JSON 参照形式・Skill および Core 公開契約）には Orca 固有の型・ID・接続・状態遷移への依存が一切存在しないため、以下のいずれの環境でもそのまま完全準拠で運用できる。

- **人間のみの運用**：ローカルの JSON ファイルや課題管理ツール上で、人間が `Worker`・`Session`（作業枠）・`Task`・`Ticket`・`Job`・`Run`・`Handoff` を運用する。
- **別オーケストレーター／CI 基盤**：GitHub Actions、Temporal、社内ワークフローエンジン等に対し、各自の薄い Adapter と `ExternalBinding` を実装して TTJ Core へ接続する。
- **単体 LLM / CLI 運用**：`skills/ttj/` をそのまま利用環境へ配置し、単一または複数の CLI セッション間で TTJ 1.1.0 規則に従った計画・実行・引き継ぎを行う。

いずれの環境でも、TTJ Core のデータモデル、状態遷移、禁止事項 1〜18、および受け入れテスト UC-01〜UC-27・UC-S01〜UC-S06 は Orca なしで同一に成立する。
