---
name: ttj
description: TTJ v1.1.0 に基づき、Worker と Session を分離した Task・Ticket・Job の計画、実行依存と情報共有参照、Execution Claim による排他・巻き取り、Task 所属の揮発性 Handoff、Project の長期知識を管理・点検する。TTJ の適用や準拠確認を求められた場合に使う。
metadata:
  version: "1.1.0"
---

# TTJ

- [必須] **適用**：[規格本文](references/specification.md) を正本とし、[JSON参照形式](references/json-format.md) で記入項目を確認する。
  - [禁止] Worker 種別（人間・Bot・AI 等）で業務ロジックや状態遷移を分岐させる、または `owner_type` を必須にする。
  - [禁止] 特定の Agent・Bot・オーケストレーターへの依存を TTJ Core の規則へ持ち込む。
  - [必須] ラベルはその行だけに適用し、配下の項目へ継承しない。
  - [必須] JSON を読み書きする場合はルートの `protocol` で規格名と版を確認し、出力には `"TTJ/1.1.0"` を付ける。
  - [禁止] 同じ文書の子レコードへ版情報を転記する。

## 計画

- [必須] **目標と主体確認**：Project・Task・Worker を確認する。
  - [必須] Project の `id`・`name` と Worker の `id`・`name` で対象と作業主体を識別できるようにする。
  - [必須] 既存の作業と採用中（`active`）の知識を先に読み、結果を左右する不足だけを質問する。
  - [禁止] Project をリポジトリに限定する。
- [必須] **記入**：Task・Ticket・Job すべての `objective` に「達成したいこと」を書く。
  - [必須] Task・Job の `criteria` に確認可能な条件を1件以上書く。
  - [任意] 期限が指定されている場合だけ、Task の `deadline` にタイムゾーン付き ISO 8601 日時を記録する。
  - [禁止] 未指定の期限を作る、または期限を Ticket・Job へ転記する。
  - [必須] Ticket には存在する単一の `Worker.id` を `owner_id` に指定し、`jobs` に Job の ID を実行順に並べる。
  - [禁止] Ticket に複数 Owner を設定する、Job ごとに個別 Owner を設定する、旧 `assignee` や `Ticket.handoffs` を使う、Ticket 独自の `criteria` を設ける。
  - [任意] 手順が不足する場合だけ Job の `instructions` を補う。
  - [禁止] Job に複雑なワークフロー構造・子 Job・待機制御構造・独立 `Request` オブジェクト・種別分類を追加する。
  - [禁止] 同じ目標・条件・結果を別の欄へ転記する。
- [必須] **正式作成**：Ticket と全 Job を一括確定する。
  - [必須] Job 列で Ticket の目標を達成できる計画か確認する。
  - [任意] 正式作成前の Draft や Ticket テンプレートを編集する。
  - [必須] 初期 `status` を `todo` にし、確定後は Job 列と作業定義を変更しない。
  - [禁止] 修正・再実行・再レビューによる Ticket 増加を理由に新しい Task を増殖させる。
- [任意] **実行依存と情報共有参照**：`depends` と `refs` を明確に区別して記録する。
  - [任意] 先に `completed` である必要がある前提 Ticket だけを `depends` に指定する。
  - [必須] `depends` の参照存在と循環依存がないことを確認する。
  - [任意] 前 Ticket の成果物・結果・中止理由・派生元・修正対象・再実行元は情報共有専用の `refs` に記録する。
  - [禁止] `refs` を開始条件や完了依存として扱う。
- [必須] **保存**：既存の手段で情報と参照の整合性を維持する。
  - [任意] 各階層を別形式・別ファイルで保存する。
  - [禁止] 専用ツール・データベース・追加ファイルを一律に導入する。
  - [必須] 履歴・原子性・排他・外部副作用防止を保証できない場合は未対応範囲を示す。

## 実行と終了

- [必須] **開始前確認**：依存・順序・状態・所有権・実行権・安全条件を確認する。
  - [必須] 前提 Ticket が `completed`、先行 Job が `completed` または `canceled`。
  - [必須] Job・Ticket が `todo` または `doing`、Task が保留中・終端ではない。
  - [必須] 実行する Session が `Ticket.owner_id` と同じ Worker に属し、当該 Ticket の有効な `Execution Claim`（最新 `generation`）を保持している。
  - [禁止] 同じ Worker であることを理由に、実行権のない別 Session による無断実行を許可する。
  - [必須] 後続 Job や再試行では、前の実行の停止確認と副作用重複を防ぐ安全条件を確認する。
- [必須] **Run 記録**：Job の1回の実行試行を `Run`（`id`・`ticket_id`・`job`・`owner_id`・`session_id`・`generation`・`started` 等）に記録する。
  - [禁止] 同じ Ticket の別 Job を同時実行する、または同じ試行の再送を二重登録する。
  - [必須] 再試行・再起動には新しい `Run` を使い、待機・実行制御情報（`wait`）は Job ではなく `Run` 側で保持する。
  - [禁止] 依頼送信や `Run` の終了だけで Job を完了する。
  - [禁止] 古い `Run` や旧 `generation` からの状態更新を適用する。
- [必須] **Job の終了と status / result 分離**：確認可能な証拠に基づき完了または中止を確定する。
  - [必須] `criteria` をすべて満たせば `status` を `completed` とし、業務判定（`approved`・`changes_requested`・`passed`・`failed` 等）を `result` に分離して記録する。
  - [必須] レビュー・調査・承認・操作依頼および依頼先からの追加依頼は通常の Ticket / Job で表現する。
  - [任意] 必要な Job だけ理由を残して `canceled` にし、後続を続ける。
  - [必須] 差し戻しや失敗に応じた修正・再実行は完了済み Job を巻き戻さず新しい Ticket で対応する。
- [必須] **Ticket の終了**：全 Job が終端（`completed`・`canceled`）なら `completed` に集約する。
  - [必須] Ticket 自身がキャンセル済みなら `canceled` を維持し、`cancel_reason` を保持する。
  - [必須] Ticket 終了時に正式な成果・結果・状態を永続化し、次 Ticket が未作成でも終了済み Ticket をアーカイブできるようにする。
  - [禁止] 終端 Ticket（`completed`・`canceled`）を再開する、または Ticket 独自の完了条件を追加する。
- [必須] **Task の完了**：全 Ticket の終了に加え、Task の `criteria` と必須成果を確認する。
  - [禁止] Ticket の完了数や中止 Job を含む処理終了だけで Task の達成率・目標達成を判断する。
- [必須] **明示的な中止**：Task・Ticket の未終端の子孫も同時に `canceled` にし、対象範囲の `Execution Claim` を失効させる。
  - [必須] 自身の中止を全 Job 終端による Ticket 完了より優先し、遅延通知による復活を禁止する。

## 停止・巻き取り・課題対応

- [必須] **状態判断**：計画内の外部結果待ちは `waiting`、明示保留は `paused`、計画外の継続不能課題は `blocked` とする。
  - [必須] 待機対象の実装種類や Worker 種別で `waiting` の状態遷移を分岐させない。
  - [必須] `waiting` 解除後は現在の未完了 Job のみから一方向に続行する（複合スクリプト内の待機も同じ）。
  - [必須] 保留中（`paused`）も未解消の課題を保持し、明示解除時に再評価する。
  - [任意] `depends`・`Run.wait`・Handoff から再開条件が分からない場合だけ `resume` を1文で書く。
- [必須] **複合スクリプトと副作用**：異常終了で継続不能なら `cancel_reason` を残して Ticket を `canceled` とし、修正・再実行は新 Ticket にする。
  - [必須] 障害注入等の副作用がある作業では、Ticket 中止と環境復旧を分離し、安全確認完了まで後続の危険な実行を禁止する。
- [必須] **生存判定と巻き取り**：`Run.activity`（`unstarted`・`alive`・`stopped`・`unknown`）で判定し、Heartbeat 途絶単独で `stopped` と確定しない。
  - [禁止] 稼働中（`alive`、`waiting` かつ稼働中を含む）または状態不明（`unknown`）の Session から作業を巻き取る。
  - [必須] 停止確認済み（`stopped`、`waiting` 中の停止確認済みを含む）かつ副作用残存がない等の安全条件成立時のみ、原子的な `generation` 更新により巻き取りを許可する（同時巻き取りは最大1 Session のみ成功）。
  - [必須] 別 Worker が巻き取る場合は旧 Ticket の `owner_id` を変更せず、必要なら旧 Ticket を中止した上で新しい Worker を `owner_id` とする新 Ticket を作成する。
  - [必須] 同一 Worker が別 Session で継続する場合も停止・安全確認と原子的な `Execution Claim` 更新を経る。Session 障害で継続不能なら新 Ticket による再実行を基本とする。
- [必須] **課題対応**：ブロックされた課題の修復、別 Worker への依頼、差し戻し修正はいずれも通常の新 Ticket にし、`refs` で元 Ticket を参照する。
  - [必須] 元 Ticket が継続可能なら、実際の課題解消を確認してから再開する（別 Ticket 完了だけで自動解除しない）。

## Handoff と長期知識

- [必須] **揮発性 Handoff**：`Handoff` は `Task` 所属の単一スロットとして扱い、再開・状況把握に必要な `current_ticket_id`・`owner_id`・`active_session_id`・`current_job_id`・`activity`・`context` 等を更新する。
  - [必須] Task 参照権限を持つ複数 Session による同時閲覧を許可し、閲覧権限と実行権限を厳格に分離する。
  - [禁止] Handoff を正式履歴・成果物の正本にする、Handoff にコマンド実行機能を持たせる、または Handoff によって無関係な Ticket を直列化する。
  - [必須] Handoff が消失・破棄されても Task・Ticket・Job・中止理由・Artifact・Run・Execution Claim が失われない構成を保つ。
- [必須] **長期知識**：Project の `knowledge` に再利用する内容（`content`）と根拠（`basis`）を残す。
  - [必須] 状態は `active`・`supersed` の2値（v1.0.0 の綴りを維持）とし、新規記録は未指定でも `active` にする。
  - [必須] 予定も現在の知識として `active` に記録し、通常は `active` だけを参照し、振り返り・再発障害確認では `supersed` も読む。
  - [必須] 使わなくなった知識は理由を残して `supersed` にし、訂正は `supersedes` 付きの新規記録とする。
- [必須] **点検**：単一不変 `owner_id`、`status`/`result` 分離、`depends`/`refs` 区別、`Execution Claim` と `generation` フェンシング、巻き取り条件、揮発性 Handoff、禁止事項1〜18、Worker 非依存性を確認する。
  - [必須] 計画・実行済みの事実・管理情報・未確認事項を分けて報告する。
