# TTJ Specification v0.1.0

**Task–Ticket–Job 管理規格**

- Status: Draft
- Version: 0.1.0
- 対象：人間・LLM・プログラムによる共同作業

## 1. 適用範囲と用語

TTJは、固定したJob列を持つTicketを基本実行単位とする、実装に依存しない作業管理規格である。Ticket間は原則独立して並列に進め、Ticket内は直列に進める。作業状態とプロセスの観測を分離し、切断・再試行・引き継ぎを経ても計画と履歴を保持する。

**MUST** は必須、**MUST NOT** は禁止、**SHOULD** は推奨、**MAY** は任意を表す。本書の表に示す条件も規範とする。推奨から外れる場合は理由と影響を記録する SHOULD。

| 用語 | 定義・必須条件 |
|---|---|
| Project | 1つのリポジトリ。リポジトリ識別情報を持つ。継続的な管理単位とし、自動完了を要求しない。 |
| Task | 1つのProjectに属するリリース項目。目的、成果物、受け入れ条件を持つ。 |
| Ticket | 1つのTaskに属する独立した作業・論理セッション。目的、受け入れ条件、1件以上の固定Job列を持つ。 |
| Job | 1つのTicketに属する固定された直列実行単位。目的、作業内容、完了条件、1つの主担当を持つ。 |
| Session | Ticketの論理的な作業コンテキスト。会話の切替や接続終了でTicketは終了しない。 |
| Run | プロセスの起動・接続単位。毎回新しいIDで識別する。 |
| Attempt | Jobの1回の実行試行。再試行は新しいAttemptとする。 |

Session・Run・Attemptは管理階層に含めない。Ticketは複数Run、Jobは複数Attemptを持てる。AttemptはJob、実行主体、使用したRun（ある場合）、実行権の世代に関連付ける MUST。人間の作業はプロセスなしでも開始・結果を記録する MUST。

## 2. 計画と不変性

Ticketとその全Jobを単一の原子的操作で正式作成する MUST。正式作成前のDraftは正式なTicketではなく、編集できる MAY。

正式作成後、Jobの追加・削除・移動・並べ替え、および目的・作業内容・完了条件の変更を禁止する MUST NOT。順序番号は1からの連番とする MUST。実行状態、成果、履歴、主担当の引き継ぎは更新できる MAY。ただし引き継ぎで作業内容や有効な実行権を重複させてはならない MUST NOT。

Ticketの目的・受け入れ条件を変更して、固定Jobの意味や完了基準を実質的に変えてはならない MUST NOT。想定外の追加作業は新しいTicketに分離する MUST。Job数や作業量の上限を設ける SHOULD。

Taskには非終端の間だけTicketを追加できる MAY。Taskの受け入れ条件や必須範囲を変更する場合は、権限を持つ主体の判断と変更履歴を記録する MUST。未達の必須範囲を無記録で除外してはならない MUST NOT。

## 3. 作業状態と遷移

`status` はTask・Ticket・Jobの管理状態、`activity` はJob・Ticketに関連するプロセスの観測、`priority` は実行順序の判断材料である。相互に代用してはならない MUST NOT。Projectの共通statusは規定しない。Taskのactivityは配下から集約できる MAY。

### 3.1. status

| 値 | 意味 | その状態で必要な情報・条件 |
|---|---|---|
| `todo` | 未着手 | 対象および配下に作業開始履歴がない。 |
| `doing` | 着手済み・未完了 | 対象または配下の作業開始履歴がある。プロセスの生存は不要。 |
| `waiting` | 計画された待機 | `wait_reason`、`wait_target`、`resume_condition`。現行計画で許容される待機である。 |
| `paused` | 明示的な保留 | `pause_reason`、`previous_status`、`paused_at`。明示的な保留判断がある。 |
| `blocked` | 想定外の継続不能 | `block_reason`、`blocked_at`、`resolution_condition`。現行計画で継続できない問題がある。 |
| `completed` | 完了 | 第4節の完了条件を満たす。終端状態。 |
| `canceled` | キャンセル | 権限を持つ主体によるキャンセル判断がある。終端状態。 |

初期状態は `todo` とする MUST。`waiting`・`paused`・`blocked` は着手前にも使える。作業開始履歴はAttemptまたは人間の開始記録で判定し、親は配下の開始履歴も含める。作成・割り当て・計画の記録だけでは着手としない。開始履歴を消去したり、着手済みの対象を `todo` に戻したりしてはならない MUST NOT。

### 3.2. 許可遷移表

| 現在 | 遷移先 |
|---|---|
| `todo` | doing, waiting, paused, blocked, canceled |
| `doing` | waiting, paused, blocked, completed, canceled |
| `waiting` | todo, doing, paused, blocked, completed, canceled |
| `paused` | todo, doing, waiting, blocked, canceled |
| `blocked` | todo, doing, paused, canceled |
| `completed` | なし |
| `canceled` | なし |

遷移は、この表・次のガード・第4節の階層規則をすべて満たす場合に限り許可する MUST。それ以外の遷移は禁止する MUST NOT。同一状態での観測・説明・履歴の更新は遷移ではない。

### 3.3. ガード

以下は遷移先ごとの共通条件であり、許可表にない経路を追加しない。複数の条件が該当する場合はすべて満たす MUST。

| 遷移先 | 必須条件 |
|---|---|
| `todo` | 待機・保留・障害が解消し、対象と配下が未着手。 |
| `doing` | 開始または再開可能で、未解消の待機・保留・障害がない。`todo` からは作業開始記録と同時に確定し、他の状態からは着手履歴を必要とする。 |
| `waiting` | 現行計画内の未解消の待機があり、明示的な保留と継続不能な障害がない。`paused` からは保留が解除されている。 |
| `paused` | 明示的な保留判断がある。既存の待機・障害情報を保持する。 |
| `blocked` | 現行計画では継続不能な問題を確認した。`paused` からは保留を解除し、問題が未解消である。 |
| `completed` | 完了条件をすべて満たし、未解消の待機・保留・継続障害がない。`waiting` からは待機結果により条件が成立した。 |
| `canceled` | 権限を持つ主体のキャンセル判断があり、子の伝播と実行権失効を同時に確定する。 |

`waiting`・`paused`・`blocked` から `doing` に戻す際は、それぞれ待機解消・保留解除・障害解消を確認する MUST。着手前なら `todo` を経て開始する。`previous_status` は参考情報にすぎず、保留解除後の状態は現在の条件で決める MUST。

`blocked → completed` と `paused → completed` は禁止する MUST NOT。障害・保留の解除後に許可された状態へ戻し、改めて完了を評価する。外部結果だけで完了条件が成立した場合は、着手前でも `waiting → completed` を許可する MAY。ただしJobの順序と親の完了条件を省略してはならない MUST NOT。

予定された再試行・タイムアウト処理が残る間、時間超過だけで `blocked` にしてはならない MUST NOT。プロセス異常終了・Attempt失敗だけでも自動的に `blocked`・`canceled`・`completed` にしてはならない MUST NOT。同じ計画で回復可能な着手済み作業は `doing` を維持する。

## 4. 階層間の整合性と完了

親子のstatusは一致しなくてよい。ただし以下を保証する MUST。

### 4.1. 開始・待機・障害・保留

- `todo` のTask・Ticket配下では作業を開始しない。最初のJob開始時、Ticketと、`todo` のTaskを同時に `doing` にする。Ticketが事前待機等にある場合は、先にガードに従って開始可能な状態へ戻す。
- 現在Jobとは、先頭から見て最初の未完了Jobをいう。先行Jobがすべて `completed` でなければ、Jobの開始・再試行・完了を確定しない。後続Jobの事前待機・障害の記録だけでは、現在JobやTicketの状態を変更しない。現在Jobが切り替わる場合は、そのJobの既存の待機・障害も同じ操作で評価する。
- 現在Jobが `waiting` になったら、Ticketも同時に `waiting` にする。ただしTicketの `paused`・`blocked`・終端状態を優先する。
- 現在Jobが `blocked` になったら、Ticketも同時に `blocked` にする。ただし `paused` と終端状態を優先し、保留中は障害情報を保持して解除時に評価する。
- Job起因の待機・障害が解消したら、Ticket側でも解除条件を評価する。未着手なら `todo`、着手済みなら `doing`、待機結果で全完了条件が成立すれば `completed` を選べる。別の待機・障害や明示保留が残る場合はそれに従う。子の完了だけでは親を自動完了しない。
- Ticketが `waiting`・`paused`・`blocked` の間、新しいJob・Attemptを開始しない。Jobのstatusを親と同名に変更する必要はない。未解消の待機があるJobの後続も開始しない。
- Taskが `paused` または終端状態の間、配下で新しいJob・Attemptを開始しない。Taskが `waiting`・`blocked` という理由だけで独立した全Ticketを停止せず、個別の開始条件を評価する。
- Task・Ticket・Jobの明示保留では、対象範囲の既存実行に安全な停止・中断を要求する。停止まで `activity = alive` でもよい。停止処理と観測は保留中も許可する。

必要な親子の状態変更、開始履歴、関連情報は原子的に確定し、外部には整合した状態だけを公開する MUST。Taskが待機・障害中に独立Ticketが着手した場合、Taskの状態を維持して開始履歴を記録し、以後 `todo` へ戻してはならない MUST NOT。

### 4.2. 完了

| 対象 | `completed` の必須条件 |
|---|---|
| Job | 定義済みの完了条件すべてと、成果・検証結果の記録。先行Jobがすべて `completed`。 |
| Ticket | 全Jobが `completed`、Ticketの受け入れ条件を充足し、継続障害がない。 |
| Task | Ticketが1件以上あり、全Ticketが終端状態。必須範囲を担う有効なTicketがすべて `completed`、Taskの受け入れ条件を充足し、未解決の必須成果物がない。 |

キャンセル済みTicketの必須範囲は、置換等で解決済み、または権限を持つ主体が理由を記録して必須範囲から除外済みであることをTask完了時に確認する MUST。キャンセルだけで必須範囲が満たされたと扱ってはならない MUST NOT。

正常なプロセス終了だけでは完了にしない MUST NOT。終端状態からの再開、終端TaskへのTicket追加、確定済み成果・完了履歴の破壊的変更は禁止する MUST NOT。完了後の追加・訂正作業は新しいTicketで扱い、元の記録を参照する。親Taskも終端なら非終端の別Taskを使用する。

### 4.3. キャンセル

TaskまたはTicketのキャンセルでは、未終端の全子孫も同時に `canceled` にする MUST。`completed`・`canceled` の子はそのまま保持する。JobだけをキャンセルしてTicketを継続することはできない。Jobのキャンセルが必要ならTicket全体をキャンセルする MUST。

キャンセル時は対象範囲の実行権を失効させ、稼働中プロセスに停止を要求する MUST。終了を確認するまでは `stopped` と記録してはならない MUST NOT。遅延した結果で状態を復活・完了させてはならない MUST NOT。結果そのものは監査履歴に記録できる MAY。

## 5. Ticket間の関係

通常のTicketは他Ticketの実行結果への依存を持たず、独立して実行できるよう設計する SHOULD。順序が必要な通常工程は同一TicketのJobにまとめる SHOULD。例外となる実行依存は障害回復の `repairs` とし、`related` に依存を隠してはならない MUST NOT。

| 関係（新Ticket → 対象Ticket） | 必須条件 |
|---|---|
| `repairs` | 対象の障害を解消する。対象は `blocked`、または障害情報を保持する `paused`。修復元・対象・障害を記録する。 |
| `replaces` | `canceled` の対象を置き換える。新しい固定Job列を定義し、引き継ぐ範囲・成果への参照と元の履歴を保持する。 |
| `related` | 実行依存を持たない関連付け。 |

追加の修復作業は新Ticketにする MUST。修復Ticketの完了だけで元Ticketを再開せず、元の `resolution_condition` が満たされたことを確認する MUST。修復・置換関係では自己参照および両関係をたどる循環を禁止する MUST NOT。置換元のキャンセルと新Ticket登録は一貫して処理する SHOULD。終端Taskに修復・置換Ticketを追加する例外は設けない。

## 6. 実行権・直列実行・再試行

Job・Attemptを開始するには、以下をすべて満たす MUST。

1. Jobが `todo` または `doing` で、全先行Jobが `completed`。
2. Ticketが `todo` または `doing`、Taskが `paused`・終端状態ではない。
3. 対象に未解消の待機・保留・継続障害がなく、主担当と必要な開始条件が確定している。
4. 排他的な実行権を取得し、同一Ticketに競合するJob・Attemptがない。
5. 前回実行の停止を確認したか、実行権の失効と副作用の重複防止等により安全な再試行条件を確認している。

同一Ticketで有効な実行権を持つJobは最大1つ、同一Jobで有効なAttemptも最大1つとする MUST。Attemptの終了またはJobの終端確定時には、その実行権を解放・失効させる MUST。複数WorkerではLeaseまたは同等の排他制御を使う SHOULD。実行権の世代とrevisionを検証し、失効した実行や競合更新の結果で現在状態を上書きしてはならない MUST NOT。

実行権の失効だけではプロセス停止の証明にならない。以前のRunが生存している可能性がある間、無条件に再実行または後続Jobの実行を開始してはならない MUST NOT。完了後も生存するプロセスは新規作業に使わず、クリーンアップする SHOULD。

再試行では新しいAttemptを記録し、固定Jobの定義は変えない MUST。異なるTicketはリソース競合を制御した上で並列実行できる MAY。`priority`、資源、実行枠は開始順序の判断材料であり、即時実行を保証しない。低優先度だけで `paused` にしてはならない MUST NOT。priorityの値域は実装に委ねる。

## 7. activityと監視

activityは特定の対象Runに対する観測であり、作業進捗ではない。Job・Ticketでactivityを示す場合は対象Runを特定する MUST。同じRunを参照するJobとTicketは整合した観測を使う MUST。複数Runを集約表示する場合は、対象と集約規則を明示し、対象Runの観測と混同してはならない MUST NOT。

| state | 定義 |
|---|---|
| `unstarted` | 対象にRunの履歴がない。人間だけの作業でも使える。 |
| `alive` | 対象Runの生存を確認しており、観測が有効。 |
| `stopped` | 対象Runの終了を確認した。停止要求だけでは不可。 |
| `unknown` | Runはあるが、生存・終了を確認できない。起動・接続直後も含む。 |

activityには `observed_at` を保持し、RunがあればそのIDも保持する MUST。`unstarted` の時刻はRun履歴なしを確認した時刻とする。生存観測の有効期間・失効条件を定め、失効した `alive` は実効状態を `unknown` とする MUST。過去の観測は履歴に残す MUST。

| 対象 | 許可する変化 |
|---|---|
| Runなし → 最初のRun | `unstarted → unknown / alive / stopped`。後二者には実際の観測が必要。 |
| 同一Run | `unknown → alive / stopped`、`alive → unknown / stopped`。`stopped` は確定した終了であり変更しない。 |
| 新しいRunへの対象切替 | 以前のactivityから `unknown / alive / stopped`。新Runの記録・観測に基づき、前Runの履歴を保持する。 |

Run履歴ができた後に `unstarted` に戻してはならない MUST NOT。同一Runの `stopped` を観測の経時失効だけで `unknown` に戻してはならない MUST NOT。対象切替は前Runの停止を意味せず、第6節の安全条件を満たす必要がある。

Supervisorなど、Job実行主体とは別の信頼された監視機構で起動・生存・終了・観測時刻・異常理由・復旧時のRun照合を管理する SHOULD。監視停止や無出力だけで終了を断定してはならない MUST NOT。activityの変化だけでstatusを完了・キャンセルにしてはならない MUST NOT。

`doing + stopped` は再開待ち、`waiting + alive` はコマンド待ち、`completed + alive` は残存プロセス、`canceled + unknown` は停止未確認などを表せる。これらは代表例であり、組み合わせの許可表ではない。

## 8. データと履歴

以下は論理的な必須情報であり、保存形式やフィールド名は固定しない。

| 対象 | 必須情報 |
|---|---|
| 全エンティティ | 一意ID、種別、作成・更新時刻、revision。Project以外は単一の親ID。 |
| Project | リポジトリ識別情報。 |
| Task・Ticket・Job | status、目的、受け入れ条件または完了条件、開始履歴、成果・検証結果への参照。未着手・成果なしも識別可能にする。 |
| Ticket・Job | 関連Runとactivity（Runなしも表現）、実行権とその世代。 |
| Job | 順序番号、固定作業内容、主担当、Attempt履歴。 |
| 該当する対象 | 待機・保留・障害の情報、修復・置換関係、キャンセル判断と理由。 |

priorityは必要に応じて保持する MAY。状態遷移、開始・終了・異常停止、引き継ぎ、修復・置換、定義可能な範囲の変更を履歴に記録する MUST。未着手への判定に必要な開始履歴を保持する MUST。

状態変更イベントには一意なイベントID、対象種別・ID、変更前後の状態、時刻、主体、理由、確定後revisionを含める MUST。関連する親子更新は同一操作として識別可能にする MUST。実行イベントには該当するRun・Attempt・実行権の世代を関連付ける MUST。

現在状態、必要な管理情報、履歴は原子的に確定し、一貫性を維持する MUST。同一操作の再送で状態遷移や副作用を二重確定せず、revision不一致・古いRun・失効した実行権からの更新を拒否する MUST。遅延イベントや無効な結果は、現在状態に適用せず履歴として保持できる MAY。

## 9. 適合範囲

TTJ準拠実装は本書のすべてのMUST・MUST NOTを満たす MUST。人手の記録でもよいが、Skillやチェックリストだけで永続化・原子性・排他・冪等性が保証されたと主張してはならない MUST NOT。部分適用では対象範囲と未対応の要件を明示する MUST。

実装言語、保存形式、データベース、UI、LLM提供元、通信方式、ID生成、監視方式、Job上限、priorityの段階数、スケジューリング方式、ソース管理サービスは規定しない。いずれの選択でも、状態遷移と不変条件は維持する MUST。

## 10. 適合確認

確認対象は、4階層と単一親、固定Job列、許可遷移とガード、階層整合性、完了条件、直列・排他、再試行、観測、永続履歴、競合防止・冪等性、修復・置換の全要件とする MUST。次の例だけを通過しても完全な適合を意味しない。

| 確認例 | 期待結果 |
|---|---|
| Task・Ticket・Jobを正式作成 | 初期statusはすべて `todo`。Ticketと全Jobを一括確定。 |
| Job列の追加・削除・並べ替え・定義変更 | 拒否。Draftの編集は許可。 |
| 初回Job開始 | Job・Ticket・`todo` のTaskの開始と状態更新を一括確定。 |
| 未着手の待機・保留・障害から原因解消後に `todo` | 許可。着手履歴があれば拒否。 |
| 条件なしの `waiting`、理由・解消条件なしの `blocked` | 拒否。 |
| 未着手の `waiting → doing` | 拒否。`todo` を経て開始記録とともに `doing`。 |
| `waiting → completed`、外部結果で全条件成立 | 許可。ただし先行Job・親の条件も満たす。 |
| `blocked / paused → completed`、終端からの復帰 | 拒否。 |
| 現在Jobの待機・障害 | 親Ticketも同時更新。保留中は保留を維持し、理由を保持。 |
| Taskが `waiting / blocked`、独立Ticketの開始条件成立 | 許可。Taskの開始履歴を保持し、以後 `todo` 復帰は禁止。 |
| TaskまたはTicketが `paused`、新Job・Attempt開始 | 拒否。既存実行の停止処理・観測は許可。 |
| Job 1未完了でJob 2を開始・完了、同Ticketで複数実行権取得 | 拒否。 |
| Ticket・Taskのキャンセル | 未終端の全子孫のキャンセルと実行権失効を一括確定。完了済みの子は保持。 |
| JobだけをキャンセルしてTicketを継続 | 拒否。 |
| プロセスの異常終了・Attempt失敗、同計画で回復可能 | statusは `doing` を維持。終了確認ならactivityは `stopped`。 |
| 生存観測の失効、監視停止 | 実効activityは `unknown`。終了を確認済みのRunは `stopped` を維持。 |
| 同一Runの `stopped → alive / unknown` | 拒否。対象変更には新Runが必要。 |
| 前Runの生存可能性が残る状態で無条件に再実行 | 拒否。 |
| 古いRun・失効実行権・revision不一致の状態更新 | 拒否。イベント再送でも二重確定しない。 |
| 修復Ticketの完了だけで元Ticket再開、修復・置換の循環 | 拒否。元の解消条件を別途確認。 |
| 全Job完了前のTicket完了、未確定Ticketが残るTask完了 | 拒否。 |
| キャンセル済み必須範囲が未解決のTask完了 | 拒否。 |
| 終端TaskへのTicket追加、会話・Run終了によるTicket自動終了 | 拒否。 |
