# TTJ 1.1.0 受け入れテスト仕様（UC-01〜UC-27）

本文書は、TTJ `1.1.0` 仕様（[specification.md](../skills/ttj/references/specification.md)・[json-format.md](../skills/ttj/references/json-format.md)）に対する公式受け入れテストケース **UC-01〜UC-27**（提出物 E）と、原典 §11 禁止事項 1〜18 のトレーサビリティを定義する。

## 1. テスト共通前提

- すべての検証対象 JSON 記録はルートに `protocol: "TTJ/1.1.0"` を持つ。
- テスト環境は特定の Agent・Bot・オーケストレーターに依存せず、TTJ Core のデータモデルと状態遷移・排他規則のみで検証できること。
- 特に **競合**（UC-20, UC-21, UC-26）、**停止・再起動・巻き取り**（UC-17, UC-18, UC-19, UC-25）、**循環依存**（UC-11）、**Worker 非依存性**（UC-03, UC-27）は境界条件まで厳密に検証すること。

---

## 2. 公式受け入れテストケース（UC-01〜UC-27）

### UC-01: Worker が Ticket を所有する
- **前提**：`Worker`（`id: "w-1"`, `name: "Worker One"`）および非終端の `Task`（`id: "t-1"`）が存在する。
- **操作**：`task: "t-1"`, `owner_id: "w-1"`, `jobs: ["j-1"]` を指定して `Ticket`（`id: "tk-1"`）を正式作成する。
- **期待結果**：
  - `tk-1` が作成され、`tk-1.owner_id` が `"w-1"`（存在する `Worker.id`）を参照する。
  - `w-1.name` を変更しても `tk-1.owner_id` の参照整合性が維持される。
  - 存在しない `Worker.id` を指定した場合は作成が拒否される。

### UC-02: Ticket に複数 Owner を設定
- **前提**：`Worker` `"w-1"` と `"w-2"`、および非終端 `Task` `"t-1"` が存在する。
- **操作**：
  1. `owner_id` に複数値（例：`["w-1", "w-2"]`）を指定して Ticket 作成を試みる。
  2. 旧仕様の `assignee: ["w-1", "w-2"]` を用いて複数 Owner 設定を試みる。
  3. 既存 Ticket `"tk-1"`（`owner_id: "w-1"`）に対し、`owner_id` を `"w-2"` へ変更、または配下 Job に個別 Owner `"w-2"` を設定しようと試みる。
- **期待結果**：
  - 1〜3 の操作はいずれも**拒否**され、既存 Ticket の `owner_id` は変更されない。

### UC-03: Worker の種類が未定義
- **前提**：種別属性（`owner_type`, `type`, `human`/`bot`/`ai`/`agent`/`service` 等）を一切持たない `Worker`（`id: "w-anon"`, `name: "Anon"`）を登録する。
- **操作**：`"w-anon"` を `owner_id` とする Ticket を作成し、`Session` `"s-anon-1"` で `Execution Claim` を取得し、Job の開始（`todo → doing`）・完了（`doing → completed`）および Ticket 完了を実行する。
- **期待結果**：
  - `owner_type` や種別未定義によるエラーは発生せず、作成・実行・完了がすべて正常に成功する。

### UC-04: 同じ Worker が複数 Ticket を所有
- **前提**：`Worker` `"w-1"` と非終端 `Task` `"t-1"` が存在する。
- **操作**：ともに `owner_id: "w-1"` を持つ独立した2件の Ticket（`"tk-1"`, `"tk-2"`）を作成する。
- **期待結果**：
  - 両方の Ticket 作成および所有が**許可**され、それぞれ独立して管理・実行できる。

### UC-05: 同じ Worker の複数 Session
- **前提**：`Worker` `"w-1"` が存在する。
- **操作**：ともに `worker_id: "w-1"` を持つ2つの Session（`"s-1a"`, `"s-1b"`）を作成する。
- **期待結果**：
  - 両 Session が個別の `id` で識別され、監査履歴・`Run`・`Execution Claim` 上で `"s-1a"` と `"s-1b"` を明確に区別できる。

### UC-06: 通常 Job の完了
- **前提**：Ticket `"tk-1"`（`owner_id: "w-1"`, `jobs: ["j-1", "j-2"]`）に対し、Session `"s-1"` が有効な `Execution Claim`（`generation: 1`）を保持し、`"j-1"` が `doing` である。
- **操作**：
  1. 確認可能な証拠なし（または `Run` 終了のみ）で `"j-1"` を `completed` にしようとする。
  2. `"j-1"` の `criteria` をすべて満たす確認証拠と `result` を記録し、`"j-1"` を `completed` にして `"j-2"` を開始する。
- **期待結果**：
  - 1 は拒否される（`Run` 終了のみでは完了しない）。
  - 2 で `"j-1"` が `completed` になり、次 Job `"j-2"` が開始可能（`doing` へ遷移可能）になる。完了済み `"j-1"` への巻き戻しや Job 列変更は不可。

### UC-07: レビュー依頼
- **前提**：Ticket `"tk-dev"` の現在 Job `"j-wait-review"` が「レビュー結果の受領」を `criteria` に持つ通常の Job である。
- **操作**：レビュー依頼を送信し、外部結果待ちとして `"j-wait-review"` を `waiting` に遷移させ、`Run.wait` に待機対象・解除条件・待機開始時刻を記録する。
- **期待結果**：
  - 独立した `Request` オブジェクトや専用 `WaitingJob` 型は生成されない。
  - 依頼送信だけでは `"j-wait-review"` は `completed` にならず、通常の Job として `waiting` 状態を維持し、待機制御情報は `Run` 側で保持される。

### UC-08: レビュー結果が差し戻し
- **前提**：レビュー担当 Ticket `"tk-rev"` のレビュー Job `"j-rev"` が `doing` または `waiting` である。
- **操作**：レビュー指摘メモ（確認証拠）を記録し、`"j-rev"` の `criteria`（照合と判定記録）達成として `status: "completed"`, `result: "changes_requested"` を確定する。
- **期待結果**：
  - `status: "completed"` と `result: "changes_requested"` が分離して有効に保存される。
  - 完了済み `"j-rev"` の巻き戻しや再開は行われず、修正対応は新 Ticket（`refs` で `"tk-rev"` を参照）で進める。

### UC-09: 別 Worker への作業依頼
- **前提**：`Worker` `"w-1"` が所有する Ticket `"tk-1"` の作業中、別 `Worker` `"w-2"` による独立した作業（調査・レビュー・環境作業等）が必要になった。
- **操作**：同一 Task 配下に `owner_id: "w-2"` の新 Ticket `"tk-2"` を作成し、`refs`（必要に応じて `depends`）で関連付ける。
- **期待結果**：
  - `"tk-1"` の `owner_id` は `"w-1"` のまま変更されず、`"w-2"` の作業は別 Ticket `"tk-2"` として作成・実行される。

### UC-10: 依頼先からの追加依頼
- **前提**：依頼先 `Worker` `"w-2"` が Ticket `"tk-2"` を実行中、元の依頼元 `Worker` `"w-1"` または第3の `Worker` `"w-3"` への追加依頼（補足調査・設定確認等）が発生した。
- **操作**：独立した `Request` オブジェクトを作らず、同一 Task 配下に担当 Worker を `owner_id` とする通常の新 Ticket `"tk-3"` と通常の Job を作成し、`refs` で `"tk-2"` を参照する（必要に応じて循環しない `depends` を設定する）。
- **期待結果**：
  - 追加依頼が通常の Ticket / Job として処理・完了でき、専用オブジェクトや特殊状態機械を増やさずに完結する。

### UC-11: 循環する完了依存
- **前提**：Ticket `"tk-a"` と `"tk-b"` が存在する（または `"tk-a" → "tk-b" → "tk-c"` の依存鎖が存在する）。
- **操作**：
  1. `"tk-a"` に自己依存 `depends: ["tk-a"]` を登録しようとする。
  2. `"tk-a"` が `depends: ["tk-b"]` を持つ状態で、`"tk-b"` に `depends: ["tk-a"]` を登録しようとする（および3者間循環 `"tk-c" → "tk-a"`）。
  3. 同様に、別 Ticket の結果を待つ `waiting` 構成で循環する完了依存を作ろうとする。
- **期待結果**：
  - いずれの循環完了依存も登録時点で**拒否**される。なお実行依存を持たない情報共有参照 `refs` は `depends` と区別される。

### UC-12: 複合スクリプト内の待機
- **前提**：複合スクリプトを管理する Ticket `"tk-script"`（`jobs: ["j-prep", "j-exec", "j-verify"]`）において、`"j-prep"` は `completed`、`"j-exec"` が `doing` であり、Session `"s-1"` が `Run` `"r-1"`（`generation: 1`）を実行中である。
- **操作**：
  1. スクリプト内部で外部操作待ちが発生し、`"j-exec"` を `waiting` に遷移させ、`Run.wait` に `target`・`condition`・`since`・`resume_point` を記録する。
  2. 外部操作結果を受領して `waiting` を解除し、`"j-exec"` を `doing` へ戻して続行する。
- **期待結果**：
  - 待機対象の Worker 種別によらず通常の `waiting` で扱われる。
  - `waiting` 解除後は現在の未完了 Job `"j-exec"` のみから一方向に続行し、完了済み `"j-prep"` への再入力や Job 列の追加・変更は行われない。

### UC-13: スクリプト異常終了
- **前提**：複合スクリプトを実行する Ticket `"tk-script"` の現在 Job `"j-exec"` が `doing`（`Run` `"r-1"` 稼働中）である。
- **操作**：スクリプトが異常終了し、現行 Ticket での継続が不能と判断されたため、中止理由（`cancel_reason`）およびログ成果物（`artifacts`）を記録して `"tk-script"` を `canceled` とし、`Execution Claim` を失効させる。
- **期待結果**：
  - `"tk-script"` は `status: "canceled"` となり、`cancel_reason` が永続保存される。
  - 未終端 Job は同時に `canceled`（`result: "error"` 等）となり、旧 `Execution Claim` は失効する。
  - 終端となった `"tk-script"` は再開不可となる。

### UC-14: 修正・再実行
- **前提**：Ticket `"tk-1"` が `canceled`（異常終了）または `completed`（レビュー結果 `changes_requested` / テスト結果 `failed`）となっている。
- **操作**：
  1. 元 Ticket `"tk-1"` の再開（`canceled/completed → doing`）または確定済み Job 列の編集を試みる。
  2. 同一 Task 配下に新 Ticket `"tk-2"` を作成し、`refs` に `"tk-1"`（`kind: "rerun_of"` または `"fixes"`）を記録して修正・再実行する。
- **期待結果**：
  - 1 は**拒否**される。
  - 2 は同一 Task 配下に新 Ticket `"tk-2"` が作成され（新 Task は増殖させない）、元 Ticket の成果物や中止理由を参照して正常に実行できる。

### UC-15: Handoff 同時参照
- **前提**：Task `"t-1"` に揮発性 `Handoff` スロットが存在し、Task 参照権限を持つ複数 Session（`"s-1"`, `"s-2"`, `"s-3"`、同一または別 Worker）が存在する。
- **操作**：`"s-1"`, `"s-2"`, `"s-3"` が同時に Task `"t-1"` の `Handoff` を閲覧する。
- **期待結果**：
  - すべての Session がブロックされることなく同時に `Handoff` を参照できる。

### UC-16: Handoff 閲覧者による実行
- **前提**：Session `"s-viewer"` は Task `"t-1"` の `Handoff` を閲覧できるが、Ticket `"tk-1"` の有効な `Execution Claim` を保持していない。
- **操作**：`"s-viewer"` が Handoff 閲覧状態から `"tk-1"` の Job 開始・状態変更・結果確定・コマンド実行を試みる。
- **期待結果**：
  - 実行権がないためすべての実行・更新操作が**拒否**される。Handoff 自体もコマンド実行機能を持たない。

### UC-17: 稼働中の巻き取り
- **前提**：Ticket `"tk-1"` に対し Session `"s-1"` が有効な `Execution Claim`（`generation: 1`, `state: "active"`）を保持し、現行 `Run` が `activity: "alive"`（稼働中、または `status: "waiting"` かつ `activity: "alive"`）である。
- **操作**：別 Session `"s-2"`（同一 Worker または別 Worker）が `"tk-1"` の巻き取りを要求する。
- **期待結果**：
  - 巻き取り要求は**拒否**され、`"s-1"` の `Execution Claim`（`generation: 1`）が維持される。

### UC-18: 停止確認後の巻き取り
- **前提**：Ticket `"tk-1"` の現行 `Run` が信頼できる監視により `activity: "stopped"`（停止確認済み）となっている。
- **操作**：
  1. 副作用残存の疑いがあり安全確認が未完了の状態で、巻き取りを要求する。
  2. 副作用残存がないこと（または環境復旧完了）の安全条件を確認した上で、巻き取りを要求する（別 Worker なら旧 Claim 失効＋必要に応じ旧 Ticket 中止＋新 Ticket 作成、同一 Worker 別 Session なら旧 Claim の原子的 `generation` 更新）。
- **期待結果**：
  - 1 は安全条件未成立のため**保留・拒否**される。
  - 2 は**許可**され、旧 `generation` が失効し新 `generation` で安全に実行開始できる。別 Worker 巻き取り時、旧 Ticket の `owner_id` は変更されない。

### UC-19: 状態不明の巻き取り
- **前提**：Ticket `"tk-1"` の現行 `Run` について、Heartbeat が途絶し観測有効期間が切れたため、実効 `activity` が `"unknown"` になっている（停止確認は未取得）。
- **操作**：
  1. Heartbeat 途絶のみを根拠に `activity` を `"stopped"` へ確定しようとする。
  2. `activity: "unknown"` の状態で別 Session `"s-2"` が巻き取りを要求する。
- **期待結果**：
  - 1 は禁止され、`activity` は `"unknown"` のまま維持される。
  - 2 の巻き取り要求は**拒否**される。

### UC-20: 同時巻き取り要求
- **前提**：Ticket `"tk-1"` の現行 `Run` が `activity: "stopped"` かつ安全条件成立済みで、`Execution Claim` の現行 `generation` が `1` である。
- **操作**：複数の Session（`"s-2"`, `"s-3"`, `"s-4"`）が同時に `"tk-1"` の巻き取り要求を発行する。
- **期待結果**：
  - 原子的 CAS（`generation` 比較交換）により**最大 1 Session のみ**が成功し（`generation: 2` または新 Ticket/Claim 確定）、残りの Session はすべて競合エラーで**拒否**される。

### UC-21: 古い Session の復帰
- **前提**：旧 Session `"s-1"`（`generation: 1`）の停止・安全確認後、巻き取りが完了して `Execution Claim` は `generation: 2`（または `state: "revoked"`）となっている。
- **操作**：ネットワーク復旧等により復帰した旧 Session `"s-1"` が、旧 `generation: 1` を伴って Job 完了・状態遷移・Handoff 更新・外部更新を試みる。
- **期待結果**：
  - `generation` 不一致によりすべての更新が**拒否**され、現行の作業状態・結果・Handoff は汚染されない。外部実行環境・ゲートウェイ側でも失効済み実行権の副作用を抑止する。

### UC-22: Handoff 消失
- **前提**：Task `"t-1"` 配下に複数の Ticket・Job・`cancel_reason`・`artifacts`・`Run`・`Execution Claim` が存在し、Task の揮発性 `Handoff` も存在している。
- **操作**：Task `"t-1"` の `Handoff` を破棄・削除（またはキャッシュ消失を模擬）する。
- **期待結果**：
  - Task の状態、Ticket 履歴、Job の達成結果（`status`/`result`）、Ticket の中止理由（`cancel_reason`）、Artifact、Run 履歴、Execution Claim は一切失われない。
  - 正本データから `Handoff` を再生成できる。

### UC-23: 次 Ticket 未作成で終了
- **前提**：Task `"t-1"`（未完了・継続中）配下の Ticket `"tk-1"` について、全 Job が終端（または Ticket が `canceled`）となったが、後続の Ticket はまだ作成されていない。
- **操作**：`"tk-1"` の正式な成果・結果・状態を永続化し、`"tk-1"` を終了・アーカイブする。
- **期待結果**：
  - 次 Ticket が未作成であっても `"tk-1"` の終了およびアーカイブが正常に完了する。
  - 後続 Ticket が後日作成された際に `refs` 等で成果物・結果・中止理由を参照できる。

### UC-24: 独立 Ticket の並列実行
- **前提**：同一 Task `"t-1"` 配下に、互いに `depends` 関係のない独立した2件の Ticket（`"tk-a"`・`"tk-b"`）が存在する。Task `"t-1"` の `Handoff` は単一スロットである。
- **操作**：Session `"s-a"` が `"tk-a"` の `Execution Claim` を取得し、同時に Session `"s-b"` が `"tk-b"` の `Execution Claim` を取得して、両 Ticket の Job を並列に実行する。
- **期待結果**：
  - 同一 Task に属することや `Handoff` が単一スロットであることを理由に直列化・拒否されず、`"tk-a"` と `"tk-b"` が並列に進行・完了できる。

### UC-25: `waiting` 中に Session 停止
- **前提**：Ticket `"tk-1"` の現在 Job `"j-1"` が `status: "waiting"` であり、`Run` `"r-1"` が待機制御情報 `wait` を保持している。その後、実行していた Session `"s-1"` が停止した。
- **操作**：
  1. `"s-1"` が `activity: "alive"` のまま（または `unknown` のまま）巻き取りを試みる。
  2. 監視機構により `"r-1"` の `activity: "stopped"`（停止確認済み）と安全条件成立を確認した上で、巻き取り（安全確認後の新 Session 引き継ぎ、または新 Ticket 作成）を実行する。
- **期待結果**：
  - 1 は**拒否**される。
  - 2 は**許可**され、旧 `generation` が失効した上で、待機制御情報・成果を確認して安全に後続処理（未完了 Job の継続または新 Ticket 実行）へ移行できる。

### UC-26: 同一 Worker の別 Session による無断実行
- **前提**：`Worker` `"w-1"` が Ticket `"tk-1"` の `owner_id` であり、`"w-1"` に属する Session `"s-1a"` が有効な `Execution Claim`（`generation: 1`）を保持している。同じ `"w-1"` に属する別 Session `"s-1b"` が存在する。
- **操作**：`"s-1b"` が巻き取り手続を経ずに `"tk-1"` の Job 実行・状態更新・結果確定を試みる。
- **期待結果**：
  - 同じ Worker `"w-1"` に属していても、`"s-1b"` は `Execution Claim` を保持していないため操作は**拒否**される。

### UC-27: 同じ条件で異なる Worker が担当
- **前提**：属性構成や背景が異なる2つの Worker（例：`"w-alpha"` と `"w-beta"`、いずれも種別属性を持たない）および同一内容の `objective`・`criteria`・`jobs` を持つ2つの Ticket（`"tk-a"`, `"tk-b"`）を用意する。
- **操作**：同じ入力・同じ待機イベント・同じ確認証拠を与え、両 Ticket で `todo → doing → waiting → doing → completed`（および `result` 記録）を実行する。
- **期待結果**：
  - 両 Ticket の許可遷移、ガード評価、達成判定、結果分離、終了集約の振る舞いが完全に同一となる。

---

## 3. 原典 §11 禁止事項（1〜18）トレーサビリティ

| No. | 禁止事項 | 仕様正本（specification.md） | 対応受け入れテスト |
|---|---|---|---|
| 1 | 人間、Bot、AI を異なる業務上の作業主体として扱う | §2 Worker, §10.1 #1 | UC-03, UC-27 |
| 2 | `owner_type` を必須にする | §2 Worker, §10.1 #2 | UC-03 |
| 3 | Worker の種類で Job の状態遷移を分岐する | §2 Worker, §5 `waiting`, §10.1 #3 | UC-03, UC-07, UC-12, UC-27 |
| 4 | `Request` を独立した業務オブジェクトにする | §2 Job, §3 別 Worker の作業と追加依頼, §10.1 #4 | UC-07, UC-09, UC-10 |
| 5 | Job に複雑なワークフロー構造を追加する | §2 Job, §8.1 待機・実行制御情報, §10.1 #5 | UC-06, UC-07, UC-12 |
| 6 | Job に個別 Owner を設定する | §2 Job, §3 単一 Owner, §10.1 #6 | UC-02 |
| 7 | Ticket に複数 Owner を設定する | §3 単一 Owner, §10.1 #7 | UC-02 |
| 8 | Ticket 確定後に Job 列を変更する | §4 確定後の変更, §10.1 #8 | UC-06, UC-12, UC-14 |
| 9 | 終了済み Ticket を再開する | §5.1 終端状態, §6 終端の保持, §10.1 #9 | UC-13, UC-14 |
| 10 | 循環する完了依存を許す | §7 実行依存, §8.1 待機・実行制御情報, §10.1 #10 | UC-11 |
| 11 | Handoff の閲覧権限を実行権と同一視する | §9.1 Handoff, §10.1 #11 | UC-15, UC-16 |
| 12 | Worker が同じという理由で実行権を共有させる | §3 Worker と Session の分離, §8.2, §10.1 #12 | UC-05, UC-26 |
| 13 | 稼働中の Session から作業を巻き取る | §8.4 巻き取り可否条件, §10.1 #13 | UC-17, UC-25 |
| 14 | 停止未確認の Session から作業を巻き取る | §8.3, §8.4 巻き取り可否条件, §10.1 #14 | UC-19, UC-25 |
| 15 | Heartbeat 切断のみで停止確認済みとする | §8.3 Heartbeat と停止確認の区別, §10.1 #15 | UC-19 |
| 16 | Handoff を正式履歴の正本にする | §9.1 揮発性, §10.1 #16 | UC-22 |
| 17 | Handoff によって無関係な Ticket を直列化する | §8.2 並列実行, §9.1 Handoff, §10.1 #17 | UC-24 |
| 18 | 特定の Agent・Bot・オーケストレーターへの依存を TTJ Core へ組み込む | §1 基本原則, §2 Worker, §10.1 #18 | UC-03, UC-27（および [orca-adapter.md](orca-adapter.md) の境界設計） |
