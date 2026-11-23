# TTJ 1.1.0 受け入れテスト仕様（UC-01〜UC-27・追加シナリオ UC-S01〜UC-S06）

本文書は、TTJ `1.1.0` 仕様（[specification.md](../skills/ttj/references/specification.md)・[json-format.md](../skills/ttj/references/json-format.md)）に対する公式受け入れテストケース **UC-01〜UC-27**、境界・異常系を補完する追加シナリオ **UC-S01〜UC-S06**（提出物 E）、および原典 §11 禁止事項 1〜18 の検証項目・トレーサビリティを定義する。

## 1. テスト共通前提

- すべての検証対象 JSON 記録はルートに `protocol: "TTJ/1.1.0"` を持つ。
- テスト環境は特定の Agent・Bot・オーケストレーターに依存せず、TTJ Core のデータモデル・公開契約・状態遷移・排他規則のみで検証できること。
- Ticket 正式作成時の `Execution Claim` 初期状態は `state: "unassigned"`, `session_id: null`, `active_run_id: null`, `generation: 1`, `run_seq: 0` とし、Claim 状態切替 1 回ごとに `generation` が厳密に `+1`（初回取得で `generation: 2`）、Run の開始結合・安全閉鎖・終了確定ごとに `run_seq` が `+1` される。
- 特に **競合**（UC-20, UC-21, UC-26, UC-S01, UC-S02）、**停止・再起動・巻き取り・外部フェンシング**（UC-17, UC-18, UC-19, UC-25, UC-S03）、**非実行経路・保留解除・親 Task 独立性・未起動閉鎖 Run**（UC-S04, UC-S05, UC-S06）、**循環依存と未解決/解消済み待機**（UC-11）、**Worker 非依存性**（UC-03, UC-27）を境界条件まで厳密に観測・検証すること。

---

## 2. 公式受け入れテストケース（UC-01〜UC-27）

### UC-01: Worker が Ticket を所有する
- **前提**：`Worker`（`id: "w-1"`, `name: "Worker One"`）および非終端の `Task`（`id: "t-1"`）が存在する。
- **操作**：
  1. `task: "t-1"`, `owner_id: "w-1"`, `jobs: ["j-1"]` を指定して `Ticket`（`id: "tk-1"`）を正式作成する。
  2. `w-1.name` を `"Worker One Renamed"` に変更し、`tk-1.owner_id` を読み取る。
  3. 存在しない `owner_id: "w-missing"` を指定して別 Ticket の作成を試みる。
- **期待結果**：
  - 1 で `tk-1` が作成され、`tk-1.owner_id == "w-1"` となる。
  - 2 で表示名変更後も `tk-1.owner_id == "w-1"` の参照整合が維持される。
  - 3 は参照先不在のため拒否される。

### UC-02: Ticket に複数 Owner を設定
- **前提**：`Worker` `"w-1"` と `"w-2"`、非終端 `Task` `"t-1"`、および `owner_id: "w-1"` の確定済み Ticket `"tk-1"`（`jobs: ["j-1"]`）が存在する。
- **操作**：
  1. `owner_id` に複数値（`["w-1", "w-2"]`）を指定して Ticket 作成を試みる。
  2. 旧仕様の `assignee: ["w-1", "w-2"]` を用いて Ticket 作成を試みる。
  3. `"tk-1"` の `owner_id` を `"w-2"` へ変更、または `"j-1"` に個別 Owner `"w-2"` の設定を試みる。
- **期待結果**：
  - 1〜3 の操作はいずれも**拒否**され、`"tk-1".owner_id` は `"w-1"` のまま変更されない。

### UC-03: Worker の種類が未定義
- **前提**：種別属性（`owner_type`, `type`, `human`/`bot`/`ai`/`agent`/`service` 等）を一切持たない `Worker`（`id: "w-anon"`, `name: "Anon"`）と非終端 `Task` `"t-1"` が存在する。
- **操作**：`"w-anon"` を `owner_id` とする Ticket `"tk-anon"`（`jobs: ["j-1"]`）を作成し、`Session` `"s-anon-1"`（`worker_id: "w-anon"`）で Claim を取得（`generation: 2`）して `Run` `"r-1"` を結合開始し、確認証拠を記録して `"j-1"` および `"tk-anon"` を `completed` にする。
- **期待結果**：
  - `owner_type` や種別未定義による拒否は発生せず、作成・Claim 取得・実行・証拠付き完了・Ticket 集約がすべて正常に成功する。

### UC-04: 同じ Worker が複数 Ticket を所有
- **前提**：`Worker` `"w-1"` と非終端 `Task` `"t-1"` が存在する。
- **操作**：ともに `owner_id: "w-1"` を持つ独立した 2 件の Ticket（`"tk-1"`, `"tk-2"`）を作成する。
- **期待結果**：
  - 両方の Ticket 作成および所有が**許可**され、それぞれ独立した `ticket_id` キーの Claim で管理・実行できる。

### UC-05: 同じ Worker の複数 Session
- **前提**：`Worker` `"w-1"` が所有する 2 件の独立 Ticket（`"tk-1"`, `"tk-2"`）が存在する。
- **操作**：
  1. ともに `worker_id: "w-1"` を持つ 2 つの Session（`"s-1a"`, `"s-1b"`）を作成する。
  2. `"s-1a"` で `"tk-1"` の `Execution Claim` を取得し、`Run` `"r-1a"` を結合開始して監査イベントを記録する。
  3. `"s-1b"` で `"tk-2"` の `Execution Claim` を取得し、`Run` `"r-1b"` を結合開始して監査イベントを記録する。
- **期待結果**：
  - `"tk-1"` の Claim・`Run` `"r-1a"`・監査レコードには `session_id: "s-1a"` が、`"tk-2"` の Claim・`Run` `"r-1b"`・監査レコードには `session_id: "s-1b"` が記録され、同一 Worker 配下の両 Session を実行権・実行履歴・監査ログ上で個別に識別できる。

### UC-06: 通常 Job の完了
- **前提**：2 つの独立フィクスチャを用意する。
  - **Fixture A（負例：Run 終了・証拠不足）**：Ticket `"tk-neg"`（`owner_id: "w-1"`, `jobs: ["j-neg-1", "j-neg-2"]`）で Session `"s-1"` が `active` Claim（`generation: 2`）を保持し、`"j-neg-1"` が `doing`（現行 `Run` `"r-neg-1"`）である。
  - **Fixture B（正例：現行 Run での証拠付き完了と次 Job 進行）**：Ticket `"tk-pos"`（`owner_id: "w-1"`, `jobs: ["j-pos-1", "j-pos-2"]`）で Session `"s-1"` が `active` Claim（`generation: 2`）を保持し、`"j-pos-1"` が `doing`（現行 `Run` `"r-pos-1"`, `active_run_id == "r-pos-1"`）である。
- **操作**：
  1. **Fixture A**：`"r-neg-1"` の終了（`ended`・`result: "succeeded"`）のみを記録し、確認可能な証拠なしで `"j-neg-1"` を `completed` にしようとする。
  2. **Fixture B**：現行 `Run` `"r-pos-1"` の結合中に `"j-pos-1"` の `criteria` をすべて満たす確認証拠と `result: "passed"` を記録して `"j-pos-1"` を `completed` に確定し、`"r-pos-1"` の終了・停止確認（または安全条件確認）により `active_run_id: null` とした上で次 Job `"j-pos-2"` の新 Run `"r-pos-2"` を結合開始する。
  3. **Fixture B**：完了済み `"j-pos-1"` への巻き戻し（`completed → doing`）および `"tk-pos".jobs` の変更を試みる。
- **期待結果**：
  - 1（Fixture A）は証拠不足のため**拒否**され、`"j-neg-1"` は `completed` にならない。
  - 2（Fixture B）で `"j-pos-1"` が `completed` になり、次 Job `"j-pos-2"` が `doing`（`active_run_id == "r-pos-2"`）で正常に開始される。
  - 3 はいずれも**拒否**される。

### UC-07: レビュー依頼
- **前提**：Ticket `"tk-dev"` の現在 Job `"j-wait-review"` が「レビュー結果の受領」を `criteria` に持つ通常の Job であり、Session `"s-1"` が現行 `Run` `"r-1"` を実行中である。
- **操作**：レビュー依頼を送信し、外部結果待ちとして `"j-wait-review"` を `waiting` に遷移させ、`"r-1".wait` に `job_id`・`target`・`condition`・`since` を記録する。
- **期待結果**：
  - 独立した `Request` オブジェクトや専用 `WaitingJob` 型は生成されない。
  - 依頼送信だけでは `"j-wait-review"` は `completed` にならず、通常の Job として `waiting` を維持し、待機制御情報は `Run` 側（`"r-1".wait`）で保持される。

### UC-08: レビュー結果が差し戻し
- **前提**：レビュー担当 Ticket `"tk-rev"` のレビュー Job `"j-rev"` が `doing` または `waiting` である。
- **操作**：レビュー指摘メモ（確認証拠）を `artifacts` に記録し、`"j-rev"` の `criteria`（照合と判定記録）達成として `status: "completed"`, `result: "changes_requested"` を確定する。
- **期待結果**：
  - `status: "completed"` と `result: "changes_requested"` が分離して有効に保存される。
  - 完了済み `"j-rev"` の巻き戻し・再開は不可であり、修正対応は新 Ticket（`refs` で `"tk-rev"` を参照）で進める。

### UC-09: 別 Worker への作業依頼
- **前提**：`Worker` `"w-1"` が所有する Ticket `"tk-1"` の作業中、別 `Worker` `"w-2"` による独立した作業が必要になった。
- **操作**：同一 Task 配下に `owner_id: "w-2"` の新 Ticket `"tk-2"` を作成し、`refs`（必要に応じて循環しない `depends`）で関連付ける。
- **期待結果**：
  - `"tk-1"` の `owner_id` は `"w-1"` のまま変更されず、`"w-2"` の作業は別 Ticket `"tk-2"` として独立に作成・実行できる。

### UC-10: 依頼先からの追加依頼
- **前提**：依頼先 `Worker` `"w-2"` が Ticket `"tk-2"` を実行中、元の依頼元 `Worker` `"w-1"`（または別 `Worker` `"w-3"`）への追加依頼（補足調査・設定確認等）が発生した。
- **操作**：
  1. 独立した `Request` オブジェクトを作らず、同一 Task 配下に `owner_id: "w-1"` の通常の新 Ticket `"tk-3"`（`jobs: ["j-extra-check"]`, `refs: [{"ticket_id": "tk-2", "kind": "derived_from"}]`）を作成する。
  2. `"w-1"` の Session `"s-1"` が `"tk-3"` の `Execution Claim` を取得して `Run` `"r-3"` を開始し、`"j-extra-check"` の `criteria` を満たす確認証拠（`artifacts`）と `result: "passed"` を記録して `"j-extra-check"` および `"tk-3"` を `completed` にする。
- **期待結果**：
  - 追加依頼が独立 `Request` や特殊状態機械なしで通常の Ticket / Job として起票・実行・証拠付き完了でき、`"tk-2"` 側から `"tk-3"` の結果・成果物を参照できる。

### UC-11: 循環する完了依存
- **前提**：非終端 Ticket `"tk-a"`, `"tk-b"`, `"tk-x"`, `"tk-y"`, `"tk-z"`, `"tk-p"`, `"tk-q"`, `"tk-c1"`, `"tk-c2"` が存在する。
- **操作**：
  1. **自己依存・静的循環**：`"tk-a"` に `depends: ["tk-a"]` を登録試行（拒否）。次に `"tk-a"` に `depends: ["tk-b"]` を登録（成功）した上で `"tk-b"` に `depends: ["tk-a"]` を登録試行。
  2. **未解決 `Run.wait` 混在循環（停止・移譲中含む）**：
     - (2a) `"tk-x"` に `depends: ["tk-y"]` を登録（成功）し、`"tk-y"` の現在 Job の未解決 `Run.wait.target` に `"tk-z"` の完了待ちを登録（成功、停止・移譲中で未解決のまま引き継がれている状態を含む）した上で、`"tk-z"` に `depends: ["tk-x"]` を登録試行。
     - (2b) `"tk-x"` が `depends: ["tk-y"]` を持つ状態で、`"tk-y"` の現在 Job の未解決 `Run.wait.target` に直接 `"tk-x"` の完了待ちを登録試行。
  3. **解消済み履歴待機を辺から外した逆向き登録（対照・許可）**：`"tk-p"` の `Run` `"r-p1"` が `"tk-q"` の完了待ちを `Run.wait` として記録した後、待機が解消・解除（`Run.wait.resolved` 記録、`"tk-p"` はもはや `"tk-q"` を待っていない）された状態で、`"tk-q"` から `"tk-p"` を待つ完了依存（`depends: ["tk-p"]` または `Run.wait`）を登録する。
  4. **逆向き競合登録の確定境界**：互いに依存のない `"tk-c1"` と `"tk-c2"` に対し、`"tk-c1" → "tk-c2"`（`depends` または `Run.wait`）と `"tk-c2" → "tk-c1"`（`depends` または `Run.wait`）をバリア同期で並行登録する。
  5. **情報共有参照の対照**：`"tk-b"` の `refs` に `{"ticket_id": "tk-a", "kind": "related"}` を登録する。
- **期待結果**：
  - 1・(2a)・(2b) の循環完了依存はいずれも登録時点で**拒否**される。
  - 3 は解消済み `Run.wait` が履歴に残ったまま完了依存グラフの辺から外れるため、逆向き登録が**許可**される。
  - 4 は原子的な循環検証・確定境界により**高々一方のみ成功**し、もう一方は循環依存として**拒否**される（双方同時成功しない、拒否側は正本無変更）。
  - 5 の情報共有参照 `refs` は実行依存ではないため**許可**される。

### UC-12: 複合スクリプト内の待機
- **前提**：複合スクリプトを管理する Ticket `"tk-script"`（`jobs: ["j-prep", "j-exec", "j-verify"]`）において、`"j-prep"` は `completed`、`"j-exec"` が `doing` であり、Session `"s-1"` が現行 `Run` `"r-1"`（`generation: 2`, `active_run_id == "r-1"`）を実行中である。
- **操作**：
  1. スクリプト内部で外部操作待ちが発生し、`"j-exec"` を `waiting` に遷移させ、`"r-1".wait` に `target`・`condition`・`since`・`resume_point: "step-3"` を記録する。
  2. 外部操作結果を受領して `waiting` を解除し（`"r-1".wait.resolved` 記録）、`"j-exec"` を `doing` へ戻して `"step-3"` 以降を続行・完了する。
- **期待結果**：
  - 待機対象の Worker 種別によらず通常の `waiting` で扱われる。
  - `waiting` 解除後は現在の未完了 Job `"j-exec"` のみから一方向に続行し、完了済み `"j-prep"` への再入力や Job 列の追加・並べ替えは行われない。

### UC-13: スクリプト異常終了
- **前提**：複合スクリプトを実行する Ticket `"tk-script"` の現在 Job `"j-exec"` が `doing`（現行 `Run` `"r-1"`、`generation: 2`）である。
- **操作**：スクリプトが異常終了し現行 Ticket で継続不能と判断されたため、`cancel_reason` とログ `artifacts` を記録して `"tk-script"` を `canceled` とし、Claim を `revoked`（`generation: 3`）へ失効させる。
- **期待結果**：
  - `"tk-script"` は `status: "canceled"` となり、`cancel_reason` が永続保存される（`cancel_reason` 省略の中止は拒否）。
  - 未終端 Job `"j-exec"` は同時に `canceled`（`result: "error"` 等）となり、旧 Claim は `revoked`・`session_id: null`・`active_run_id: null` となる。
  - 終端となった `"tk-script"` の再開は拒否される。

### UC-14: 修正・再実行
- **前提**：Ticket `"tk-1"` が `canceled`（異常終了）または `completed`（レビュー結果 `changes_requested` / テスト結果 `failed`）となっている。
- **操作**：
  1. 元 Ticket `"tk-1"` の再開（`canceled/completed → doing`）または確定済み Job 列の編集を試みる。
  2. 同一 Task 配下に新 Ticket `"tk-2"` を作成し、`refs` に `"tk-1"`（`kind: "rerun_of"` または `"fixes"`）を記録して修正・再実行する。
- **期待結果**：
  - 1 は**拒否**される。
  - 2 は同一 Task 配下に新 Ticket `"tk-2"` が作成され（新 Task は増殖させない）、元 Ticket の成果物や中止理由を参照して正常に実行できる。

### UC-15: Handoff 同時参照
- **前提**：Task `"t-1"` に揮発性 `Handoff` スロットが存在し、Task 参照権限を持つ複数 Session（`"s-1"`, `"s-2"`, `"s-3"`）が存在する。
- **操作**：`"s-1"`, `"s-2"`, `"s-3"` が同時に Task `"t-1"` の `Handoff` を閲覧する。
- **期待結果**：
  - すべての Session がブロックされることなく同時に `Handoff` を参照できる。

### UC-16: Handoff 閲覧者による実行
- **前提**：Session `"s-viewer"` は Task `"t-1"` の `Handoff` を閲覧できるが、Ticket `"tk-1"` の `active` な `Execution Claim` を保持していない。
- **操作**：`"s-viewer"` が Handoff 閲覧状態から `"tk-1"` の Job 開始・状態変更・結果確定・Handoff 経由のコマンド実行を試みる。
- **期待結果**：
  - 実行権がないためすべての実行・更新操作が**拒否**される。Handoff 自体もコマンド実行機能を持たない。

### UC-17: 稼働中の巻き取り
- **前提**：Ticket `"tk-1"` に対し Session `"s-1"` が `active` な `Execution Claim`（`generation: 2`）を保持し、現行 `Run` `"r-1"` が `activity: "alive"`（`doing` 中、または `status: "waiting"` かつ `activity: "alive"`）である。また、中止等で Claim が `revoked` 済みだが旧 `Run` が依然 `activity: "alive"` で残存するフィクスチャも用意する。
- **操作**：別 Session `"s-2"` が `"tk-1"` の巻き取り（または新規取得）を要求する。
- **期待結果**：
  - いずれのケースも巻き取り・取得は**拒否**され、既存 Claim・Ticket の状態は変更されない。

### UC-18: 停止確認後の巻き取り
- **前提**：2 つの独立フィクスチャを用意する。
  - **Fixture A（同一 Worker 移譲）**：`Worker` `"w-1"` 所有の Ticket `"tk-a"` に対し、Session `"s-1a"` が Claim（`generation: 2`, `run_seq: 2`, `state: "active"`, `active_run_id: null`）を保持し、全過去 Run（`"r-1a"`）が独立監視により `activity: "stopped"`（終了確定済み）になっている。同一 Worker `"w-1"` の別 Session `"s-2a"` が存在する。
  - **Fixture B（別 Worker への新 Ticket 移管）**：`Worker` `"w-1"` 所有の未終端 Ticket `"tk-b"` に対し、Session `"s-1b"` の全過去 Run（`"r-1b"`）が `activity: "stopped"`（終了確定済み、`generation: 2`, `run_seq: 2`, `active_run_id: null`）になっている。別 Worker `"w-2"` の Session `"s-2b"` が存在する。
- **操作**：
  1. Fixture A・B の双方で、副作用残存の疑いがあり安全条件未成立の状態で巻き取りを要求する。
  2. 安全条件成立を確認した上で：
     - Fixture A：`"s-2a"` が `(expected_generation: 2, expected_run_seq: 2)` で Claim 移譲を実行し、続いて旧 `"s-1a"` が旧 `generation: 2` で更新を試みる（さらに待機・障害のない `doing` Job であれば移譲後に `"s-2a"` が新 Run を結合開始する）。
     - Fixture B：`"s-2b"` が `(expected_generation: 2, expected_run_seq: 2, operation_id: "op-b-1")` で新 Ticket への巻き取りを実行し、続いて旧 `"s-1b"` が旧 `generation: 2` で `"tk-b"` の更新を試みる。
- **期待結果**：
  - 1（失敗時）：Fixture A・B ともに巻き取りは**保留・拒否**され、双方の Claim（`generation: 2`, `run_seq: 2`, `session_id`, `state`）および Ticket は一切変化しない。
  - 2（Fixture A 成功時）：`"tk-a"` の Claim が `generation: 3`, `session_id: "s-2a"`, `active_run_id: null`, `state: "active"` へ更新され、待機・障害のない `doing` Job なら新 Run を開始でき、旧 `"s-1a"`（`generation: 2`）の更新は**拒否**される。
  - 2（Fixture B 成功時）：`"tk-b"` は `owner_id: "w-1"` のまま `status: "canceled"`（`cancel_reason` 保存）・Claim `revoked`（`generation: 3`, 再取得永久禁止、二重失効なし）となり、`owner_id: "w-2"` の新 Ticket `"tk-b2"`（`refs` に `"tk-b"`）と新 Claim（`unassigned(gen=1) → active(gen=2)`）が作成され、旧 `"s-1b"`（`generation: 2`）の更新は**拒否**される。

### UC-19: 状態不明の巻き取り
- **前提**：Ticket `"tk-1"` の現行 `Run` `"r-1"` について、Heartbeat が途絶し観測有効期間を超過したため実効 `activity` が `"unknown"` になっている（停止確認は未取得）。また Claim 失効済みだが `"r-1"` が `activity: "unknown"` のフィクスチャも用意する。
- **操作**：
  1. Heartbeat 途絶のみを根拠に `"r-1".activity` を `"stopped"` へ確定しようとする。
  2. `activity: "unknown"` の状態で別 Session `"s-2"` が巻き取り（または新規取得）を要求する。
- **期待結果**：
  - 1 は禁止され、`"r-1".activity` は `"unknown"` のまま維持される。
  - 2 の巻き取り・取得要求はいずれも**拒否**される。

### UC-20: 同時巻き取り要求
- **前提**：Ticket `"tk-1"` の全過去 Run が `activity: "stopped"`・`active_run_id: null` かつ安全条件成立済みで、`Execution Claim` は `generation: 2, run_seq: 2` である。適格な 3 つの Session（`"s-2"`, `"s-3"`, `"s-4"`）を用意する（同一 Worker 移譲および別 Worker 新 Ticket 移管の双方を検証）。
- **操作**：`"s-2"`, `"s-3"`, `"s-4"` がすべて同一の `(expected_generation: 2, expected_run_seq: 2)` を指定し、バリア同期で同時に `"tk-1"` の巻き取り要求を発行する。
- **期待結果**：
  - 原子的 CAS により**厳密に 1 Session のみ**が成功する。
  - 同一 Worker 移譲では旧 Ticket Claim のみが `generation: 3` へ更新され、別 Worker 新 Ticket 移管では**旧 Ticket `"tk-1"` の Claim が `state: "revoked", generation: 3`** となり、**後続 Ticket は厳密に 1 つだけ作成されその後続 Ticket の Claim が初回取得の `state: "active", generation: 2`**（`unassigned(gen=1) → active(gen=2)`）となる。
  - 敗者 2 Session は競合エラーで**拒否**され、敗者による Claim・Ticket・Job・監査状態の部分変更は一切残らない。

### UC-21: 古い Session の復帰
- **前提**：外部リソース更新を伴う実行ゲートウェイ（モック外部ストア `ext_value = "v0"`）を接続し、2 ケースを用意する。
  - **Case A（世代不一致）**：同一 Worker 移譲により Claim が `generation: 2 → 3`（新 Session `"s-2"`）へ進んだ状態。
  - **Case B（失効）**：中止または別 Worker 巻き取りにより Claim が `state: "revoked"`, `generation: 3` となった状態。
- **操作**：ネットワーク復旧等で復帰した旧 Session `"s-1"` が、旧 `generation: 2` を提示して (1) TTJ Core への Job 完了・状態遷移・Handoff 更新、(2) 外部ゲートウェイ経由の `ext_value = "stale-write"` 更新、をそれぞれ実行する。
- **期待結果**：
  - Case A・Case B の双方で、TTJ Core 側の状態・結果・Handoff 更新はすべて**拒否**される。
  - さらに外部ゲートウェイ（効果確定側）でも旧 `generation: 2` / 失効済み Claim が拒否され、`ext_value` が `"v0"` のまま実際に変更されないことを観測できる。

### UC-22: Handoff 消失
- **前提**：Task `"t-1"` 配下に、進行中（永続正本 `V1`: `source_revision = 10`）・`blocked`（正本 `resume` に解除条件を記録）・`waiting`（`Run.wait` に待機条件を記録）・`canceled`（`cancel_reason` を記録）の各 Ticket・Job・`artifacts`・`Run`・`Execution Claim`（`generation: 2`）・管理履歴が存在し、Task の揮発性 `Handoff` も存在している。旧処理 `P_stale` が `V1`（`source_revision = 10`）のスナップショットを保持したまま一時遅延している。
- **操作**：
  1. 同一 Ticket・同一 Claim 世代（`generation: 2`）のまま正本状態が `V1`（`source_revision = 10`）から `V2`（`source_revision = 12`）へ進んだ後、Task `"t-1"` の `Handoff` を完全破棄・削除する。
  2. 実行プロセスおよび Session を**再起動**し、認可された管理再生成処理が永続正本最新版（`V2`, `source_revision = 12`）から `Handoff` を再生成する（`revision = 1, source_revision = 12`）。
  3. 遅延していた旧処理 `P_stale` が、消失前の旧スナップショット `V1`（`source_revision = 10`, 同一 `generation: 2`）に対し、再生成後より大きい `revision = 99` を付けて `Handoff` 更新を試みる。
  4. 再起動後の Session が正本レコードのみを読み取り、`blocked` の解除条件・`waiting` の再開位置・`canceled` の中止理由を照合する。
- **期待結果**：
  - 1〜2 で Task・Ticket・Job・`cancel_reason`・`artifacts`・正本 `resume`・`Run`（`wait` 含む）・`Execution Claim`・管理履歴は一切失われず、管理再生成はいかなる Session にも実行権を付与せず `V2` の表示を復元する。
  - 3 で `P_stale` がより大きい `revision = 99` と一致する `generation: 2` を提示しても、永続正本版 `source_revision` の比較（`10 < 12`）により遅延適用が**確実に拒否**され、現行 `Handoff`（`V2`）は上書きされない。
  - 4 で再起動後も揮発性 Handoff なしで再開判断・障害解除照合を完了できる。

### UC-23: 次 Ticket 未作成で終了
- **前提**：継続中の Task `"t-1"` 配下に Ticket `"tk-1"`（`jobs: ["j-1", "j-2"]`, `owner_id: "w-1"`）があり、`"j-1"` は `completed`、最後の Job `"j-2"` が `doing`（現行 `Run` `"r-2"`）である。後続 Ticket は一切作成されていない。
- **操作**：
  1. 後続 Ticket を作成しないまま、`"j-2"` の確認証拠と `result: "passed"` を記録して `"j-2"` を `completed` に確定する。
  2. 同時集約後の `"tk-1"` を正式成果・状態・履歴とともに永続化し、アーカイブ（終了保管）する。
  3. アーカイブ後に `"tk-1"` の正式記録（`status`, `jobs`, `artifacts`, `Run`, 監査履歴）を参照する。
- **期待結果**：
  - 1 で `"j-2"` の終端確定と同時に `"tk-1"` が `completed` へ自動集約され、`"tk-1"` の Claim が解放（`released`）される。
  - 2 で次 Ticket 未作成でもアーカイブが正常に完了する。
  - 3 でアーカイブ後も正式記録を参照でき、後日作成する新 Ticket の `refs` から参照できる。

### UC-24: 独立 Ticket の並列実行
- **前提**：同一 Task `"t-1"` 配下に、互いに `depends` 関係のない独立した 2 件の Ticket（`"tk-a"`, `"tk-b"`）が存在し、Task `"t-1"` の `Handoff` は単一スロットである。
- **操作**：Session `"s-a"` が `"tk-a"` の Claim（キー `"tk-a"`）を取得し、同時に Session `"s-b"` が `"tk-b"` の Claim（キー `"tk-b"`）を取得して、両 Ticket の Job を並列に実行・完了する。
- **期待結果**：
  - 同一 Task に属することや `Handoff` が単一スロットであることを理由に直列化・拒否されず、`"tk-a"` と `"tk-b"` が並列に進行・完了できる。

### UC-25: `waiting` 中に Session 停止
- **前提**：Ticket `"tk-1"`（`owner_id: "w-1"`）の現在 Job `"j-1"` が `status: "waiting"` であり、旧 Session `"s-1"` が実行していた `Run` `"r-1"`（`generation: 2`）が待機制御情報 `wait`（`target`, `condition`, `since`, `resume_point: "step-4"`）を保持している。同一 Worker `"w-1"` の別 Session `"s-2"`（巻き取り要求者）が存在する。
- **操作**：
  1. `"r-1"` が `activity: "alive"` または `"unknown"`（または終了未確定で `active_run_id == "r-1"`）の状態で、`"s-2"` が巻き取りを要求する。
  2. 独立監視により `"r-1".activity` が `"stopped"`（停止確認済み）となり、かつ `"r-1"` の終了確定（`ended`・`result: "interrupted"`・`claim.active_run_id: null`・`run_seq: 2`）と安全条件成立が揃った状態で、`"s-2"` が巻き取り（Claim 移譲：`expected_generation: 2, expected_run_seq: 2 → generation: 3`）を実行し、`"r-1".wait` を引き継いで状態を再評価する。
  3. 待機条件が未成立の段階で `"j-1"` の次工程開始を試みる。その後、外部応答が到着し、認可済み証拠取り込み経路（または現行 `active` Claim を保持する `"s-2"`）で待機解除（`waiting → doing`、`wait.resolved` 記録）を確定した上で、`"s-2"` が新 `Run` `"r-2"`（`generation: 3`）を結合開始して `"step-4"` 以降を完了する。
- **期待結果**：
  - 1 は**拒否**される。
  - 2 で `"s-2"` が `generation: 3` の Claim を取得し、待機条件・再開位置 `"step-4"` を引き継ぐ。
  - 3 で待機未成立の間は `waiting` を維持し（未解除待機中の実作業開始は拒否）、認可済み経路による待機解除後にのみ `"j-1"` を一方向に続行・完了できる。

### UC-26: 同一 Worker の別 Session による無断実行
- **前提**：`Worker` `"w-1"` が Ticket `"tk-1"` の `owner_id` であり、`"w-1"` に属する Session `"s-1a"` が `active` な Claim（`generation: 2`）を保持している。同じ `"w-1"` に属する別 Session `"s-1b"` が存在する。
- **操作**：`"s-1b"` が巻き取り手続を経ずに `"tk-1"` の Job 開始・状態更新・結果確定・Handoff 更新を試みる。
- **期待結果**：
  - 同じ Worker `"w-1"` に属していても、`"s-1b"` は Claim を保持していないためすべての操作が**拒否**される。

### UC-27: 同じ条件で異なる Worker が担当
- **前提**：異なる外部主体に対応する 2 つの Worker（例：人間オペレーター相当の外部主体に紐づく `"w-alpha"` と、自動化ボット相当の外部主体に紐づく `"w-beta"`。いずれも TTJ 上は `owner_type` 等の種別属性を持たない）および同一内容の `objective`・`criteria`・`jobs` を持つ 2 つの Ticket（`"tk-a"`, `"tk-b"`）を用意する。
- **操作**：
  1. **正常系比較**：両 Ticket に同一の入力・同一の待機イベント・同一の確認証拠を与え、`todo → doing → waiting → doing → completed`（および `result` 記録）を実行する。
  2. **拒否ガード比較**：両 Ticket に対し、同一の不正操作（証拠なし完了、無権限 Session 実行、稼働中巻き取り、循環 `depends` 登録）をそれぞれ実行する。
- **期待結果**：
  - 1 の正常系（許可遷移・達成判定・結果分離・終了集約）および 2 の拒否ガード（拒否結果・状態無変化）の双方が、`"w-alpha"` と `"w-beta"` で完全に同一となる。

---

## 3. 追加シナリオ（UC-S01〜UC-S06）

### UC-S01: 同じ世代の旧 Run 拒否・並行再試行排他・同一 Claim 安全閉鎖
- **前提**：2 つの独立フィクスチャを用意する。
  - **Fixture A（並行再試行排他と同世代旧 Run 拒否）**：Ticket `"tk-1"`（`owner_id: "w-1"`, `jobs: ["j-1"]`）で Session `"s-1"` が `active` Claim（`generation: 2`, `run_seq: 2`, `active_run_id: null`）を保持し、最初の試行 `Run` `"r-1"` が停止確認済み（`activity: "stopped"`, `ended`・`result: "interrupted"`）である。
  - **Fixture B（`unknown` 旧試行の安全閉鎖による同 Claim 再試行および Job 単独中止後続開始）**：Ticket `"tk-2"`（`owner_id: "w-1"`, `jobs: ["j-2a", "j-2b"]`）で Session `"s-1"` が `active` Claim（`generation: 2`, `run_seq: 1`, `active_run_id: "r-unk-1"`）を保持し、`"r-unk-1"` は `activity: "unknown"` だが外部フェンシングで確実に無効化でき残存副作用の疑いがない。
- **操作**：
  1. **Fixture A（並行再試行と遅延結果）**：同じ停止済み `"r-1"` を前提に、`"s-1"` から 2 件の再試行開始要求（`"r-2a"`, `"r-2b"`）をバリア同期で並行発行し、勝者 Run（`"r-2a"`）を敗者（`"r-2b"`）の照合完了まで実行中（`active_run_id == "r-2a"`）に保つ。続いて、同じ `session_id: "s-1"`, `generation: 2` を持つ旧 Run `"r-1"` の遅延結果（`result: "succeeded"`・Job 完了要求）を送信し、対照として独立監視機構が失効後 Run の専用観測経路で `activity: "stopped"` のみを記録する。
  2. **Fixture B（安全閉鎖による同 Claim 再試行・後続開始）**：`"s-1"` が Claim を失効させずに `"r-unk-1"` の試行権限を安全閉鎖（永久閉鎖・終了 `result: "interrupted"`・外部フェンシング確認・`active_run_id: null`・`run_seq: 2`）し、同じ `active` Claim（`generation: 2`）へ新 Run `"r-unk-2"` を結合開始する（`run_seq: 3`）。同様に `"j-2a"` を単独中止（`canceled`）した上で安全閉鎖を満たし、後続 Job `"j-2b"` を同 Claim で開始する。対照として、別 Session `"s-2"` による巻き取りは `"r-unk-1"` が `unknown` の間は拒否されることも確認する。
- **期待結果**：
  - 1 で開始前照合（`active_run_id == null`）の原子的結合により厳密に 1 件（`"r-2a"`, `run_seq: 3`）のみ起動し、`"r-2b"` は競合拒否される。同一 Session・同一 `generation: 2` の旧 `"r-1"` 遅延結果は開始後照合（`active_run_id != "r-1"`）により**拒否**され、独立監視経路のみが観測情報を記録して業務状態を変更しない。
  - 2 で `R1` が `unknown` のままでも安全条件成立により Claim を失効させず同 Claim（`generation: 2`）で `"r-unk-2"` および単独中止後続 `"j-2b"` を開始でき、一方別 Session への巻き取りは全過去 Run 停止未確認のため**拒否**される。

### UC-S02: 巻き取りの途中停止復旧・再送冪等性・既存予約衝突・切替時陳腐化拒否
- **前提**：未終端の停止済み Ticket `"tk-old"`（`owner_id: "w-1"`, 全過去 Run `stopped`, `active_run_id: null`, 安全条件成立, Claim `generation: 2, run_seq: 2`）を用い、以下 4 パターン（独立フィクスチャ）を用意する。
- **操作**：
  1. **Pattern (a) 勝者 `S2` が永久に戻れない場合の認可済み復旧**：別 Worker `"w-2"` の Session `"s-2"`（`operation_id: "op-tk-a"`）が CAS（`generation: 2 → 3`, `state: "revoked"`, `takeover: { status: "reserved", successor_ticket_id: "tk-new-a" }`）成功直後に Session 終了（`Session.ended` 確定・永久離脱）する。予約中に旧 `"tk-old"` の再取得を試み、続いて認可された復旧主体（監査記録付き）が同一予約を確定し、その後 `"w-2"` の生存 Session `"s-2-live"` が後続 `"tk-new-a"` の Claim を取得する。
  2. **Pattern (b) 勝者 `S2` による同一要求の再送（古い期待世代）**：`"s-2"`（生存復帰）が元の古い `expected_generation: 2` のまま同一 `operation_id: "op-tk-b"` で巻き取り要求を再送し、確定後にもう一度同一 `operation_id: "op-tk-b"`（`expected_generation: 2`）を再送する。
  3. **Pattern (c) 別 Session `S3` が現行世代 3 で送る別要求の衝突拒否**：`"s-2"` の予約（`generation: 3`, `takeover.status: "reserved"` または `"committed"`）が存在する状態で、別 Worker `"w-3"` の Session `"s-3"` が**現行世代 `expected_generation: 3`** と別 `operation_id: "op-tk-c"` を提示して巻き取りを要求する。
  4. **Pattern (d) 切替直前の旧 `S1` による同世代 `R2` 介入（陳腐化安全確認の拒否）**：`"s-2"` が `R1` 停止・安全確認を `(expected_generation: 2, expected_run_seq: 2)` で完了した直後、旧 `"s-1"` が同世代 `generation: 2` で新 Run `"r-2"` を開始（`run_seq: 3`）する。(d1) `"r-2"` が稼働中（`active_run_id: "r-2"`）のまま `"s-2"` の巻き取りが到着する場合、(d2) `"r-2"` が完了終了して `active_run_id` が再び `null`（`run_seq: 4`）に戻った後に `"s-2"` の古い確認 `(expected_generation: 2, expected_run_seq: 2)` に基づく巻き取りが到着する場合、の双方を実行する。
- **期待結果**：
  - (a) 予約中から旧 `"tk-old"` の再取得は永久禁止により**拒否**される。認可済み復旧により `"tk-old"` は `canceled`（旧 Claim は二重失効・追加世代加算なし）へ終端化し、`"tk-new-a"` が厳密に 1 件確定する。終了済み `"s-2"` には新 Claim が**結合されず**、生存 `"s-2-live"` が取得するまで `"tk-new-a"` の Claim は `unassigned (generation: 1)` を維持し、取得時に `active (generation: 2)` となる。
  - (b) 最優先の同一 `operation_id` 判定により、元の `expected_generation: 2` が現行世代 3 より古くても世代不一致で拒否されず、予約確定および 2 度目の再送で同一の後続 Ticket を冪等に返却する。
  - (c) `"s-3"` は現行世代 3 を提示しているが、世代不一致ではなく**既存予約／確定済み衝突**を根拠に**拒否**され、正本は一切変更されない。
  - (d) (d1)・(d2) のいずれも `run_seq` 不一致（`2 != 3` / `2 != 4`）により切替時に**拒否**され、稼働中介入および古い安全確認の流用を確実に防ぐ。

### UC-S03: 失効直前に認可された外部処理・同世代閉鎖 Run の確定側拒否と安全確認保留
- **前提**：外部副作用を伴う Ticket `"tk-fx"` に対し、Session `"s-1"` が `generation: 2` の Claim を保持し、3 ケースを用意する：
  - (1) Claim 失効直前に認可された外部操作 `op-ext-1`（`generation: 2`）
  - (2) 同一 `generation: 2` 内で終了・閉鎖済みの旧 Run `"r-1"` から送出された外部操作 `op-ext-closed`（現行は `"r-2"`、`generation: 2`）
  - (3) 失効前に外部側で実行開始済み・結果未確認の障害注入操作 `op-ext-2`
- **操作**：
  1. Claim が `generation: 3` へ進み `generation: 2` が失効した後に `op-ext-1` が外部ゲートウェイの効果確定口へ到達する。
  2. 同一 `generation: 2` で `"r-1"` が閉鎖され `"r-2"` が現行となった後に、`op-ext-closed`（`run_id: "r-1"`, `generation: 2`）が外部ゲートウェイの効果確定口へ到達する。
  3. `"s-1"` のプロセスは `activity: "stopped"` と確認されたが、失効前開始済みの `op-ext-2` の完了・ロールバックが未確認の状態で、後続 Session `"s-2"` が巻き取りおよび危険な後続実行を要求する。その後、`op-ext-2` の無効化・環境復旧の安全確認を完了してから `"s-2"` が再要求する。
- **期待結果**：
  - 1・2 ともに外部ゲートウェイの効果確定側が失効済み `generation: 2` および同世代の閉じた `run_id: "r-1"` を**拒否**し、外部副作用が確定しない。
  - 3 で安全確認完了前の `"s-2"` 要求は**保留・拒否**され、安全確認・環境復旧完了後にのみ後続実行が許可される。

### UC-S04: Claim がないときの管理操作・Task 待機・依存評価・証拠取り込み遷移
- **前提**：
  - (A) Claim 未発行（論理的初期 Claim：`state: "unassigned"`, `generation: 1`, `run_seq: 0`）の Ticket `"tk-dep"`（`depends: ["tk-pre"]`）および `"tk-unclaimed-cancel"`。
  - (B) Run を持たない Task `"t-wait-approval"`（配下 Ticket 終端済みまたは外部承認待ち計画あり）。
  - (C) 2 つの Run 終了後フィクスチャ（いずれも現行 `active_run_id == null`）：`waiting` 中の Ticket `"tk-ev-wait"`（現在 Job `"j-ev-wait"`、閉じた旧 Run `"r-w1"`）と、Run 終了後も `doing` のまま残る Ticket `"tk-ev-doing"`（現在 Job `"j-ev-doing"`、閉じた旧 Run `"r-d1"`）。
- **操作**：
  1. **依存評価**：`"tk-pre"` 未完了時に `"tk-dep"` の依存評価を実行し、さらに `"tk-pre"` を `canceled` にした後に再評価する。
  2. **未取得 Ticket の管理中止**：権限保持運用者が未取得 Ticket `"tk-unclaimed-cancel"` を `cancel_reason` 付きで `canceled` にし、その後 `"tk-unclaimed-cancel"` の Claim 取得および終端復活を試みる。
  3. **Run を持たない Task の計画内 `waiting`**：権限保持運用者が `"t-wait-approval"` を解除条件（正本 `Task.resume`）付きで `waiting` に遷移させ、その後、外部承認の確認証拠を認可経路で取り込んで `waiting` を解除（条件充足なら `completed`）する。
  4. **Run 終了後 `waiting` / `doing` Job の証拠取り込み**：`"j-ev-wait"` および `"j-ev-doing"` に対し、(4a) 閉じた旧 Run（`"r-w1"`, `"r-d1"`）名義での直接完了送信を試み、(4b) 検証済み外部証拠を認可済み証拠取り込み経路（または現行 `active` Claim 保持 Session）で取り込んで `completed`（`result: "approved"` / `"passed"`）を確定する。
- **期待結果**：
  - 1 で Claim 未発行でも `"tk-dep"` が `todo → waiting → blocked` へ遷移し監査記録が残る。
  - 2 で未取得 Ticket の Claim が `unassigned (gen=1) → revoked (gen=2)` へ一意確定し、以後の Claim 取得および終端復活はいずれも**拒否**される。
  - 3 で Run を持たない Task が認可経路で計画内 `waiting` に入り、承認証拠の照合により解除・完了でき、監査記録が残る。
  - 4 で (4a) 閉じた旧 Run 名義の直接更新は `waiting`/`doing` いずれも**拒否**され、(4b) 認可済み証拠取り込み経路でのみ両 Job が `completed` へ原子的に確定し監査記録が残る。

### UC-S05: 保留（`paused`）解除時に `waiting`・`blocked` が残る場合
- **前提**：
  - (A) 未解消の `blocked`（正本 `resume` に「障害Xの復旧確認」を記録）を抱えた Ticket `"tk-blk"` を明示保留（`paused`）にした状態。
  - (B) 未解消の `waiting`（前提 Ticket 待ちまたは `Run.wait` 保持）を抱えた Ticket `"tk-wait"` および Task レベルの `waiting` を抱えた Task `"t-wait"` を明示保留（`paused`）にした状態。
  - (C) 負例フィクスチャとして、**Job 自体が `paused`** である `"j-paused"`（および `paused` の Task `"t-paused"`）。
- **操作**：
  1. `paused` 解除前に `"tk-blk"` / `"tk-wait"` の Run 開始（実行再開）を試み、さらに `"j-paused"`（Job 自体が `paused`）および `"t-paused"`（Task 自体が `paused`）に対して `paused → completed` を試みる。
  2. 未解消の障害・待機がそのままの状態で、`"tk-blk"`・`"tk-wait"`・`"t-wait"` の `paused` を権限保持運用者が明示解除する。
  3. `"tk-blk"` について、正本 `resume` の解除条件（障害Xの復旧確認）を満たした実状態を確認した上で、権限保持運用者（管理操作）または現行 `active` Claim を保持する認証済み Session が `blocked` からの復帰を確定する。
- **期待結果**：
  - 1 はいずれも**拒否**される（保留解除前の実行再開、および Job・Task 双方の `paused → completed` を禁止）。
  - 2 で無条件に `doing` へ復元されず、`"tk-blk"` は `blocked` へ、`"tk-wait"` および `"t-wait"` は `waiting` へ再評価遷移する。
  - 3 で認可された復帰経路による実障害解消確認後にのみ、未着手なら `todo`、着手済みなら `doing` へ復帰し監査記録が残る。

### UC-S06: 親 Task 待機・障害下の独立 Ticket 開始と未起動閉鎖 Run の保存
- **前提**：
  - (A) 親 Task `"t-parent"` が別 Ticket `"tk-other"` 起因（または Task レベルの待機・障害）により `status: "waiting"` または `"blocked"` になっている。同一 `"t-parent"` 配下に、依存未解決や制約のない独立した実行可能 Ticket `"tk-indep"`（`jobs: ["j-indep"]`, `status: "todo"`, `active` Claim 保持）が存在する。
  - (B) 起動前予約として登録された Run `"r-cancel-unstarted"`（`activity: "unstarted"`, `started` 未設定）がある。
- **操作**：
  1. `"t-parent"` が `waiting` の状態、および `blocked` の状態のそれぞれで、`"tk-indep"` の `"j-indep"` を開始（`todo → doing`、新 Run 結合）する。
  2. `"r-cancel-unstarted"` を実作業開始前に取り消して閉鎖（`activity: "unstarted" → "stopped"`, `ended` と `result: "canceled"` を記録、`started` は未設定のまま）し、TTJ 1.1.0 準拠レコードとして保存・検証する。
- **期待結果**：
  - 1 で親 Task が `waiting`・`blocked` であっても（`paused`・終端ではないため）`"tk-indep"` の開始が**許可**され、親 Task の状態は維持されたまま開始履歴が追加される。
  - 2 で実作業を開始せずに閉じた `"r-cancel-unstarted"` が `started` 省略かつ終了 `result: "canceled"` で矛盾なく保存・検証に合格する。

---

## 4. 原典 §11 禁止事項（1〜18）トレーサビリティと検査項目

| No. | 禁止事項 | 仕様正本（specification.md） | 対応テスト・追加検査項目 |
|---|---|---|---|
| 1 | 人間、Bot、AI を異なる業務上の作業主体として扱う | §2 Worker, §10.1 #1 | UC-03, UC-27（**異なる外部主体で同一入力に対する拒否ガードと達成判定の完全一致を検証**） |
| 2 | `owner_type` を必須にする | §2 Worker, §10.1 #2 | UC-03 |
| 3 | Worker の種類で Job の状態遷移を分岐する | §2 Worker, §5 `waiting`, §10.1 #3 | UC-03, UC-07, UC-12, UC-27（**異なる外部主体で同一入力に対する状態遷移・拒否ガード・達成判定の完全一致を検証**） |
| 4 | `Request` を独立した業務オブジェクトにする | §2 Job, §3 別 Worker の作業と追加依頼, §10.1 #4 | UC-07, UC-09, UC-10（**契約検査：TTJ Core のデータモデル定義および公開操作一覧に独立 `Request` オブジェクト／専用操作が存在しないことを確認**） |
| 5 | Job に複雑なワークフロー構造を追加する | §2 Job, §8.1 待機・実行制御情報, §10.1 #5 | UC-06, UC-07, UC-12（**契約検査：TTJ Core の Job スキーマおよび公開操作に子 Job・個別 Owner・複雑な分岐ワークフロー表現が存在しないことを確認**） |
| 6 | Job に個別 Owner を設定する | §2 Job, §3 単一 Owner, §10.1 #6 | UC-02（および上記 #5 の契約検査） |
| 7 | Ticket に複数 Owner を設定する | §3 単一 Owner, §10.1 #7 | UC-02 |
| 8 | Ticket 確定後に Job 列を変更する | §4 確定後の変更, §10.1 #8 | UC-06, UC-12, UC-14 |
| 9 | 終了済み Ticket を再開する | §5.1 終端状態, §6 終端の保持, §10.1 #9 | UC-13, UC-14, UC-S04 |
| 10 | 循環する完了依存を許す | §7 実行依存, §8.1 待機・実行制御情報, §10.1 #10 | UC-11（未解決待機辺・解消済み除外・逆向き競合登録を含む） |
| 11 | Handoff の閲覧権限を実行権と同一視する | §9.1 Handoff, §10.1 #11 | UC-15, UC-16, UC-22 |
| 12 | Worker が同じという理由で実行権を共有させる | §3 Worker と Session の分離, §8.2, §10.1 #12 | UC-05, UC-26, UC-S01 |
| 13 | 稼働中の Session から作業を巻き取る | §8.4 巻き取り可否条件, §10.1 #13 | UC-17, UC-25, UC-S02 |
| 14 | 停止未確認の Session から作業を巻き取る | §8.3, §8.4 巻き取り可否条件, §10.1 #14 | UC-17, UC-19, UC-25, UC-S01 |
| 15 | Heartbeat 切断のみで停止確認済みとする | §8.3 Heartbeat と停止確認の区別, §10.1 #15 | UC-19 |
| 16 | Handoff を正式履歴の正本にする | §9.1 揮発性, §10.1 #16 | UC-22（再起動後の正本照合・消失横断の遅延適用拒否を含む） |
| 17 | Handoff によって無関係な Ticket を直列化する | §8.2 並列実行, §9.1 Handoff, §10.1 #17 | UC-24, UC-S06 |
| 18 | 特定の Agent・Bot・オーケストレーターへの依存を TTJ Core へ組み込む | §1 基本原則, §2 Worker, §10.1 #18 | UC-03, UC-27（**契約検査：Orca 非利用環境での動作に加え、`skills/ttj/` 配下の正本仕様・JSON 形式・Skill および Core 公開契約に Orca 固有の型・ID・接続・状態遷移への依存が一切存在しないことを確認**。[orca-adapter.md](orca-adapter.md)） |
