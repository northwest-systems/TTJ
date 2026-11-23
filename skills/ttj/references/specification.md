# TTJ Specification v1.1.0

**Task–Ticket–Job 管理規格**

- [定義] 規格版：`1.1.0`。
- [定義] 文書状態：`stable`。

## 1. 基本原則

- [定義] **TTJ**：担当や環境が変わっても、作業・申し送り・知識を安全に継続する規格。
  - [任意] 単一主体・複数主体の分担・担当引き継ぎ・長寿命 Task の継続運用に使う。
  - [必須] 必要な情報を一度だけ記録する。
  - [推奨] フィールド名はできるだけ1単語とし、意味を明確にする連結名（`owner_id`・`task_id`・`ticket_id`・`worker_id`・`session_id`・`active_run_id`・`run_seq`・`current_ticket_id`・`active_session_id`・`current_job_id`・`source_revision`・`cancel_reason` 等）を統一して使う。
  - [禁止] 不要な分類・専用の型・重複記入・不要な状態機械を増やす。
  - [禁止] 特定の Agent・Bot・オーケストレーター実装への依存を TTJ Core へ組み込む。
- [定義] **規格の識別**：`protocol` は JSON 文書の規格名と版を表す。
  - [必須] JSON で保存・交換する場合は文書ルートに `protocol: "TTJ/1.1.0"` を持つ。
  - [定義] 同じ文書に含まれる子レコードはその版を共有し、個別の転記は不要。
  - [必須] 読み手は規格名と版を確認して解釈する。
  - [任意] 識別項目は生成側が補完する。
- [定義] **ラベル**：その行の扱いだけを示し、配下の項目へ継承しない。
  - [定義] **[必須]**：準拠に必要な条件。
  - [定義] **[禁止]**：実施してはならない操作。
  - [定義] **[推奨]**：通常は採用し、外れる場合は理由を残す規則。
  - [定義] **[任意]**：必要な場合だけ選ぶ情報・操作。
  - [定義] **[定義]**：用語や値の意味。
  - [定義] **[例]**：特定の実装や値を強制しない説明例。
  - [必須] 箇条書きの各項目にラベルを付ける。

## 2. 管理単位と作業主体

- [定義] **Worker**：作業の責任を負う主体。
  - [必須] 一意に識別できる不変の `id`（接頭辞 `wkr-`）と、変更可能な表示名 `name` を持つ。
  - [禁止] 人間・Bot・Agent・AI・Service 等の種別分類を業務上の作業主体として区別する。
  - [禁止] `owner_type` や Worker 種別を必須にする、または Worker 種別で Job・Ticket の状態遷移や達成判定を分岐させる。
  - [定義] 認証方法や実行環境ごとの設定は外部 Adapter 側で管理し、TTJ Core に持ち込まない。
- [定義] **Session**：Worker が実際の作業実行を行う単位。
  - [必須] 一意に識別できる `id` と、所属する Worker を参照する `worker_id` を持つ。
  - [定義] 同じ Worker が複数の Session を持て、各 Session を個別に識別できる。
  - [禁止] Session の終了・切断・変更によって `Ticket.owner_id` を変更する。
- [定義] **Project**：作業と長期知識を保持する継続的な領域。
  - [必須] `id`（接頭辞 `pj-`）と意味の分かる `name` で識別する。
  - [例] リポジトリ1つは粒度の目安。
  - [任意] リポジトリなし、または複数リポジトリを含む領域として運用する。
  - [禁止] Task の終了・削除に連動して長期知識を失わせる。
  - [定義] 共通の作業状態や自動完了は規定しない。
- [定義] **Task**：Project 内の長寿命な成果目標を管理する単位。
  - [必須] `id`（接頭辞 `tsk-`）・`project`・`objective`・`criteria` を持つ。
  - [必須] 単一の Project に属し、目標・達成条件・複数 Ticket の管理・結果集約・現在の進捗・ブロッカー・揮発性 Handoff・最終達成判定を扱う。
  - [任意] 期限がある場合だけ `deadline` に Task の完了期限を書く。
  - [必須] 期限を記載する場合はタイムゾーン付きの ISO 8601 日時を使う。
  - [定義] 期限の超過そのものは作業状態の遷移条件にしない。
  - [定義] Task は個々の Ticket のライフサイクルより長く存在できる。
  - [禁止] 修正・再実行・再レビューによる Ticket 増加を理由に新しい Task を作成・増殖させる。
  - [禁止] Ticket の完了数だけで Task の達成率や目標達成を決定する。
- [定義] **Ticket**：単一 Worker が責任を持つ、一方向の作業単位。
  - [必須] `id`（接頭辞 `tkt-`）・`task`・`owner_id`・`objective`・`jobs` を持つ。
  - [必須] 単一の Task に属し、存在する単一の `Worker.id`（`wkr-*`）を `owner_id` で参照し、1件以上の Job を持つ。
  - [定義] Ticket は短命でよく、次 Ticket が未作成でも終了・アーカイブできる。
  - [定義] 終了判定は Job の状態から行い、独自の `criteria` は持たない。
  - [禁止] 会話・Session・接続・Run の終了だけで Ticket を終了する。
- [定義] **Job**：完了可否を `criteria` で機械的または確認可能な証拠により判断できる最小実行単位。
  - [必須] `id`（接頭辞 `job-`）・`objective`・`criteria` を持つ。
  - [必須] 単一 Ticket の `jobs` に属し、その位置で順序と親を特定する。
  - [任意] 目標だけでは手順が伝わらない場合だけ `instructions` を補う。
  - [禁止] 親 ID や順序番号を重複記入させる。
  - [禁止] Job に複雑な分岐・処理ロジック、子 Job、個別 Owner、汎用依存関係グラフ、Handoff 排他制御、実行権管理、待機制御の詳細構造を持たせる。
  - [禁止] `HumanJob`・`BotJob`・`AIJob`・`HumanWaitingJob`・`BotWaitingJob` 等の種別ごとの Job 分類を作る。
- [定義] **目標**：`objective` は Task・Ticket・Job すべてで「何を達成したいか」を表す。
  - [必須] 階層によってフィールドの意味を変えない。
  - [必須] Task・Job の `criteria` には、完了を確認できる条件を1件以上書く。
  - [禁止] 同じ目標・成果物・確認条件を別の欄へ転記させる。
- [定義] **識別**：`id` は保存中のレコードを一意に参照する値。
  - [必須] Project・Task・Ticket・Job・Run・Worker の `id` は、それぞれ `pj-`・`tsk-`・`tkt-`・`job-`・`run-`・`wkr-` の接頭辞（prefix）で始める。
  - [必須] Project と Worker は対象を判別できる名称を持つ。
  - [推奨] ID には接頭辞に続けて対象を判別しやすい短い識別値を使う。
  - [任意] 指定された接頭辞以降の形式・他レコード（Session・知識・監査記録等）の ID 形式・発行方法・保存先は運用で決める。
  - [禁止] 同じ ID を複数のレコードへ割り当てる、または Project・Task・Ticket・Job・Run・Worker に指定外の ID 接頭辞を使う。
- [定義] **記入形式**：各階層はそれぞれ必要な情報だけを持つ。
  - [任意] 別ファイル・別テーブル・別の JSON 形式で保存する。
  - [禁止] 共通化のために全階層へ同じ項目を追加する。
  - [定義] フィールドと記入例は [JSON参照形式](json-format.md) に、代表的な場面ごとの状態遷移とサンプルパターンは [代表ユースケースとサンプルパターン](use-cases.md) に示す。

## 3. 所有権と Session 分離

- [定義] **単一 Owner**：`Ticket.owner_id` は、その Ticket と配下の全 Job に責任を持つ単一の `Worker.id` を表す。
  - [必須] Ticket 作成時に存在する単一の `Worker.id` を `owner_id` に指定する。
  - [禁止] 1つの Ticket に複数 Owner を設定する。
  - [禁止] Job ごとに独立した Owner を設定する。
  - [禁止] 同じ Ticket の `owner_id` を変更する、または別 Owner で同一 Ticket を実行する。
  - [定義] 同じ Worker が複数の Ticket を同時に所有することは許可する。
- [必須] **別 Worker の作業と追加依頼**：別 Worker に独立作業や追加依頼を行う場合は、その Worker を `owner_id` とする新しい Ticket を作成する。
  - [必須] レビュー・調査・承認・操作依頼・依頼先からの追加依頼も通常の Ticket と Job で表現する。
  - [禁止] `Request` を独立した業務オブジェクトとして導入する。
  - [定義] 同じ条件であれば、どの Worker が担当しても状態遷移と達成判定は同一とする。
- [定義] **Worker と Session の分離**：責任主体（Worker）と実行単位（Session）を明確に分ける。
  - [必須] 同一作業系列（Ticket）の実行権を同時に取得できる Session は最大1つとする。
  - [禁止] 同じ Worker に属することを理由に、実行権を持たない別 Session へ実行権を自動付与・共有させ、無断実行を許可する。
  - [必須] 実行権のない Session からの状態更新・作業実行は、同一 Worker であっても拒否する。
- [定義] **直列性**：同じ Ticket 内で同時に進める Job は最大1つとし、定義順に一方向で進行する。
  - [禁止] 同じ Ticket の別 Job を同時実行する。
  - [禁止] 同じ試行の再送を二重実行する。
- [定義] **v1.0.0 からの廃止事項**：旧 `assignee`、同一 Job の共同作業の仕組み、同一 Ticket の担当交代、`Ticket.handoffs` は v1.1.0 で廃止する。

## 4. 固定計画と Ticket 増加への対応

- [必須] **正式作成**：Ticket と全 Job の定義を原子的に確定する。
  - [必須] `jobs` に Job の ID を実行順に並べる。
  - [必須] Job 列の結果で Ticket の `objective` を達成できることを確認する。
  - [必須] 正式作成時に当該 Ticket に対する論理的な初期 `Execution Claim`（`state: "unassigned"`, `session_id: null`, `active_run_id: null`, `generation: 1`, `run_seq: 0`）を成立させる（物理レコードを遅延生成する実装でも、未取得 Ticket に対する取得・中止等は同一の観測結果とする）。
  - [任意] 正式作成前の Draft やテンプレートを編集する。
- [禁止] **確定後の変更**：Ticket 確定後に Job 列と作業定義を変更しない。
  - [禁止] Job の追加・削除・移動・並べ替え、`objective`・`criteria`・`instructions` の実質変更、完了済み Job への巻き戻し・再入力。
  - [任意] 状態・結果・成果物参照・中止理由・管理記録を更新する。
- [必須] **追加・修正・再実行**：修正・再実行・再レビュー・想定外の作業は通常の新 Ticket として同一 Task 配下に作成する。
  - [禁止] 終端 Task へ新 Ticket を追加する。
  - [禁止] 修正・再実行を理由に新しい Task を増殖させる。
  - [推奨] Ticket の Job 数や作業量に上限を設ける。
- [定義] **Ticket 増加への対策**：修正・再実行・再レビューによる Ticket 増加を許容しつつ、Worker 種別を問わず起票と管理の負荷を最小化するため、準拠する実装・運用は次の機能を備える。
  - [必須] **Ticket テンプレート**：確定前は編集可能とし、確定後は Job 列を固定する。
  - [必須] **標準 Job 列**：反復する工程を再利用可能な標準定義として提供する。
  - [必須] **後続 Ticket 生成支援**：前 Ticket の成果物・結果・中止理由を引き継いだ新 Ticket 起票を支援する。
  - [必須] **前 Ticket の成果物参照と中止理由参照**：`refs`・`artifacts`・`cancel_reason` により重複作業と文脈断絶を防ぐ。
  - [必須] **Ticket 系列表示と終了 Ticket の折りたたみ**：派生・修正・再実行の系列を追いやすくし、終了済み Ticket を折りたたんで扱えるようにする。
  - [必須] **Task 単位の進捗集約と現在 Handoff 表示**：Ticket 完了数のみではなく Task の `criteria`・主要成果・ブロッカー・現在 Handoff で状況を提示する。
  - [任意] これらの機能の交換形式・保存構造・具体的な UI 実装方法は利用環境や運用に応じて選択する。
- [必須] **Task の範囲変更**：権限を持つ主体と変更前後・理由を記録する。
  - [禁止] 未達の必須範囲を無記録で除外する。
  - [禁止] 終端 Task の目標・完了条件を変更する。

## 5. 作業状態・結果分離と再開

- [定義] `todo`：対象と配下が未着手。
  - [必須] Task・Ticket・Job の初期値にする。
- [定義] `doing`：対象または配下が着手済みで未終了。
  - [定義] プロセスが現在動いている必要はない。
- [定義] `waiting`：Task・Ticket・Job が現行計画に含まれる外部結果・前提条件の成立を待機している状態。
  - [例] Task 全体の外部承認・計画イベント待ち、前提 Ticket の完了待ち、レビュー・承認結果待ち、外部コマンドや複合スクリプト内の操作結果待ち。
  - [禁止] 待機対象の実装種類や Worker 種別によって状態遷移を分岐させる。
- [定義] `paused`：明示的な判断による保留。
  - [必須] 実行再開には明示解除と、残る待機・障害の解消が必要。
  - [必須] 保留中も未解消の待機・課題を保持し、解除時に再評価する。
- [定義] `blocked`：スクリプト異常・必要な成果物の欠落・実行環境の不備・必要な作業の実施不能・復旧未完了など、現行計画では継続できない計画外の問題がある状態。
- [定義] `completed`：その単位の完了判定を満たした終端状態。
- [定義] `canceled`：権限を持つ主体が中止した終端状態。
- [定義] **status と result の分離**：`status` は進行状態、`result` は確認された作業結果・業務判定を表す。
  - [必須] Job の完了（`status: "completed"`）と業務上の判定結果（`result`：`approved`・`changes_requested`・`passed`・`failed` 等）を分離して記録する。
  - [例] レビュー Job が `status: "completed"` かつ `result: "changes_requested"`、テスト実行 Job が `status: "completed"` かつ `result: "failed"` であることはいずれも有効な状態。
  - [必須] 依頼を送信しただけ、または `Run` が終了しただけでは Job を `completed` にせず、確認可能な証拠に基づき `criteria` を満たした時点で完了とする。
  - [必須] 結果（`changes_requested`・`failed` 等）に応じて修正や再実行が必要な場合は、完了済み Job を巻き戻さず新しい Ticket を作成する。
  - [定義] 異常終了や中断時は `result` に `interrupted`・`error` 等の結果を残し、継続不能なら理由を記録して Ticket を `canceled` とする。
- [定義] **再開条件と正本保存**：`resume` は Task・Ticket・Job において「何を確認できれば再開できるか」を示す。
  - [必須] 再開・再試行・障害解除・停止判断に不可欠な条件および証拠は、必ず永続的な正本（`depends`・`Run.wait`・`Run.result`・`Ticket.cancel_reason`・対象レコードの `resume`・管理履歴）に保存する。
  - [任意] `depends`・`Run.wait`・管理履歴といった永続正本から再開条件が一意に分かる場合だけ `resume` を省略できる。
  - [禁止] 揮発性 `Handoff` に記載したことを理由に、正本側の `resume` や障害解除条件・再試行条件を省略する。
  - [禁止] 既に正本へ記録された条件を複数箇所へ不要に転記させる。
  - [必須] 中断・引き継ぎ前および重要な発見・失敗時には正本（`Run`・`resume`・管理履歴）と必要な `Handoff` を更新し、再開時は正本および実状態と照合して、条件が変わっていない失敗を根拠なく繰り返さない。
- [定義] **着手判定**：実際の開始記録で判断する。
  - [必須] 親には配下の開始履歴も含める。
  - [禁止] 作成・割り当て・計画だけを開始とみなす。
  - [禁止] 開始履歴を消す、または着手済み作業を `todo` に戻す。

### 5.1. 通常の許可経路

- [任意] `todo`：事前の判断や開始に従う。
  - [任意] 遷移先：`doing`・`waiting`・`paused`・`blocked`・`canceled`。
- [任意] `doing`：着手後の判断や結果に従う。
  - [任意] 遷移先：`waiting`・`paused`・`blocked`・`completed`・`canceled`。
- [任意] `waiting`：待機結果や新たな判断に従う。
  - [任意] 遷移先：`todo`・`doing`・`paused`・`blocked`・`completed`・`canceled`。
- [任意] `paused`：明示解除または中止の判断に従う。
  - [任意] 遷移先：`todo`・`doing`・`waiting`・`blocked`・`canceled`。
- [任意] `blocked`：課題解消・保留・中止の判断に従う。
  - [任意] 遷移先：`todo`・`doing`・`waiting`・`paused`・`canceled`。
- [禁止] `completed`・`canceled`：他の状態へ遷移しない（終端 Ticket・終端 Job・終端 Task の再開禁止）。
- [定義] **Ticket の集約**：自身がキャンセル済みでなければ、全 Job 終端時に `completed` へ集約する。
  - [必須] 第6節の集約を通常経路より優先し、Job の終端確定と同時に反映する。
  - [定義] Ticket に限り、未着手・保留中・ブロック中からも全 Job 終端による完了が成立する。

### 5.2. ガードと操作種別ごとの認可

- [必須] **遷移の検証**：通常経路または Ticket の集約規則、親子規則、および操作種別に応じた認可条件を満たす。拒否時は正本を一切変更しない。
  - [禁止] 定義外の状態・遷移、および失効した実行権・旧 `generation`・閉じた `Run.id` からの状態遷移。
  - [定義] 同じ状態での説明・観測・履歴更新は状態遷移に含めない。
- [必須] **実行操作の認可（開始前と開始後を明確に区別する）**：
  - [必須] **Run 開始・結合操作（開始前照合）**：認証済み Session と Owner の参照整合（`Session.worker_id == Ticket.owner_id`）、`active` な `Execution Claim`（現行 `session_id`・最新 `generation`）、対象現在 Job、開始状態ガード、試行枠の空き（`claim.active_run_id == null`、または第8.2節の安全閉鎖条件による旧試行解放と同時・先行の空き枠確保）、および**要求が前提とする実行系列版（`expected_run_seq` または前回試行基準）と現行 `claim.run_seq` の一致**を検証し、新 `Run.id` を原子的に `claim.active_run_id` へ結合して `claim.run_seq` を `+1` する。
  - [必須] **結合済み Run の実行更新・結果確定操作（開始後照合）**：認証済み Session と Owner の参照整合（`Session.worker_id == Ticket.owner_id`）、`active` な `Execution Claim`（現行 `session_id`・最新 `generation`）、対象現在 Job に加え、**現行 Run の非 `null` 一致（`claim.active_run_id == run.id`）**をすべて照合する。1つでも不一致（未結合・閉じた同世代旧 Run・旧 `generation`・別 Session）なら拒否する。
- [必須] **管理・評価・証拠取り込み操作の認可**：`Execution Claim` を要しない操作（および未着手 Ticket の前提評価）は、次の正当な経路に限定して許可し、すべての経路で認可主体の検証・終端状態（`completed`/`canceled`）の復活禁止・監査記録（操作主体・時刻・理由・対象・変更前後・確定世代）を必須適用する。
  - [必須] **管理操作**：権限を持つ運用主体による Task・Ticket・Job の明示保留（`paused`）・明示解除・中止（`canceled`）・障害解消確認後の `blocked` 復帰・依存変更・範囲変更。未割当（`unassigned`）または `released` の Ticket を中止する場合は Claim を `revoked`（`generation +1`）へ確定し、既に当該操作・巻き取りで `revoked` 済みの Claim を持つ旧 Ticket に対しては二重失効・追加世代加算を行わない。
  - [必須] **Task 計画内待機操作（Run を持たない Task）**：権限を持つ運用主体が、Run を持たない Task を外部承認・計画イベント待ちとして `waiting` に遷移させる（解除条件を正本 `Task.resume` または管理履歴へ必須保存）。承認・イベントの確認可能な証拠が到着した際は、権限を持つ運用主体または認可済み証拠取り込み主体が証拠を照合して `waiting` を解除し、未着手なら `todo`・着手済みなら `doing`、Task 完了条件（Ticket が1件以上すべて終端し全 `criteria` 成立）を満たしていれば `completed` へ原子的に確定する。
  - [必須] **依存評価操作**：`depends` に基づく未着手 Ticket（未割当 Ticket および **`active` Claim を持つ未着手 Ticket（現在 Job が未着手かつ `active_run_id == null`）** の双方を含む）の `todo ↔ waiting` および前提 Ticket 中止に伴う `blocked` 遷移。前提 Ticket の確定状態に基づいて決定的に評価する（依存解除の `waiting → todo` のみを行い、開始前照合・原子的結合手順なしに Run を開始しない）。
  - [必須] **証拠取り込み操作**：Run 終了後（`claim.active_run_id == null`）に到着した確認可能な外部応答・証拠の取り込み。対象 Ticket が `paused`・`blocked`・終端でなく、現在 Job が **`waiting` または（Run 終了後の）`doing`** である場合に、現行 `active` Claim を保持する認証済み Session または検証済み証拠を取り込む認可済み主体が、現在 Job の `criteria`・待機条件を照合して `completed`（`result` 確定）または待機解除（`waiting → doing` / `todo`）を原子的に確定する。閉じた旧 `Run.id` 名義・旧 `generation` からの直接更新は拒否する。
- [必須] `todo`：制約が解消し、対象と配下が未着手。
- [必須] `doing`：開始可能で、待機・保留・障害が解消している。
  - [必須] `todo` からは開始記録を同時確定し、他の状態からは開始履歴がある。
  - [必須] `waiting` 解除後は現在の未完了 Job のみから一方向に続行する。
- [必須] `waiting`：計画内の待機があり、明示保留と継続不能な課題がない。
- [必須] `paused`：明示的な判断があり、未解消の待機・課題を保持する。
- [必須] `blocked`：継続不能な課題があり、保留中なら先に明示解除している。
- [必須] `completed`：第6節の単位ごとの完了判定を満たす。
  - [必須] Task・Job では待機・保留・障害を解消済みである。
  - [必須] Task・Job の `waiting` からは、待機結果の確認可能な証拠によって `criteria`（Task では全 Ticket 終端等の完了条件も含む）が成立する。
- [必須] `canceled`：中止判断・中止理由の記録と、対象範囲の実行権失効・停止要求を確定する。
- [必須] **復帰先と保留解除の順序**：`paused` 解除前は実行を再開しない。明示解除時は保持していた未解消の障害（`blocked`）・待機（`waiting`）および前提依存を先に再評価し、残存する制約に従う（残る障害があれば `blocked`、残る待機があれば `waiting`）。すべての制約が解消した場合のみ、未着手なら `todo`、着手済みなら `doing` を選ぶ。
  - [禁止] 保留前の状態を無条件に復元する。
  - [禁止] Task・Job の `paused → completed`・`blocked → completed`。

## 6. 親子の整合性と終了判定

- [必須] **一括更新**：関連する状態・開始履歴・実行情報を原子的に確定する。
  - [必須] 最初の実作業開始時は Job・Ticket・`todo` の Task を同時に `doing` にする。
  - [必須] Task が `waiting`・`blocked` なら状態を維持し、開始履歴を追加する。
  - [定義] 親子の状態は同じでなくてよい。
- [定義] **現在 Job**：`jobs` の先頭から見て最初の非終端 Job。
  - [必須] 全先行 Job が `completed` または `canceled` になるまで開始・再試行・完了しない。
  - [必須] 切り替え時は現在 Job と Ticket に残る制約を評価する。
  - [禁止] 後続 Job の事前情報だけで現在の Ticket 状態を変える。
  - [禁止] 完了済み Job への巻き戻しや再入力を行う。
- [必須] **待機・障害の連動**：現在 Job の待機・障害を Ticket にも反映する。
  - [必須] 待機では Ticket の保留・障害・終端を、障害では保留・終端を優先する。
  - [必須] 保留中の待機・課題を保持し、解除時に再評価する。
  - [必須] Job が終端になったら、次の Job または Ticket の終了を判定する。
- [必須] **実行制限**：対象 Ticket または現在 Job の保留（`paused`）・障害（`blocked`）・終端（`completed`/`canceled`）中、および未解除の待機（`waiting`）中は、新しい作業（新規 Job や未解除待機中の次工程）を開始しない。
  - [必須] `waiting` 中の継続・再開では、第8.4節の順序に従い待機条件と再開位置を引き継いで状態を再評価し、待機解除で `doing` へ遷移した後に同一未完了 Job の残処理を実行する（待機条件の成立により直接 `completed` へ遷移する場合は新 Run 開始を要しない）。
  - [禁止] 親 Task の保留（`paused`）・終端（`completed`/`canceled`）中の開始。
  - [禁止] 親 Task が別 Ticket 起因で `waiting`・`blocked` であることだけを理由に、独立した実行可能 Ticket・Job の開始を停止・拒否する。
  - [必須] 明示保留（`paused`）では、既存実行の安全な停止・中断を求める。
- [必須] **Job 完了**：確認可能な証拠に基づき Job の `criteria` をすべて満たした時点で `completed` とし、`result` および必要な成果物参照を残す。
- [任意] **Job の中止**：必要な Job だけを理由を残して `canceled` にし、後続を続ける。
  - [必須] **中止確定と同時の試行権限即時無効化**：該当 Job に結合中の Run がある場合、Job の `canceled` 確定と**同時に**当該 `Run.id` の試行権限を即座に無効化・閉鎖（終了 `result: "canceled"` 等を記録し、効果確定側で当該 `Run.id` の外部書込みを即時拒否可能な状態にし、`claim.active_run_id = null`、`claim.run_seq + 1` を原子確定）し、停止要求を発行する。Ticket 全体の `Execution Claim` を失効させる必要はない。
  - [必須] **後続 Job 開始前の停止・安全確認（即時無効化と分離）**：中止確定時の即時無効化とは別に、後続 Job を開始する前に前の Job の実行に対する「停止確認**または**副作用重複・残存副作用がない安全条件」を確認する（失効前開始済みの外部操作や残存副作用の疑いがある場合は安全確認完了まで後続 Job 開始を保留する）。
  - [禁止] 中止した Job が目標を達成したとみなす。
- [必須] **Ticket 完了**：全 Job が `completed` または `canceled` になれば `completed` にする。
  - [必須] 自身がすでに `canceled` なら、その状態を維持する。
  - [必須] Ticket が終端になった時点で対象 Ticket の `Execution Claim` が `active` であれば解放または失効（`session_id = null`, `active_run_id = null`, `generation +1`）させる（既に `released`/`revoked` 済みなら二重変更しない）。
  - [禁止] Ticket 独自の完了条件や承認項目を追加する。
  - [定義] 全 Job が中止でも処理終了を表し、目標達成を保証しない。
  - [必須] Ticket 終了時に正式な成果・結果・状態を永続化する。
  - [定義] 次 Ticket が作成されていなくても、終了済み Ticket をアーカイブできる。
- [必須] **Task 完了**：Task の `criteria` を満たし、未解決の必須成果物がない。
  - [必須] Ticket が1件以上あり、すべて終端状態。
  - [必須] 中止 Job・中止 Ticket や差し戻し結果で未達となった必須範囲を解決するか、権限を持つ主体が理由付きで除外する。
  - [禁止] Ticket の処理終了や完了 Ticket 数だけを目標達成の証拠にする。
- [必須] **Task・Ticket の中止**：未終端の子孫を同時に `canceled` にし、Ticket では中止理由（`cancel_reason`）を必須記録する。
  - [必須] 終端済みの子を保持し、対象範囲の `Execution Claim` が未失効（`unassigned`・`active`・`released`）であれば `revoked`（`generation +1`, `session_id = null`, `active_run_id = null`、再取得永久禁止）として失効させ、実行中の Run へ停止を要求する。既に当該巻き取り・中止で `revoked` 済みの Claim に対しては二重失効・追加世代加算を行わない。
  - [必須] 明示的な中止を、全 Job 終端による Ticket 完了より優先する。
  - [禁止] 遅延結果で終端状態を復活させる、または停止要求だけで終了確認にする。
- [必須] **終端の保持**：保存中の状態・成果・中止理由・履歴を改ざんしない。
  - [禁止] プロセスの正常終了だけで作業を完了する。
  - [必須] 追加・訂正・修正・再実行は新 Ticket にし、親 Task も終端なら別の非終端 Task を使う。
  - [定義] 終了後の保存・削除方法と保持期限は規格の対象外。

## 7. 実行依存と情報共有参照

- [任意] **実行依存**：`depends` に、先に処理を終える必要がある前提 Ticket の ID を記録する。
  - [定義] 複数指定はすべての処理完了を待ち、空の一覧は前提なしを意味する。
  - [任意] 別 Task・別 Project の Ticket を参照する。
  - [必須] 参照先または検証可能な完了記録の存在を確認する。
  - [禁止] 自己依存・重複参照・循環する完了依存。
- [必須] **完了依存グラフの辺と競合確定境界**：
  - [定義] 完了依存グラフの有向辺に含めるのは、(1) 非終端 Ticket の `depends` による前提 Ticket への辺、および (2) 表示上の状態（`waiting`・通常の明示保留 `paused`・一時障害 `blocked`・停止/移譲中）によらず、**非終端 Ticket・Job が保持しているすべての未解決待機**における `Run.wait` の別 Ticket 完了待ちへの辺とする。
  - [必須] 解消済み・解除済みとなった `Run.wait` は実行履歴として保持しつつ完了依存グラフの辺から除外する（待機が解消・解除された後に逆向きの完了依存を登録することは循環に該当せず許可される）。
  - [必須] 完了依存の登録・変更（静的 `depends` と動的 `Run.wait` の混在を含む）は、循環を閉じ得る逆向き登録が並行に発行されても双方同時に成功しない原子的な循環検証・確定境界で評価し、高々一方のみを確定し他方を拒否する（拒否時は正本無変更）。
- [必須] **実行依存の評価**：全依存先が `completed` になった場合だけ開始する。
  - [必須] 前提待ちは `waiting` とし、保留・障害・終端を優先する。未割当 Ticket に加え、`active` Claim を持つ未着手 Ticket（現在 Job 未着手かつ `active_run_id == null`）も認可済み依存評価経路で `waiting → todo`（前提中止で継続不能なら `waiting → blocked`）へ再評価・遷移できる。
  - [禁止] Ticket 自身のキャンセル・削除・不明状態を完了とみなす。
  - [必須] 前提 Ticket のキャンセルで継続不能なら、保留中・終端を除いて `blocked` とする。
  - [定義] 完了 Ticket に中止 Job が含まれることは許容される。
  - [必須] 必要な入力や成果の有無は、作業内容と結果から確認する。
- [任意] **実行依存の変更**：未着手の Ticket に限り、理由と変更前後を残して修正する。
  - [必須] 参照・循環・現在の制約を再評価する。
  - [禁止] 着手後・終端後の変更、固定 Job の意味を変える修正。
- [任意] **情報共有参照**：`refs` に、重複作業や文脈断絶を避けるための関連 Ticket 参照を記録する。
  - [例] 前 Ticket の成果物・完了結果・中止理由の参照、派生元 Ticket、修正対象 Ticket、再実行元 Ticket。
  - [必須] `refs` は情報共有専用とし、実行依存 `depends` と明確に区別する。
  - [禁止] `refs` を開始条件・完了待機・状態遷移の制約として解釈する。
  - [任意] 参照要素には `ticket_id` と任意の補足ラベル（`kind`：`derived_from`・`fixes`・`rerun_of`・`related` 等）や短文メモ（`note`）を記載する。
  - [禁止] `refs` の `kind` によって業務ロジックや状態遷移を分岐させる。
- [定義] **課題対応・修正・再実行**：ブロックされた課題の解消、差し戻しへの修正、異常終了からの再実行はいずれも通常の新 Ticket で行う。
  - [必須] 元 Ticket が継続不能な場合は、`cancel_reason` を記録して `canceled` とし、新 Ticket の `refs` で元 Ticket を参照する。
  - [必須] 元 Ticket が継続可能で一時的に `blocked` となった場合は、元 Ticket の正本（`resume`・管理履歴）に障害内容と解除条件を記録し、修復用新 Ticket の `refs` に元 Ticket を記録して、元 Ticket 側で実際の課題解消を確認してから再開する。
  - [推奨] 修復用 Ticket を、元 Ticket の処理完了待ち（`depends`）にしない。
  - [禁止] 別 Ticket の完了だけを理由に元 Ticket の障害を自動解除する。

## 8. Run・実行権・生存判定・巻き取り

### 8.1. Run（実行試行の正本）

- [定義] **Run**：Job の1回の実行試行（Execution / Attempt に相当）を記録する正本。
  - [必須] `id`（接頭辞 `run-`）・`ticket_id`・`job`・`owner_id`・`session_id`・`generation` を追跡する。
  - [必須] **実作業の開始事実と `started`**：実作業の開始を確定した Run では `started`（開始時刻）を必須記録する。起動前予約（`activity: "unstarted"`）のまま、または実作業を開始せずに取り消して `stopped` 等で終了閉鎖した Run には開始時刻が存在しないため、`started` を省略できる。
  - [必須] Run の終了確定時に `result` を必須記録し、終了時刻を確認できる場合は `ended` も記録する。
  - [必須] `Run.owner_id` は対象 Ticket の `owner_id` と一致し、`Run.session_id` はその Worker に属する実行権保持 Session と一致する。
  - [必須] 同一 Ticket で有効な実行試行（`Claim.active_run_id` に結合された Run）は同時に最大1つとする。
  - [必須] 正常終了・異常終了・Job 単独中止・Ticket 中止・失効・安全閉鎖により Run が終了確定した時点で、その `Run.id` の試行権限を即座に閉じ（`Claim.active_run_id` を `null` へ解放し、`Claim.run_seq` を `+1`）、閉じた `Run.id` からの後着結果・業務状態更新・外部効果確定・Handoff 更新を拒否する。
  - [定義] 独立した `Execution` や `Attempt` オブジェクトを別途設けず、`Run` を実行試行の正本として統一する。
  - [必須] すべての作業主体による実行を同じ `Run` 形式で記録する。
- [必須] **待機・実行制御情報**：待機中の制御情報（待機中の Job ID、待機対象の参照、解除条件、待機開始時刻、再開位置、タイムアウト、`session_id`、Run 識別子、および解消・解除時の `resolved` 時刻）は Job に複雑構造として追加せず、`Run`（`wait` 領域等）および実行管理側で保持する。
  - [必須] `waiting` 解除後は現在の未完了 Job のみから一方向に続行し、古い `Run`（閉じた `Run.id`）や旧 `generation` からの状態更新を拒否する。
  - [必須] 別 Ticket の結果を待つ場合も、第7節の完了依存グラフ（`depends` と非終端の未解決 `Run.wait`）に循環が存在しないことを保証する。
- [定義] **複合スクリプトと副作用管理**：複合スクリプトは Ticket 単位で管理する。
  - [定義] 細かな内部関数やコマンドすべてを個別の Job として定義する必要はないが、各 Job の完了判定には確認可能な証拠が必要。
  - [必須] 複合スクリプト内の外部操作待ちも通常の `waiting` で扱い、待機対象の Worker 種別は問わない。
  - [任意] 独立作業が必要なら別 Worker を Owner とする別 Ticket を生成し、単純な操作結果の通知で完結するなら必ずしも別 Ticket を作らない。
  - [必須] 異常終了で継続不能となった場合は `cancel_reason` を記録して Ticket を `canceled` とし、元 Ticket は再開せず、修正・再実行は新 Ticket で管理する。
  - [必須] 次の3操作を明確に区別する：1. 生存中（`alive`）の Run への監視再接続（新 Run 作成・世代更新なし）、2. 停止確認（`stopped`）および安全確認後の同一 Worker 別 Session への Claim 移譲による再開（`generation +1`）、3. 業務上の再実行（新 Ticket 作成）。
  - [必須] 障害注入等の副作用を持つスクリプトでは、Ticket の中止と環境復旧を分離し、実行停止時または失効前に開始済みの外部操作がある場合に副作用が残存している可能性があるなら、安全確認・環境復旧の完了まで後続の危険な実行・巻き取りを禁止・保留する。
  - [推奨] Session 障害で進行不能となった場合は、停止・安全確認の上で新 Ticket による再実行を基本とする。

### 8.2. Execution Claim（実行権の正本）と generation フェンシング

- [定義] **Execution Claim**：Ticket 単位（一意キー：`ticket_id`）の実行権の正本。
  - [必須] `task_id`・`ticket_id`・`owner_id`・`session_id`（`active` 以外は `null`）・`active_run_id`（現在有効な単一試行の `Run.id`、未実行・終了・非 `active` 時は `null`）・`generation`・`run_seq`（試行結合・安全閉鎖・終了のたびに単調増加する実行系列版、初期値 `0`）・`state`（`unassigned`・`active`・`released`・`revoked`）を持つ。
  - [必須] 同一作業系列（`ticket_id`）で同時に実行権を保持できる Session は最大1つとする。独立した別 Ticket は別キー（別 `ticket_id`）の Claim を持ち並列実行できる。
  - [必須] **`generation` の初期値・増分・非巻戻し規則**：Ticket 作成時の初期 Claim（`state: "unassigned"`, `session_id: null`, `active_run_id: null`, `run_seq: 0`）の `generation` は正の整数 **`1`** とし、取得・再取得・移譲・解放・失効の切替1回ごとに厳密に **`+1`** 単調増加させる。確定済み Claim の `generation` は決して巻き戻さない。
  - [必須] **`session_id` と `active_run_id` の整合**：`state` が `unassigned`・`released`・`revoked` のときは必ず `session_id: null` かつ `active_run_id: null` とする。`state` が `active` のときのみ非 `null` の生存する適格 `Session.id`（`Session.worker_id == owner_id`）を持つ。終了済み Session には新しい `active` Claim を決して付与しない。
  - [必須] **Claim の許可遷移と再取得規則**：
    - [必須] `unassigned → active`（初回取得、`generation +1`）：対象 Ticket が非終端、過去実行の残存なし、かつ取得要求 Session が生存する適格 Session の場合のみ可。
    - [必須] `unassigned → revoked`（未取得 Ticket の管理中止・新 Ticket 巻き取りによる失効、`generation +1`）：終端 Ticket につき再取得永久禁止。
    - [必須] `active → active`（同一 Worker 別 Session への安全移譲、`generation +1`、`active_run_id = null`）：全過去 Run が `stopped`・`active_run_id == null`・安全条件成立・`expected_generation` と `expected_run_seq` の一致・移譲先が生存する適格 Session を満たす場合のみ原子的 CAS で可。
    - [必須] `active → released`（正常解放、`generation +1`）：Session の正常解放または Ticket の `completed` 確定時。
    - [必須] `active → revoked`（強制失効、`generation +1`）：中止・強制回収・新 Ticket への巻き取り確定時。
    - [必須] `released → active`（再取得、`generation +1`）：対象 Ticket が**非終端**かつ過去 Run に未停止（`alive`/`unknown`）や安全未確認が残っていない場合に限り、同一 `owner_id` の生存する適格 Session に許可（終端 Ticket では再取得禁止）。
    - [必須] `released → revoked`（一時解放済み Ticket の管理中止・新 Ticket 巻き取りによる失効、`generation +1`）：終端または後続移管済みにつき再取得永久禁止。
    - [必須] `revoked → active`（再取得、`generation +1`）：対象 Ticket が**非終端かつ後続 Ticket へ巻き取り移管済みでない**場合に限り、過去 Run の全停止確認（`stopped`）と安全条件成立のもと同一 `owner_id` の生存する適格 Session に許可。**終端 Ticket（`completed`/`canceled`）または後続 Ticket へ巻き取り移管済みの Ticket では `revoked` からの再取得を永久禁止**する。
  - [必須] **開始前と開始後の照合分離**：
    - [必須] **Run 開始・結合前**：認証済み Session（`Session.worker_id == Ticket.owner_id`）、`claim.state == "active"`、`claim.session_id == session_id`、`claim.generation == generation`、対象現在 Job、開始ガード、空き枠（`claim.active_run_id == null`、または下記「試行権限の安全閉鎖」による解放）、および**要求が前提とする実行系列版（`expected_run_seq` または前回試行基準）と現行 `claim.run_seq` の一致**を同一競合境界で検証し、原子的操作で新 `Run.id` を `claim.active_run_id` へ結合して `claim.run_seq` を `+1` する。これにより、同じ前回 Run（例：`run-1`）を前提に同時発行された並行再試行のうち、先着 `run-2` が開始・終了して `active_run_id` が再び `null` に戻った後に到着した後着要求も古い前提として拒否し、`run-2` を前提に取り直した正当な再試行のみ許可する。
    - [必須] **開始後の実行更新・結果確定**：認証済み Session（`Session.worker_id == Ticket.owner_id`）、`claim.state == "active"`、`claim.session_id == session_id`、`claim.generation == generation`、対象現在 Job、および**現行 Run の非 `null` 一致（`claim.active_run_id == run.id`）**をすべて照合し、1つでも不一致なら拒否する。
  - [必須] **同一 Claim 内での試行権限の安全閉鎖（v1.0.0「停止または安全条件」の実現）**：
    - [必須] 同一計画内の通常再試行および Job 単独中止後の後続 Job 開始に限り、旧試行 `run-1` の `activity` が `stopped` でなく `unknown` であっても、(1) `run-1` の試行権限を永久閉鎖して終了結果（`result`）を記録し、(2) 副作用を伴う操作では外部フェンシングにより旧試行の外部影響を確実に無効化でき、かつ残存副作用の疑いがない安全条件を満たすなら、**`Execution Claim` 自体を失効させずに `claim.active_run_id` を `null` へ解放し `claim.run_seq` を `+1`** した上で、同一 `active` Claim（同一 `session_id`・同一 `generation`）へ新 `Run` `run-2` を結合できる（なお Job 単独中止時は第6節のとおり中止確定と同時に `run-1` の試行権限を即時無効化し、後続 Job 開始前に停止確認または安全条件を別途確認する。残存副作用の疑いがあり安全条件を満たせない場合は停止確認・復旧まで次開始を禁止する）。
    - [必須] 上記安全閉鎖は同一 Claim 内の通常再試行・後続 Job 開始に限定し、別 Session・別 Worker への**巻き取り**には適用しない（巻き取りでは全過去 Run の `stopped` 確認かつ安全条件の両方必須を維持する）。
  - [必須] **副作用防止のための外部フェンシング**：外部リソース変更・障害注入等の副作用を防ぐ必要がある操作では、効果の確定側（実行環境・ゲートウェイ・外部更新口）でも失効済み権限（旧 `generation`・失効済み Claim・閉じた同世代 `Run.id`・Job 単独中止で即時無効化された `Run.id`）を拒否し、失効前に開始済みの操作が残存し得る場合は安全確認完了まで後続実行を保留することを準拠必須条件とする。
- [必須] **開始条件と保護**：
  - [必須] 前提 Ticket は `completed`、先行 Job は `completed` または `canceled`。
  - [必須] 対象 Ticket および現在 Job が `todo` または `doing`、親 Task が保留（`paused`）・終端（`completed`/`canceled`）ではない（親 Task が別 Ticket 起因で `waiting`・`blocked` であっても独立した実行可能 Ticket・Job は開始できる）。
  - [必須] 実行主体の Session は `Ticket.owner_id` と同じ Worker に属する生存中の適格 Session であり、当該 Ticket の `active` な `Execution Claim` を保持している。
  - [必須] 同一計画内の通常再試行・再起動および後続 Job 開始には新しい `Run` を使い、前の実行の停止確認**または**上記安全条件を確認する（v1.0.0 の規則を維持）。
  - [禁止] 監視の再接続や同一操作の再送だけで、別の `Run` として二重登録する。
- [必須] **回復可能な中断**：同じ計画で再開できる一時障害を計画の破綻と区別する。
  - [必須] 元の `status` が `doing` なら、その状態を維持する。
  - [禁止] Run の終了・切断・失敗だけで Job や Ticket の完了・中止・ブロックを判断する。
  - [禁止] 計画された再試行・タイムアウト処理が残る間、時間超過だけで `blocked` にする。
- [任意] **並列実行と優先度**：資源競合を制御しつつ、独立した別 Ticket を並列実行する。
  - [禁止] 同一 Task に属することや Handoff が単一スロットであることを理由に、無関係な Ticket の並列実行を遮る・直列化する。
  - [任意] `priority` を開始順の判断材料にする。
  - [禁止] 低い優先度だけで `paused` にする。

### 8.3. プロセス・Session 生存判定

- [定義] **生存観測**：Run の実行プロセス・Session の状態を `activity` と観測時刻 `observed` で追跡する。
  - [定義] `unstarted`：起動前（予約登録のみで実作業未開始）。
  - [定義] `alive`：生存確認済み（稼働中）。
  - [定義] `stopped`：停止確認済み（信頼できる監視により、当該 Run の実行主体および配下の実行コンテキストが終了し、新規の副作用操作を開始できない状態であることを確認した確定値）。
  - [定義] `unknown`：状態不明（確認不能・生存観測の有効期間超過）。
- [必須] **Heartbeat と停止確認の区別**：Heartbeat の途絶・監視タイムアウト・無出力・接続断・停止要求の送信単独で `stopped`（停止確認済み）と確定しない。
  - [必須] 生存観測（`alive`）の有効期間を定め、最終観測時刻から有効期間を超過した場合は実効値を `unknown` にする。一方、同一 Run で一度確定した `stopped` は終端観測値であり、時間経過で `unknown` に戻さない。
  - [必須] 停止確認（`stopped`）と、停止前に開始された外部操作・障害注入の残存副作用がないことの確認（安全条件）を区別して評価する。
- [必須] **独立監視による停止証拠記録と業務更新の分離**：実行主体から独立した信頼できる監視機構は、対象 Claim や `Run.id` の試行権限が失効・終了した後であっても、専用の観測記録経路により対象 Run の `activity: "stopped"` と `observed`・停止証拠のみを記録できる（監査記録に監視主体と観測時刻を残す）。旧 Run による業務状態・結果・Handoff の更新経路とは厳格に分離し、後者は常に拒否する。
- [任意] **activity の許可遷移**：起動前は `unstarted` から `unknown`・`alive`・`stopped` に移る（実作業開始前に取り消して `unstarted → stopped` とした Run は `started` を持たず終了 `result` のみ持つ）。
  - [任意] 同じ Run では `unknown → alive`・`unknown → stopped`・`alive → unknown`・`alive → stopped` に移る。
  - [禁止] 同じ Run の `stopped → alive`・`stopped → unknown`、起動後の `unstarted` 復帰。
  - [必須] 再起動は新しい `Run` とし、前 Run の履歴を残す。
  - [任意] プロセスを監視しない作業では観測項目を省略できるが、停止未確認のまま巻き取りを許可してはならない。
- [推奨] 失効した Run の残存プロセスは独立した監視・管理機構により終了処理する。
- [禁止] 観測結果だけで作業を `completed` や `canceled` にする。

### 8.4. 巻き取り可否条件と単一原子確定契約

- [必須] **巻き取り可否条件**（過去の実行が残存する場合は、Claim が `revoked` 等で `session_id: null` であっても「未割当」とはみなさず、稼働中・状態不明の拒否ガードを優先する）：
  - [必須] **未割当**：`Execution Claim` が `unassigned`（または再取得可能な `released`/`revoked`）であり、**かつ対象 Ticket に未停止（`alive`/`unknown`）または安全未確認の過去 Run が残存しない**場合のみ、Handoff 参照可・新規実行権取得可（終端 Ticket・巻き取り移管済み Ticket は不可、`paused` は明示解除が必要）。
  - [必須] **稼働中**（対象 Ticket に `activity: "alive"` の Run が残存、Claim 失効済みであっても同じ）：Handoff 参照は可、巻き取り・再取得は不可。
  - [必須] **`waiting` かつ稼働中**（`status: "waiting"` かつ `activity: "alive"`）：Handoff 参照は可、巻き取りは不可。
  - [必須] **停止確認済み**（対象 Ticket の過去 Run がすべて `activity: "stopped"` かつ `active_run_id == null`、`waiting` 中の停止確認済みを含む）：Handoff 参照は可、巻き取りは**安全条件成立時のみ可**（停止前に開始された外部操作や副作用の残存が疑われる場合は安全確認・復旧完了まで保留）。
  - [必須] **状態不明**（対象 Ticket に `activity: "unknown"` の Run が残存、Heartbeat 途絶・Claim 失効済みだが停止未確認を含む）：Handoff 参照は可、巻き取り・再取得は不可。
- [必須] **切替時の競合境界（陳腐化した安全確認の防止）**：旧実行による新規 Run 開始・試行閉鎖と巻き取りは**同一の競合境界**で排他する。巻き取り確定時は、停止確認・安全確認を行った時点の `expected_generation` および `expected_run_seq`（実行系列版）が現行 Claim の `generation`・`run_seq` と一致し、かつ全過去 Run が `stopped`・`active_run_id == null` であることを原子的に検証する。確認後に同世代で別 Run が開始された場合（その Run が切替前に終了して `active_run_id` が再び `null` に戻った場合を含む）は `run_seq` 不一致により巻き取りを拒否し、古い安全確認の流用を防止する。
- [必須] **新 Ticket 移管巻き取りの公開契約（単一の原子的確定）**：新 Ticket へ移す巻き取り（別 Worker への巻き取り、または障害で進行不能な場合の再実行）の公開契約は**単一の原子的確定**に一本化し、未確定の中間予約状態を公開しない。内部実装で複数段階の書込みや永続予約を用いる場合であっても、**途中段階を読み取り・実行開始・成功応答から一切観測できない**こと（外部からは単一原子確定と完全に同一の観測結果となること）を必須とする。
  - [必須] **1. 確定済み再送と衝突の照合順序**：要求（`operation_id`, 認可主体, `expected_generation`, `expected_run_seq`, `claimant_session_id`, `target_owner_id`, 後続計画・中止理由等の実質パラメータ）を受理した際は、次の順序で照合する。
    - [必須] (1a) **確定済み同一要求の再送（最優先）**：旧 Claim に同一 `operation_id` の確定済み `takeover` 記録が存在し、かつ**認可主体および実質パラメータがすべて一致**する場合、元の `expected_generation`・`expected_run_seq` が現行値より古くても世代不一致とせず、既存の確定結果（同一 `successor_ticket_id` 等）をそのまま冪等に返却する（追加作成・世代加算なし）。
    - [必須] (1b) **同一 `operation_id` の主体・パラメータ不一致**：既存の確定済み `operation_id` を再利用して認可主体または実質パラメータが異なる要求は**明示的に拒否**する（正本無変更）。
    - [必須] (1c) **別要求の衝突拒否**：旧 Claim に別の `operation_id` の確定済み `takeover` が既に存在する場合、提示された世代が現行世代と一致していても確定済み巻き取りとの衝突（旧 Ticket 終端済み）として**拒否**する（正本無変更）。
  - [必須] **2. 未確定（確定前失敗）後の再送と前提検証**：まだ確定していない要求（確定前障害後の再送を含む）は、現行の正本前提（非終端の親 Task・非終端の旧 Ticket `tkt-1`・全過去 Run `stopped`・`active_run_id == null`・安全条件成立・`generation == expected_generation`・`run_seq == expected_run_seq`・生存する適格 Session）を再検証し、1つでも満たさなければ拒否する（正本無変更・途中状態の外部露出ゼロ）。
  - [必須] **3. 一体で原子的に確定する範囲**：前提を満たした同時巻き取り要求のうち**厳密に最大1件**のみ、次の (a)〜(d) を単一の原子的確定として一体確定する。
    - [必須] (a) 旧 Ticket `tkt-1` を中止理由（`cancel_reason`）付きで `status: "canceled"`（未終端 Job も `canceled`）へ終端化する（`tkt-1.owner_id` は変更しない）。
    - [必須] (b) 旧 Claim を `state: "revoked"`, `session_id: null`, `active_run_id: null`, `generation: g + 1`（再取得永久禁止、二重失効・追加世代加算なし）へ確定する。
    - [必須] (c) `target_owner_id` を `owner_id` とする後続 Ticket `tkt-2`（ID=`successor_ticket_id`、`refs` に `tkt-1` を記録）とその初期 Claim（`state: "unassigned"`, `generation: 1`, `run_seq: 0`）を確定する。要求元が確定時点で生存する適格 Session（`worker_id == target_owner_id`・未終了）であり初回取得を一体で行う場合は `unassigned(gen=1) → active(gen=2)` とし、要求元 Session が終了済みであれば新しい実行権（`active` Claim）を決して付与せず `unassigned(gen=1)` とする（または即時取得要求を未確定のまま拒否する）。公開後に勝者 Session が通常終了した場合も、確定済み Claim の世代は巻き戻さず、通常の停止確認・安全確認＋巻き取りで `generation + 1` する。
    - [必須] (d) 同一操作再送用の永続結果（`operation_id`・認可主体・実質パラメータ・`successor_ticket_id` 等）を旧 Claim の `takeover`（および監査記録）に確定保存する。
  - [必須] **4. 競合操作（親 Task 中止・旧 Job 証拠確定）との線形化境界**：巻き取り、親 Task 中止、旧 Job の検証済み証拠確定の先後は**正本での確定順**で決まり、後続操作は先に確定した正本状態を検証する。
    - [必須] **親 Task 中止が先**：終端 Task への後続 Ticket 作成を含む巻き取りは拒否し、終端済み親 Task および `tkt-1` を書き換えない。
    - [必須] **旧 Job 証拠確定（`tkt-1` 完了集約）が先**：完了済み `tkt-1` に対する巻き取りは拒否し、完了済み `tkt-1` を書き換えない。
    - [必須] **巻き取りが先**：`tkt-1` は `canceled` 確定のため後着の旧 Job 証拠確定は終端保護で拒否され、後続の親 Task 中止は既に存在する後続 Ticket `tkt-2` も子孫として対象に含む（`tkt-2` も `canceled`・Claim `revoked`）。
- [必須] **同一 Worker の安全な移譲（同一 Ticket 上の Session 切替）**：旧 Ticket の未完了 Job から安全に継続できる条件を満たす場合、同一競合境界で認可主体・非終端の親 Task および Ticket・`expected_generation = g`・`expected_run_seq = s`・全過去 Run `stopped`・`active_run_id == null`・安全条件成立・移譲先 `S_new` が生存する適格 Session であることを原子的に検証し、CAS で旧 Claim を `state: "active"`, `session_id: S_new`, `active_run_id: null`, `generation: g + 1` に更新する（確定済み世代は巻き戻さない）。その後、対象 Ticket・現在 Job の状態に応じて次のとおり処理する：
  - [必須] (a) **未解決の待機（`waiting`）がある場合**：直前 Run の待機条件（`Run.wait`）・再開位置・正本 `resume` を引き継いで状態を再評価し、待機未成立なら `waiting` 維持（新規作業開始禁止）、証拠により `criteria` 成立なら完了確定、待機解除で `doing` へ遷移した後に新 `Run` を開始する。
  - [必須] (b) **`paused`・`blocked` がある場合**：`paused` は明示解除まで実行再開を禁止し、`blocked` は実課題解消確認まで開始を禁止する。
  - [必須] (c) **待機・保留・障害がいずれもなく既に実行可能な状態（`todo` または `doing`）である場合**：第8.2節の開始前照合に従い新しい `Run` を原子的に結合（`active_run_id = run-new`, `run_seq +1`）して、現在の未完了 Job から一方向に続行する。

## 9. Handoff と長期知識

### 9.1. Task 所属の揮発性 Handoff

- [定義] **Handoff**：`Task` に所属する揮発性の単一スロット（Task ごとに最大1つ）。
  - [必須] 少なくとも `task_id`、揮発スロット側の書込連番 `revision`、および**消失をまたいで比較可能な永続正本側の単調確定版 `source_revision`** を持ち、状況に応じて `current_ticket_id`・`owner_id`・`active_session_id`・`current_job_id`・`source_generation`・`activity`・`context`（および `stop_position`・`blockers`・`notes`・`updated` 等）を保持する。
  - [必須] Task への参照権限を持つ複数の Session が同時に閲覧できる。
  - [必須] 閲覧権限と実行権限を厳格に分離し、Handoff を閲覧できることを実行権の保持と同一視しない。
  - [禁止] Handoff 自体にコマンド実行機能や状態遷移を直接引き起こす実行権限を持たせる。
  - [定義] **揮発性**：Handoff は正本から得られる現在状況の要約表示と揮発性の補足キャッシュであり、正式な監査履歴・成果物・再開条件の正本としない。
  - [必須] Handoff は必要に応じて再生成・破棄でき、Handoff が消失しても Task の状態、Ticket の履歴、Job の達成結果、Ticket の中止理由、Artifact、正本 `resume`、Run 履歴（`wait` 含む）、Execution Claim、管理履歴（および永続正本側の単調確定版）は失われず、正本のみから再開判断と Handoff 再生成を行える。
  - [必須] Task 内の無関係な Ticket の並列実行を Handoff によって遮ったり直列化したりしない。
  - [必須] **更新入口の3分離と鮮度・スロットインスタンス保証**：Handoff の更新経路を次の3つに明確分離し、拒否時は現行 Handoff を変更しない。
    - [必須] **1. Run 起因の更新**：認証済み Session（`Session.worker_id == Ticket.owner_id`）、表示対象 `current_ticket_id` の `active` Claim（`session_id`・最新 `generation`）、および **`claim.active_run_id != null && claim.active_run_id == run_id`**（非 `null` 一致）をすべて照合する。`active_run_id == null` の空き枠へ閉じた旧 `run_id` 名義で到着した更新は常に拒否する。
    - [必須] **2. Run によらない Session 更新**：認証済み Session（`Session.worker_id == Ticket.owner_id`）、表示対象 `current_ticket_id` の `active` Claim（`session_id`・最新 `generation`）、および**閉じた `run_id` 名義でないこと**（Run 非紐付けの Session 更新入口）を検証する。
    - [必須] **3. 管理再生成（Claim 不要）**：権限を持つ管理・システム主体として認可された処理が、永続正本の最新確定版（`source_revision`）を読み取り、**書込み時点でもその `source_revision` が永続正本の現行最新であることを検証（または同等の原子的再生成保証）**した上で閲覧キャッシュのみを再構築する。いかなる Session にも実行権を付与せず、正本状態を変更しない。
    - [必須] **消失横断および同一正本版での上書き防止**：経路1・2・3のすべてにおいて、提示された `source_revision` が永続正本側の現行確定版および現行 Handoff の `source_revision` より古い更新を拒否する。さらに、**永続正本版（`source_revision`）が変わらないまま揮発スロットが消失し新スロットが再生成された後に、消失前に準備された旧更新（同じ `source_revision`・より大きい `revision`）が遅れて到着しても、新スロットを上書きできないこと**（内部スロットインスタンス／世代照合または同等の条件付き書込み契約により異なるスロットインスタンス向けの旧更新を拒否すること）を保証する。
  - [必須] Ticket 終了時に正式な成果・結果・状態を永続化し、Handoff を長期記憶として扱わない。

### 9.2. Project の長期知識

- [定義] **長期知識**：Project の `knowledge` に再利用する内容と根拠を保持する。
  - [必須] `content` に残す内容、`basis` に根拠・判断理由・出典をまとめる。
  - [例] 未記載の事実・暗黙のルール・過去の判断・将来の展望。
  - [必須] 元の Task・Ticket がなくても内容と判断経緯を理解できる。
  - [禁止] 未確認の内容や展望を確認済みの事実として扱う。
  - [必須] 再利用する知見は Project の知識へ選別して移し、Handoff の全文を無条件に複製しない。
- [定義] **知識の状態**：現在使う知識と、過去の知識を2つの値で区別する（v1.0.0 の綴りを維持）。
  - [定義] `active`：現在の知識として採用している内容。
  - [定義] `supersed`：置換・廃止等により現在は使わない内容。
  - [必須] 新規記録は未指定でも `active` とする。
  - [例] 「次回は配信を自動化する予定」という計画も `active` の知識。
  - [必須] 状態の変更主体・時刻・理由を履歴に残す。
  - [必須] 通常は `active`、振り返り・再発障害・判断理由の確認では `supersed` も参照する。
- [必須] **知識の履歴**：内容や根拠の訂正は新しい記録にして元を残す。
  - [必須] 置換時は `supersedes` で同じ Project の現在の知識を参照する。
  - [必須] 旧記録の `supersed` と新記録の `active` を同時確定する。
  - [任意] 後継がない場合も、理由を残して `active` から `supersed` にする。
  - [任意] 同じ内容を再び使う場合は、理由を残して `supersed` から `active` に戻す。
  - [禁止] 内容・根拠の上書き、自己参照、循環。

## 10. 禁止事項・記録・適合確認

### 10.1. 禁止事項（不変条件）

- [禁止] 1. 人間、Bot、AI 等を異なる業務上の作業主体として扱う。
- [禁止] 2. `owner_type` を必須にする。
- [禁止] 3. Worker の種類で Job の状態遷移を分岐する。
- [禁止] 4. `Request` を独立した業務オブジェクトにする。
- [禁止] 5. Job に複雑なワークフロー構造を追加する。
- [禁止] 6. Job に個別 Owner を設定する。
- [禁止] 7. Ticket に複数 Owner を設定する。
- [禁止] 8. Ticket 確定後に Job 列を変更する。
- [禁止] 9. 終了済み Ticket を再開する。
- [禁止] 10. 循環する完了依存を許す。
- [禁止] 11. Handoff の閲覧権限を実行権と同一視する。
- [禁止] 12. Worker が同じという理由で実行権を共有させる。
- [禁止] 13. 稼働中の Session から作業を巻き取る。
- [禁止] 14. 停止未確認の Session から作業を巻き取る。
- [禁止] 15. Heartbeat 切断のみで停止確認済みとする。
- [禁止] 16. Handoff を正式履歴の正本にする。
- [禁止] 17. Handoff によって無関係な Ticket を直列化する。
- [禁止] 18. 特定の Agent・Bot・オーケストレーターへの依存を TTJ Core へ組み込む。

### 10.2. 記録と適合確認

- [任意] **保存形式**：意味と参照を保てれば、保存先・データ構造・UI 等を選べる。
  - [禁止] 特定の保存期間や自動削除を規格として強制する。
- [必須] **管理記録**：作成・状態変更・開始・結果・中止・異常停止・試行権限の安全閉鎖および即時無効化・実行権取得および失効・巻き取り確定・依存変更・範囲変更・期限変更・知識の採用・独立監視による停止確認・管理/評価/証拠取り込み操作を追跡する。
  - [必須] 変更前後・主体（`worker_id`・`session_id`、または管理・監視主体）・時刻・理由・確定世代と関連する実行（`run`・`generation`）を識別する。
  - [任意] 時刻・世代等は更新手段が補完する。
  - [禁止] 補完できない事実を捏造する。
- [必須] **整合性**：現在状態・実行権・現行 Run 結合（`active_run_id`・`run_seq`）・管理情報・履歴を原子的に確定する。
  - [禁止] 同じ操作の再送で状態変更や副作用を二重確定する。
  - [禁止] 古い Run（閉じた `Run.id`）・失効した実行権・旧 `generation`・競合した世代・陳腐化した `run_seq` から業務状態を更新する。
  - [任意] 無効な結果を状態へ反映せず監査記録に残す。
- [必須] **確認結果**：成果と検証を一度記録し、親では参照する。
  - [禁止] 記録で分かる結果や管理情報を各階層へ転記させる。
- [必須] **準拠**：本書のすべての [必須] と [禁止] を満たす。
  - [必須] 部分適用では未対応範囲を示す。
  - [禁止] JSON や Skill だけで永続化・原子性・排他・外部副作用防止を保証したと称する。
- [必須] **適合確認**：ID 接頭辞（`pj-`・`tsk-`・`tkt-`・`job-`・`run-`・`wkr-`）・所有権・Session 分離・Run 開始前/開始後の照合分離・並行再試行の古い前提拒否・Job 単独中止時の即時試行権限無効化と後続開始ガード分離・試行安全閉鎖・Job の `status`/`result` 分離・依存循環拒否（保留/一時障害で保持中の未解決待機辺および逆向き競合登録含む）・Handoff 更新3経路分離と消失横断/同一正本版上書き防止・`Execution Claim` の単一原子巻き取り契約および競合線形化・外部フェンシング・管理/依存評価（`active` Claim 付き未着手 Ticket 含む）/証拠取り込み認可・保留解除時の再評価・Worker 非依存性を合わせて検証する。
  - [必須] 全 Job 終端時の Ticket 完了と、Ticket 自身の中止優先を確認する。
  - [必須] 中止 Job を含む Ticket の処理終了や Ticket 完了数と、Task の目標達成を区別する。
  - [必須] 同一 Worker の別 Session 無断実行、同世代の閉じた旧 Run 結果、空き枠復帰後の古い前提再試行、稼働中・状態不明・失効後残存の巻き取り、切替前に新 Run が介入した巻き取り、同時巻き取りの多重成功、旧 `generation`・Job 単独中止直後の旧 Run からの復帰更新および外部更新確定を拒否する。
  - [必須] Handoff 消失および再起動後も Task・Ticket・Job・中止理由・Artifact・正本 `resume`・Run・Execution Claim の正式履歴から再開判断でき、消失前スナップショット（異正本版・同正本版双方）および閉じた Run 名義の遅延適用が拒否されることを確認する。
  - [必須] 独立した Ticket の並列実行と、次 Ticket 未作成での終了 Ticket アーカイブが妨げられないことを確認する。
  - [必須] 短期の揮発性 Handoff と、採用中（`active`）および過去（`supersed`）の長期知識を区別して参照できる。
