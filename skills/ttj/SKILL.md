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
  - [必須] 初期 `status` を `todo` にし、論理的な初期 `Execution Claim`（`unassigned`, `generation: 1`, `run_seq: 0`）を成立させ、確定後は Job 列と作業定義を変更しない。
  - [必須] Ticket 増加への対策（テンプレート・標準 Job 列・後続 Ticket 生成支援・成果物/中止理由参照・系列表示と終了折りたたみ・Task 進捗集約と現在 Handoff 表示）を運用・実装に備える（具体的な形式や UI は選べる）。
  - [禁止] 修正・再実行・再レビューによる Ticket 増加を理由に新しい Task を増殖させる。
- [任意] **実行依存と情報共有参照**：`depends` と `refs` を明確に区別して記録する。
  - [任意] 先に `completed` である必要がある前提 Ticket だけを `depends` に指定する。
  - [必須] `depends` と**未解決の `Run.wait`** を合わせた完了依存グラフに循環がないことを確認する（解消済み・解除済み待機は履歴に保持しつつ辺から外す。逆向き競合登録は双方同時成功を禁止）。
  - [任意] 前 Ticket の成果物・結果・中止理由・派生元・修正対象・再実行元は情報共有専用の `refs` に記録する。
  - [禁止] `refs` を開始条件や完了依存として扱う。
- [必須] **保存**：既存の手段で情報と参照の整合性を維持する。
  - [任意] 各階層を別形式・別ファイルで保存する。
  - [禁止] 専用ツール・データベース・追加ファイルを一律に導入する。
  - [必須] 履歴・原子性・排他・外部副作用防止を保証できない場合は未対応範囲を示す。

## 実行と終了

- [必須] **開始前確認（開始前照合）**：依存・順序・状態・所有権・実行権・空き枠・安全条件を確認する。
  - [必須] 前提 Ticket が `completed`、先行 Job が `completed` または `canceled`。
  - [必須] 対象 Ticket および現在 Job が `todo` または `doing`、親 Task が保留（`paused`）・終端ではない（親 Task が別 Ticket 起因で `waiting`・`blocked` でも独立した実行可能 Ticket・Job は開始できる）。
  - [必須] 実行する Session が `Ticket.owner_id` と同じ Worker に属し、当該 Ticket の `active` な `Execution Claim`（現行 `session_id`・最新 `generation`）を保持し、`active_run_id == null`（または下記の安全閉鎖により解放済み）であることを確認して新 `Run.id` を原子的に結合し `run_seq` を `+1` する。
  - [禁止] 同じ Worker であることを理由に、実行権のない別 Session による無断実行を許可する。
  - [必須] 同一計画内の通常再試行および（Job 単独中止後の）後続 Job 開始では、前の実行の停止確認**または**同一 Claim 内での試行権限の安全閉鎖（`R1` の永久閉鎖・外部フェンシングによる無効化・残存副作用なし確認のうえで `active_run_id` を解放し `run_seq +1`）を確認する（v1.0.0 規則を維持。残存副作用の疑いがあれば停止確認・復旧まで開始禁止）。
- [必須] **Run 記録と開始後照合**：Job の1回の実行試行を `Run` に記録し、結合後の実行更新・結果確定では認証済み Session・`active` Claim・対象現在 Job・**現行 `active_run_id == run.id`（非 `null` 一致）**をすべて照合する。
  - [禁止] 同じ Ticket の別 Job を同時実行する、同じ停止済み Run を見て複数 Run を並行起動する、または同じ試行の再送を二重登録する。
  - [必須] 実作業を開始した Run では `started` を必須記録する（未起動予約 `unstarted` や実作業なしで閉鎖した `unstarted → stopped` では省略可）。終了確定時に `result` を必須記録し、終了時点で当該 `Run.id` の試行権限を即座に閉じる（`active_run_id = null`, `run_seq +1`）。
  - [必須] 待機・実行制御情報（`wait`）は Job ではなく `Run` 側で保持する。
  - [禁止] 依頼送信や `Run` の終了だけで Job を完了する。
  - [禁止] 閉じた同世代の旧 `Run.id` や旧 `generation` からの業務状態更新を適用する。
  - [必須] 副作用を防ぐ必要がある操作では、効果の確定側でも失効済み権限（旧 `generation`・失効 Claim・閉じた `Run.id`）を拒否し、失効前開始済み操作が残る場合は安全確認完了まで後続実行を保留する。
- [必須] **管理・評価・証拠取り込み操作**：Claim 未発行時の前提待ち評価、権限保持運用者による保留・解除・中止（未取得 `unassigned` Ticket 中止時は `revoked`・既失効 Claim は二重失効なし）・Run を持たない Task の計画内 `waiting` 入退/完了・障害解消復帰・巻き取り復旧、および Run 終了後の `waiting` / `doing` Job への検証済み外部証拠取り込みは、それぞれの認可主体・終端保護・監査記録を満たして実行する。
- [必須] **Job の終了と status / result 分離**：確認可能な証拠に基づき完了または中止を確定する。
  - [必須] `criteria` をすべて満たせば `status` を `completed` とし、業務判定（`approved`・`changes_requested`・`passed`・`failed` 等）を `result` に分離して記録する。
  - [必須] レビュー・調査・承認・操作依頼および依頼先からの追加依頼は通常の Ticket / Job で表現する。
  - [任意] 必要な Job だけ理由を残して `canceled` にし、後続を続ける。
  - [必須] 差し戻しや失敗に応じた修正・再実行は完了済み Job を巻き戻さず新しい Ticket で対応する。
- [必須] **Ticket の終了**：全 Job が終端（`completed`・`canceled`）なら `completed` に集約し、`active` な Claim を解放・失効させる。
  - [必須] Ticket 自身がキャンセル済みなら `canceled` を維持し、`cancel_reason` を必須記録する。
  - [必須] Ticket 終了時に正式な成果・結果・状態を永続化し、次 Ticket が未作成でも終了済み Ticket をアーカイブできるようにする。
  - [禁止] 終端 Ticket（`completed`・`canceled`）を再開する、または Ticket 独自の完了条件を追加する。
- [必須] **Task の完了**：全 Ticket の終了に加え、Task の `criteria` と必須成果を確認する。
  - [禁止] Ticket の完了数や中止 Job を含む処理終了だけで Task の達成率・目標達成を判断する。
- [必須] **明示的な中止**：Task・Ticket の未終端の子孫も同時に `canceled` にし、未失効の `Execution Claim`（`unassigned`・`active`・`released`）を `revoked` として失効させる（既に失効済みの Claim は二重失効しない）。
  - [必須] 自身の中止を全 Job 終端による Ticket 完了より優先し、遅延通知による復活を禁止する。

## 停止・巻き取り・課題対応

- [必須] **状態判断**：Task・Ticket・Job における計画内の外部結果待ちは `waiting`、明示保留は `paused`、計画外の継続不能課題は `blocked` とする。
  - [必須] 待機対象の実装種類や Worker 種別で `waiting` の状態遷移を分岐させない。
  - [必須] `waiting` 解除後は現在の未完了 Job のみから一方向に続行する（複合スクリプト内の待機も同じ）。
  - [必須] 保留中（`paused`）は実行再開を禁止し、未解消の待機・課題を保持して明示解除時に再評価する（保留前状態の無条件復元禁止）。
  - [必須] 再開・障害解除・再試行に必要な条件や証拠は必ず永続正本（`depends`・`Run.wait`・`Run.result`・`Ticket.cancel_reason`・`resume`・管理履歴）に保存し、揮発性 Handoff のみに委ねない。
  - [必須] 中断・引き継ぎ前および重要な失敗時は正本と Handoff を更新し、再開時は正本・実状態と照合して、条件が変わっていない失敗を根拠なく繰り返さない。
- [必須] **複合スクリプトと副作用**：異常終了で継続不能なら `cancel_reason` を残して Ticket を `canceled` とし、修正・再実行は新 Ticket にする。
  - [必須] 障害注入等の副作用がある作業では、Ticket 中止と環境復旧を分離し、安全確認完了まで後続の危険な実行を禁止する。
- [必須] **生存判定と巻き取り**：`Run.activity`（`unstarted`・`alive`・`stopped`・`unknown`）で判定し、Heartbeat 途絶単独で `stopped` と確定しない。独立監視による失効後停止証拠の記録経路は、旧 Run の業務更新（禁止）と分離する。
  - [禁止] 稼働中（`alive`、`waiting` かつ稼働中を含む）または状態不明（`unknown`）の Run が残存する状態で作業を巻き取る・再取得する（Claim 失効済みであっても未割当とみなさない）。
  - [必須] 巻き取りは、過去 Run の全停止確認（`stopped`）・`active_run_id == null` **かつ** 副作用残存がない等の安全条件成立、さらに確認時点の `expected_generation` と `expected_run_seq` の一致を同一競合境界で検証して許可する（同時巻き取りは最大1 Session のみ成功、確認後に新 Run が介入した場合は拒否）。
  - [必須] 新 Ticket へ移す巻き取りでは、同一 `operation_id` の再送は元の期待世代が古くても冪等に既存予約/確定結果を返し、別要求は「既存予約/確定済み衝突」で拒否する。予約時点から旧 Ticket 再取得を永久禁止とし、旧 Ticket は `owner_id` 不変で `canceled` 終端化・後続 Ticket は一意作成する。予約勝者が永久に戻れない場合は認可済み復旧経路が同一予約を確定し、生存する適格 Session 取得まで後続 Claim を `unassigned` とする。
  - [必須] 同一 Worker が別 Session で安全に引き継ぐ場合は、原子的 CAS 移譲後に待機があれば待機条件・再開位置を引き継いで再評価し、待機・保留・障害がなく実行可能（`todo`/`doing`）なら新 Run を結合して続行する。生存中 Run への監視再接続、Claim 移譲、業務再実行（新 Ticket）を混同しない。
- [必須] **課題対応**：ブロックされた課題の修復、別 Worker への依頼、差し戻し修正はいずれも通常の新 Ticket にし、`refs` で元 Ticket を参照する。
  - [必須] 元 Ticket が継続可能なら、正本に記録した解除条件に基づき実際の課題解消を確認してから再開する（別 Ticket 完了だけで自動解除しない）。

## Handoff と長期知識

- [必須] **揮発性 Handoff**：`Handoff` は `Task` 所属の単一スロットとして扱い、消失横断の永続正本版 `source_revision` とスロット連番 `revision` で鮮度を検証する。Session 更新は表示対象 `current_ticket_id` の `active` Claim（結合中は現行 `Run.id` も含む）を照合し、正規の管理再生成は永続正本最新版から閲覧キャッシュのみを再構築する（実行権付与なし）。
  - [必須] Task 参照権限を持つ複数 Session による同時閲覧を許可し、閲覧権限と実行権限を厳格に分離する。
  - [禁止] Handoff を正式履歴・成果物・再開条件の正本にする、Handoff にコマンド実行機能を持たせる、または Handoff によって無関係な Ticket を直列化する。
  - [必須] Handoff が消失・破棄されても Task・Ticket・Job・中止理由・Artifact・正本 `resume`・Run・Execution Claim・管理履歴が失われず、正本から再開判断と再生成ができる構成を保つ。
- [必須] **長期知識**：Project の `knowledge` に再利用する内容（`content`）と根拠（`basis`）を残す。
  - [必須] 状態は `active`・`supersed` の2値（v1.0.0 の綴りを維持）とし、新規記録は未指定でも `active` にする。
  - [必須] 予定も現在の知識として `active` に記録し、通常は `active` だけを参照し、振り返り・再発障害確認では `supersed` も読む。
  - [必須] 使わなくなった知識は理由を残して `supersed` にし、訂正は `supersedes` 付きの新規記録とする。
- [必須] **点検**：単一不変 `owner_id`、開始前/開始後の照合分離、試行安全閉鎖、`status`/`result` 分離、`depends` と未解決 `Run.wait` の循環禁止、外部フェンシング、巻き取りの `run_seq` 検証と勝者離脱復旧、管理/評価/証拠取り込み認可、正本への再開条件保存、揮発性 Handoff の `source_revision` 鮮度検証、禁止事項1〜18、Worker 非依存性を確認する。
  - [必須] 計画・実行済みの事実・管理情報・未確認事項を分けて報告する。
