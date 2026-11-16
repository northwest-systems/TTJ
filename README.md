# TTJ

**Task–Ticket–Job 管理規格 · v0.1.0 · Draft**

- [定義] **目的**：担当や環境が変わっても、作業と知識を継続する。
  - [任意] 人間のみ・LLMのみ・共同作業・担当交代・専属で運用する。
  - [必須] 必要な情報を一度だけ書き、専用の分類や重複記入を増やさない。

```text
Project（名前を持つ継続領域）
└── Task（目標と完了条件）
    ├── Ticket A ─┐
    ├── Ticket B ─┴─→ Ticket C（A・Bの処理終了後に開始）
    └── Ticket（固定したJob列）
        └── Job 1 → Job 2 → …（同じJob内の共同作業は可能）
```

- [必須] **規格の識別**：独立したJSON文書のルートに `protocol: "TTJ/0.1.0"` を持つ。
  - [定義] 子レコードは同じ版を共有し、転記は不要。
- [必須] **Project**：`id` と意味の分かる `name` を持つ。
  - [任意] リポジトリを伴わない作業にも使う。
  - [例] リポジトリ1つは粒度の目安。
- [定義] **目標**：`objective` はTask・Ticket・Jobで同じ意味の「達成したいこと」。
- [必須] **Task**：`id`・`project`・`objective`・`criteria` を持つ。
  - [任意] 期限がある場合だけ `deadline` をタイムゾーン付き日時で記載する。
- [必須] **Ticket**：`id`・`task`・`objective`・`jobs` を持つ。
  - [必須] 全Jobが `completed` または `canceled` になれば `completed` にする。
  - [必須] Ticket自身がキャンセル済みなら `canceled` を維持する。
  - [禁止] Ticket独自の完了条件を追加する。
- [必須] **Job**：`id`・`objective`・`criteria` を持つ。
  - [必須] 実行順はTicketの `jobs` に従う。
  - [任意] 必要なJobだけ中止し、後続を続ける。
- [任意] **担当**：人・LLM・LLMセッション等の識別値を `assignee` の一覧に指定する。
  - [任意] Job固有の担当を指定し、省略時はTicketの担当を使う。
  - [任意] 複数担当が同じJobを共同で進める。
  - [必須] 必要人数や全員の確認はJobの `criteria` で表す。
- [任意] **依存**：`depends` に前提Ticketを指定する。
- [定義] **状態の分離**：`status` は作業状態、`activity` はプロセス観測。
  - [任意] 再開条件が参照や申し送りから分からない場合だけ `resume` を書く。
- [必須] **課題への対応**：元Ticketの `handoffs` に課題を書き、通常の新Ticketで対応する。
  - [必須] 対応するTicketの `handoffs` に元TicketのIDと対応内容を書く。
  - [禁止] 専用の関係フィールドや分類を追加する。
- [定義] **実行記録**：RunはJobの1回の実行と結果をまとめる。
  - [必須] 人間の作業も同じ形で記録し、再試行ごとに新しいRunを使う。
  - [任意] プロセスを監視する場合だけ観測情報を加える。
- [必須] **記憶**：短期の `handoffs` とProjectの長期知識 `knowledge` を分ける。
  - [定義] `active` は現在使う知識、`supersed` は現在使わない過去の知識。
  - [必須] 新規の知識や予定は `active` とし、通常は `active` だけを参照する。
  - [必須] 過去の内容・根拠・判断理由も残す。
- [定義] **Ticketの完了**：処理終了を表し、目標達成そのものは保証しない。
  - [必須] Taskの目標達成はTaskの `criteria` で確認する。
- [任意] **終了後の保管**：保存・削除方法と保持期限は運用で決める。
  - [必須] Projectの知識はTask・Ticketの保存有無に依存させない。
- [定義] **[規格本文](skills/ttj/references/specification.md)**：作業規則の正本。
- [定義] **[JSON参照形式](skills/ttj/references/json-format.md)**：項目ごとの記入方法と例。
- [定義] **[TTJ Skill](skills/ttj/SKILL.md)**：LLMが規格を適用する手順。
  - [任意] `skills/ttj` 全体を利用環境のSkill配置先へコピーする。
  - [必須] コピーする場合は参照文書も含める。
  - [禁止] Skill単独で永続化や排他を保証したと称する。
- [定義] **ライセンス**：[Apache-2.0](LICENSE)。
