# Orca 補足設計・Adapter 設計（非規範）

> [!IMPORTANT]
> 本文書は `skills/ttj/` の**外**に配置する外部 Adapter 向けの補足設計資料（提出物 F）である。**TTJ Core（規格本文・JSON 参照形式・Skill）には Orca 固有の型・ID・状態遷移・接続処理を一切組み込まない。** TTJ Core は Orca が存在しない環境でも単独で完全に成立する。

---

## 1. 境界原則（TTJ Core 非依存の保証）

TTJ 1.1.0 の禁止事項 #18 に従い、責務を次の2層に厳格分離する。

| 層 | 責務 | 禁止事項 |
|---|---|---|
| **TTJ Core** | Project / Task / Ticket / Job / Run / Worker / Session / Execution Claim / Handoff / Knowledge のデータモデル、状態遷移、不変条件、`generation` フェンシング | Orca 専用 Run ID・Orchestration Task ID・Session 型・Bot 種別・Orca 固有状態遷移・API 接続処理をコア仕様やコアコードへ組み込むこと |
| **Orca Adapter（外部層）** | TTJ 概念と Orca 概念の対応付け（`ExternalBinding`）、認証・実行環境設定の保持、Dispatch 起動・監視、ゲートウェイ側での旧 `generation` 失効強制、Handoff 参照 UI 連携 | TTJ の業務完了判定や巻き取り可否判定をバイパスすること、Orca の稼働表示のみで安全巻き取りを許可すること |

---

## 2. 概念対応関係（TTJ Core ↔ Orca）

| TTJ 1.1.0 概念 | Orca 側の対応概念 | 補足・注意点 |
|---|---|---|
| `Task` | Orca `Run`（長寿命の目標実行単位） | 1つの長寿命目標に対し複数 Ticket がぶら下がる。Ticket 増加で Orca `Run`（TTJ `Task`）を増殖させない |
| `Ticket` | Orca `Orchestration Task` | 単一 Worker が責任を持つ一方向の作業単位。別 Worker への依頼や修正・再実行は新しい `Orchestration Task`（新 Ticket）として起票する |
| `Job` | `Orchestration Task` 内の作業工程（ステップ） | Ticket 確定後は固定列。`status` と `result` を分離し、`criteria` に基づく証拠で完了判定する |
| `Run`（TTJ 実行試行） | Orca `Dispatch` | Job の1回の実行試行に対応。待機制御情報（`wait`）と観測状態（`activity`, `observed`）を保持する |
| `Worker` | 外部識別子（External Worker ID） | 人間オペレーター・既存 Orca Bot・Orchestration Worker のいずれであっても TTJ 上は同一の `Worker`（`id`, `name`）として扱い、`owner_type` 等で業務分岐しない |
| `Session` | Orca `Session` | 同一 Worker に複数 Orca Session が存在し得るが、TTJ 上は個別の `Session.id` として識別し、同一作業系列の実行権を共有させない |
| `Execution Claim` | TTJ Core が正本を保持（Adapter はトークン/`generation` を検証・伝播） | 実行権の正本は常に TTJ Core に置く。Orca 側ディスパッチャ／ゲートウェイは失効済み `generation` からの外部副作用を遮断する |
| `Handoff` | TTJ Core（Task 所属の揮発性単一スロット） | Orca UI・コーディネータ・他 Session は TTJ Core の Handoff を参照・表示するのみ。Handoff 閲覧は実行権を意味しない |

---

## 3. 実行フローと完了判定の境界

### 3.1 Dispatch 完了 ≠ 無条件の TTJ 業務完了

Orca の `Dispatch`（TTJ `Run`）がプロセス終了コード 0 やメッセージ送信完了で終了しても、それだけで TTJ の `Job` や `Ticket` を `completed` にしてはならない。

1. **実行開始**：Orca Adapter は TTJ Core に対し、`Ticket.owner_id` と一致する Worker の Orca Session を用いて `Execution Claim`（最新 `generation`）を検証・取得した上で、新しい TTJ `Run` を作成し `Dispatch` を起動する。
2. **待機（`waiting`）**：`Dispatch` 内でレビュー・承認・外部操作・別 Ticket の結果待ちが発生した場合、Job を `waiting` とし、待機制御情報（`target`, `condition`, `since`, `resume_point`, `timeout`）を TTJ `Run.wait` へ記録する。
3. **証拠確認と完了判定**：確認可能な成果物・ログ・判定記録が `Job.criteria` をすべて満たした時点で初めて、現行 `generation` を提示して `Job.status = "completed"` および `Job.result`（例：`approved`, `changes_requested`, `passed`, `failed`）を確定する。
4. **修正・再実行**：`result` が `changes_requested` や `failed` の場合、または `Dispatch` 異常終了で継続不能となり Ticket を `canceled`（`cancel_reason` 保存）にした場合、元 Ticket は再開せず、同一 TTJ `Task`（Orca `Run`）配下に新 Ticket（新 `Orchestration Task`、`refs` 付き）を作成して対応する。

---

## 4. Handoff 参照と TTJ Core 経由の巻き取り

### 4.1 稼働表示と安全巻き取りの分離

- **Orca 側の稼働表示・Heartbeat 途絶だけで安全巻き取りを許可しない**。
  - Orca の UI 上で「オフライン」「無応答」「Heartbeat 途絶」と表示されていても、TTJ Core 上の `Run.activity` は **`unknown`** であり、`stopped`（停止確認済み）ではない。
  - 巻き取り可否の判定と実行権の原子的切替は、必ず **TTJ Core の `Execution Claim` と巻き取り手順** を経由して行う。
- **停止確認と副作用確認の分離**：
  - 実行プロセス・コンテナ・リモートジョブの実際の停止を信頼できる監視機構で確認して初めて `Run.activity = "stopped"` を記録する。
  - 障害注入や外部リソース変更など副作用の可能性がある作業では、停止確認に加えて環境復旧・安全条件の成立を確認するまで、Adapter は後続の巻き取り・再実行 Dispatch を保留する。

### 4.2 既存 Orca Bot と Orchestration Worker の差異への注意

Orca 環境では、常駐型の既存 Orca Bot と、オンデマンドに起動される Orchestration Worker とで Session の寿命・接続方式・プロセス監視方法・認証トークン形式が異なる場合がある。

- **Adapter 側で吸収する差異**：認証方式、Session 起動・停止確認の実装手段、Heartbeat の監視機構、外部ゲートウェイへの `generation` 伝播方法。
- **TTJ Core へ持ち込まない事項**：既存 Orca Bot か Orchestration Worker かを示す `owner_type` や種別フラグ、種別ごとの Job 状態遷移分岐、種別ごとの完了判定ルール。
- いずれの主体であっても、同一 Worker であっても、**実行権（有効な `Execution Claim`）を持たない Session からの無断実行は拒否**され、別 Worker が巻き取る場合は旧 Ticket の `owner_id` を変えずに新 Ticket を作成する。

---

## 5. `ExternalBinding` と Adapter 設計

### 5.1 `ExternalBinding` の役割

TTJ Core のレコードに Orca 固有の ID や認証情報を直接埋め込まず、Adapter 側が管理する `ExternalBinding` テーブル／ドキュメントで TTJ ID と外部 ID を対応付ける。

```json
{
  "binding_id": "bind-orca-001",
  "adapter": "orca",
  "ttj_entity": "ticket",
  "ttj_id": "revise-invitation-02",
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

1. **Identity Binding**：Orca の利用者・Bot・Orchestration Worker の外部識別子を TTJ `Worker.id` に、Orca Session を TTJ `Session.id` にマッピングする。認証情報・資格情報は Adapter 側のみで管理する。
2. **Claim & Fencing Proxy**：
   - Dispatch 開始前に TTJ Core から `Execution Claim`（`generation`）を取得・検証する。
   - 実行中のツール呼び出し・状態更新ゲートウェイに `(ticket_id, session_id, generation)` を添付し、TTJ Core 側で失効した旧 `generation` からの外部書き込み・副作用をゲートウェイ側でも遮断する。
3. **Liveness Observer**：
   - Orca ランタイム・OS プロセス・コンテナ監視から独立した観測結果を収集し、有効期間内の生存は `alive`、Heartbeat 途絶・通信断は `unknown`、確実な終了確認のみ `stopped` として TTJ Core の `Run.activity` へ報告する。
4. **Takeover Coordinator**：
   - 巻き取り要求時は必ず TTJ Core の原子的巻き取り手順（停止確認・安全確認・`generation` CAS・必要に応じた旧 Ticket `canceled` と新 Ticket 作成）を呼び出し、同時要求の競合を TTJ Core で解決する。
5. **Read-Only Handoff Projection**：
   - Orca ダッシュボードや他 Session 向けに TTJ Core の Task 揮発性 `Handoff` を読み取り専用で表示する。Handoff ビューから直接作業実行やコマンド送信を行わせない。

---

## 6. Orca 非利用環境での独立成立性

TTJ Core（`skills/ttj/` および規格本文）は Orca Adapter を一切前提としないため、以下のいずれの環境でもそのまま完全準拠で運用できる。

- **人間のみの運用**：ローカルの JSON ファイルや課題管理ツール上で、人間が `Worker`・`Session`（作業枠）・`Task`・`Ticket`・`Job`・`Run`・`Handoff` を運用する。
- **別オーケストレーター／CI 基盤**：GitHub Actions、Temporal、社内ワークフローエンジン等に対し、各自の薄い Adapter と `ExternalBinding` を実装して TTJ Core へ接続する。
- **単体 LLM / CLI 運用**：`skills/ttj/` をそのまま利用環境へ配置し、単一または複数の CLI セッション間で TTJ 1.1.0 規則に従った計画・実行・引き継ぎを行う。

いずれの環境でも、TTJ Core のデータモデル、状態遷移、禁止事項 1〜18、および公式受け入れテスト UC-01〜UC-27 は Orca なしで同一に成立する。
