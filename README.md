# TTJ

**Task–Ticket–Job 管理規格 · v0.1.0 · Draft**

人間・LLM・プログラムが共同で作業するための、実装に依存しない作業管理規格。

```text
Project（リポジトリ）
└── Task（リリース項目）
    ├── Ticket（独立した作業・論理セッション）
    │   └── Job 1 → Job 2 → …（作成時に固定した直列実行単位）
    └── Ticket（並列実行可能）
        └── Job 1 → Job 2 → …
```

Ticket内のJobは固定して直列に進め、想定外の追加作業は別Ticketに分ける。作業の管理状態 `status`、実行プロセスの観測状態 `activity`、実行順序の判断材料 `priority` を分離する。プロセスの停止だけでは、作業の完了やブロックを意味しない。

- [規格本文](skills/ttj/references/specification.md)：状態遷移、ガード、親子の整合性、実行・履歴、適合条件。
- [TTJ Skill](skills/ttj/SKILL.md)：LLMによる計画・実行・再開・点検の手順。`skills/ttj` フォルダー全体を、利用環境のSkill配置先にコピーして使う。参照文書も一緒に配置する。

規格の正本は `skills/ttj/references/specification.md`。Skillはその適用手順であり、単独で排他制御や永続化を保証する実装ではない。

ライセンス：[Apache-2.0](LICENSE)。
