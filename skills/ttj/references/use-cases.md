# TTJ 1.1.0 代表ユースケースとサンプルパターン（UC-01〜UC-10）

本文書は、[規格本文](specification.md) および [JSON参照形式](json-format.md) を実運用へ適用する際に迷いやすい代表的な **10場面（UC-01〜UC-10）** について、**「どのような場面か（シナリオ）」「規格上のポイント（どの状態・フィールドを使うか）」「サンプル JSON パターン」** をまとめた実践リファレンスである。

すべての JSON サンプルは文書ルートに `"protocol": "TTJ/1.1.0"` を持ち、確定済みの7種の ID 接頭辞（`pj-`・`tsk-`・`tkt-`・`job-`・`run-`・`wkr-`・`ssn-`）に従う完全な JSON として記述している。

---

## 目次

1. [UC-01: 基本の実行フロー](#uc-01-基本の実行フロー)
2. [UC-02: 複数 Worker による並列実行](#uc-02-複数-worker-による並列実行)
3. [UC-03: Ticket 間の依存関係（`depends_on` / `depends`）とブロック解除](#uc-03-ticket-間の依存関係depends_on--dependsとブロック解除)
4. [UC-04: 外部待機（`waiting`）と一時停止（`paused`）の使い分け](#uc-04-外部待機waitingと一時停止pausedの使い分け)
5. [UC-05: Job の失敗（`failed`）と同一 Ticket・同一 Session 内での再試行](#uc-05-job-の失敗failedと同一-ticket同一-session-内での再試行)
6. [UC-06: Worker / Session 再開不可時の旧 Ticket 中止と新規 Ticket への引き継ぎ](#uc-06-worker--session-再開不可時の旧-ticket-中止と新規-ticket-への引き継ぎ)
7. [UC-07: Task 所属の揮発性 Handoff による引き継ぎ](#uc-07-task-所属の揮発性-handoff-による引き継ぎ)
8. [UC-08: 単独 Job の中止（`canceled`）と Task 全体の中止](#uc-08-単独-job-の中止canceledと-task-全体の中止)
9. [UC-09: レビュー差し戻し・追加 Job の発生](#uc-09-レビュー差し戻し追加-job-の発生)
10. [UC-10: Task 完了後の整理（Ticket 肥大化対策）](#uc-10-task-完了後の整理ticket-肥大化対策)

---

## UC-01: 基本の実行フロー

### どのような場面か（シナリオ）
Project 配下に長寿命の成果目標である `Task` を作成し、単一 Worker（`wkr-01`）の単一 Session（`ssn-wkr01-01`）が担当する `Ticket` と最小実行単位の `Job` 列に分解して正式作成する。その後、`ssn-wkr01-01` が `Run` を開始し、確認可能な証拠に基づいて作業完了（`status: "completed"` / `Run.result: "succeeded"`）まで一方向に進める最も基本的なシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **正式作成と初期状態**：Task・Ticket・Job の初期 `status` はすべて `"todo"`。Ticket は直接 `worker_id`（`wkr-*`）と `session_id`（`ssn-*`）を持ち、確定後は担当セッション（`worker_id`・`session_id`）と `jobs` 配列（順序固定）を変更しない。
- **Run 開始**：対象 Ticket の `worker_id`・`session_id` と一致する Session が初回 `Run`（`run-*`）を作成し、実作業開始と同時に Job・Ticket・`todo` の親 Task を一括で `status: "doing"` に遷移させ、`Run.started` を記録する（実行中のため `result` と `ended` はまだ無い）。
- **完了（completed / succeeded）と `status` / `result` 分離**：TTJ 1.1.0 では「作業工程の完了」を Task・Ticket・Job の `status: "completed"` で表し、Job の業務判定を `Job.result`（例：`"passed"`）、実行試行の成功を `Run.result: "succeeded"`（および `ended` 記録）として明確に分離する。

### サンプル JSON パターン

#### パターン 1-A：正式作成直後（`todo`）

```json
{
  "protocol": "TTJ/1.1.0",
  "project": {
    "id": "pj-api-platform",
    "name": "API基盤開発"
  },
  "worker": {
    "id": "wkr-01",
    "name": "Worker 01"
  },
  "session": {
    "id": "ssn-wkr01-01",
    "worker_id": "wkr-01",
    "started": "2026-02-01T09:00:00Z"
  },
  "task": {
    "id": "tsk-auth-endpoint",
    "project": "pj-api-platform",
    "objective": "認証トークン検証エンドポイントを実装して公開可能にする",
    "criteria": [
      "エンドポイント実装と単体テストが完了し全件合格している"
    ],
    "status": "todo",
    "deadline": "2026-02-05T18:00:00+09:00"
  },
  "ticket": {
    "id": "tkt-impl-auth-01",
    "task": "tsk-auth-endpoint",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "トークン検証ハンドラを実装し単体テストを通す",
    "jobs": [
      "job-code-handler-01",
      "job-unit-test-01"
    ],
    "status": "todo"
  },
  "jobs": [
    {
      "id": "job-code-handler-01",
      "objective": "トークン検証ハンドラを実装する",
      "criteria": [
        "src/auth/verify.ts が作成されビルドが通る"
      ],
      "status": "todo"
    },
    {
      "id": "job-unit-test-01",
      "objective": "トークン検証ハンドラの単体テストを実行する",
      "criteria": [
        "単体テストが全件合格しテスト結果ログが保存されている"
      ],
      "status": "todo"
    }
  ]
}
```

#### パターン 1-B：最初の Job の Run 実行中（`doing`）

```json
{
  "protocol": "TTJ/1.1.0",
  "task": {
    "id": "tsk-auth-endpoint",
    "status": "doing"
  },
  "ticket": {
    "id": "tkt-impl-auth-01",
    "task": "tsk-auth-endpoint",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "jobs": [
      "job-code-handler-01",
      "job-unit-test-01"
    ],
    "status": "doing"
  },
  "jobs": [
    {
      "id": "job-code-handler-01",
      "status": "doing"
    },
    {
      "id": "job-unit-test-01",
      "status": "todo"
    }
  ],
  "run": {
    "id": "run-code-handler-01",
    "ticket_id": "tkt-impl-auth-01",
    "job": "job-code-handler-01",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "started": "2026-02-01T09:05:00Z"
  }
}
```

#### パターン 1-C：全 Job・Ticket・Task の完了確定（`completed` / `succeeded`）

```json
{
  "protocol": "TTJ/1.1.0",
  "task": {
    "id": "tsk-auth-endpoint",
    "project": "pj-api-platform",
    "objective": "認証トークン検証エンドポイントを実装して公開可能にする",
    "criteria": [
      "エンドポイント実装と単体テストが完了し全件合格している"
    ],
    "status": "completed"
  },
  "ticket": {
    "id": "tkt-impl-auth-01",
    "task": "tsk-auth-endpoint",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "トークン検証ハンドラを実装し単体テストを通す",
    "jobs": [
      "job-code-handler-01",
      "job-unit-test-01"
    ],
    "status": "completed",
    "artifacts": [
      "src/auth/verify.ts",
      "logs/unit-test-auth-01.log"
    ]
  },
  "jobs": [
    {
      "id": "job-code-handler-01",
      "objective": "トークン検証ハンドラを実装する",
      "criteria": [
        "src/auth/verify.ts が作成されビルドが通る"
      ],
      "status": "completed",
      "result": "passed",
      "artifacts": [
        "src/auth/verify.ts"
      ]
    },
    {
      "id": "job-unit-test-01",
      "objective": "トークン検証ハンドラの単体テストを実行する",
      "criteria": [
        "単体テストが全件合格しテスト結果ログが保存されている"
      ],
      "status": "completed",
      "result": "passed",
      "artifacts": [
        "logs/unit-test-auth-01.log"
      ]
    }
  ],
  "runs": [
    {
      "id": "run-code-handler-01",
      "ticket_id": "tkt-impl-auth-01",
      "job": "job-code-handler-01",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T09:05:00Z",
      "ended": "2026-02-01T09:20:00Z",
      "result": "succeeded"
    },
    {
      "id": "run-unit-test-01",
      "ticket_id": "tkt-impl-auth-01",
      "job": "job-unit-test-01",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T09:21:00Z",
      "ended": "2026-02-01T09:30:00Z",
      "result": "succeeded"
    }
  ]
}
```

---

## UC-02: 複数 Worker による並列実行

### どのような場面か（シナリオ）
同一の `Task` 配下に、互いに実行依存を持たない独立した複数の `Ticket`（バックエンド実装 `tkt-backend-01` とフロントエンド実装 `tkt-frontend-02`）を作成し、`wkr-01`（`ssn-wkr01-01`）と `wkr-02`（`ssn-wkr02-01`）がそれぞれの Ticket で並行して `Run` を同時実行するシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **Ticket ごとの担当セッション固定と分割**：1つの Ticket に複数 Worker や複数 Session を設定することは禁止されているため、複数 Worker（`wkr-01`, `wkr-02`）が分担する場合は Worker / Session ごとに別 Ticket を作成し、それぞれ不変の `worker_id` と `session_id` を設定する。
- **並列実行の非阻害**：一方の Ticket が `waiting` や `blocked` になったり、Task 所属の単一スロット `Handoff` が一方の Ticket 文脈を表示していたりしても、無関係な独立 Ticket の開始・並行実行を遮ったり直列化したりしてはならない（[規格本文](specification.md) §6・§8.2・§9.1・禁止事項 #16）。

### サンプル JSON パターン

```json
{
  "protocol": "TTJ/1.1.0",
  "workers": [
    {
      "id": "wkr-01",
      "name": "Worker 01"
    },
    {
      "id": "wkr-02",
      "name": "Worker 02"
    }
  ],
  "sessions": [
    {
      "id": "ssn-wkr01-01",
      "worker_id": "wkr-01",
      "started": "2026-02-01T10:00:00Z"
    },
    {
      "id": "ssn-wkr02-01",
      "worker_id": "wkr-02",
      "started": "2026-02-01T10:00:00Z"
    }
  ],
  "task": {
    "id": "tsk-search-feature",
    "project": "pj-api-platform",
    "objective": "検索APIと検索UIを実装する",
    "criteria": [
      "検索APIエンドポイントと検索画面コンポーネントの双方がテストを通過している"
    ],
    "status": "doing"
  },
  "tickets": [
    {
      "id": "tkt-backend-01",
      "task": "tsk-search-feature",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "objective": "検索APIエンドポイントを実装する",
      "jobs": [
        "job-impl-search-api"
      ],
      "depends": [],
      "status": "doing"
    },
    {
      "id": "tkt-frontend-02",
      "task": "tsk-search-feature",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "objective": "検索UIコンポーネントを実装する",
      "jobs": [
        "job-impl-search-ui"
      ],
      "depends": [],
      "status": "doing"
    }
  ],
  "runs": [
    {
      "id": "run-search-api-01",
      "ticket_id": "tkt-backend-01",
      "job": "job-impl-search-api",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T10:01:00Z"
    },
    {
      "id": "run-search-ui-01",
      "ticket_id": "tkt-frontend-02",
      "job": "job-impl-search-ui",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "started": "2026-02-01T10:01:30Z"
    }
  ]
}
```

---

## UC-03: Ticket 間の依存関係（`depends_on` / `depends`）とブロック解除

### どのような場面か（シナリオ）
後続 Ticket が先行 Ticket の完了を前提としている場面。先行 Ticket が未完了（あるいは先行 Ticket が中止となり継続不能な `blocked` 状態）のため後続 Ticket が開始を阻まれている状態から、先行 Ticket（または代替 Ticket）の `completed` 確定によってブロックが解除され、後続 Ticket が実行可能な `"todo"` へ遷移するシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **実行依存フィールド名 `depends`**：実行依存は `Ticket.depends`（前提 Ticket ID の配列 `["tkt-..."]`）として記述する。情報共有専用の `refs` とは明確に区別し、循環依存は厳禁とする。
- **先行 Ticket 未完了時の状態（計画内待機 `waiting` と計画外障害 `blocked`）**：
  1. **通常の先行 Ticket 完了待ち**：先行 Ticket が進行中（`todo` / `doing` 等）の間、後続 Ticket は計画内の前提待機として `status: "waiting"` とし、先行 Ticket が `status: "completed"` になった時点で依存評価操作により `waiting → todo` へ解除される。
  2. **先行 Ticket の中止や成果物欠落による `blocked` とその解除**：先行 Ticket（例：`tkt-schema-01`）が `status: "canceled"` になった場合など、現行計画のままでは継続不能になったとき、後続 Ticket（`tkt-migration-02`）は `status: "blocked"` に遷移し `resume` に解除条件を永続記録する。その後、代替の先行 Ticket（`tkt-schema-02`）が完了し、未着手の後続 Ticket の `depends` 更新と課題解消が確認された時点で `blocked → todo` へ遷移してブロック解除される（[規格本文](specification.md) §5.2・§7）。

### サンプル JSON パターン

#### パターン 3-A：先行 Ticket 中止により後続 Ticket が `blocked` で待機中

```json
{
  "protocol": "TTJ/1.1.0",
  "tickets": [
    {
      "id": "tkt-schema-01",
      "task": "tsk-db-upgrade",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "objective": "新テーブルのスキーマ定義を作成する",
      "jobs": [
        "job-draft-schema-01"
      ],
      "status": "canceled",
      "cancel_reason": "旧仕様のカラム設計に不整合が判明したため中止。tkt-schema-02 で再作成する。"
    },
    {
      "id": "tkt-schema-02",
      "task": "tsk-db-upgrade",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-02",
      "objective": "修正仕様に基づきスキーマ定義を確定する",
      "jobs": [
        "job-finalize-schema-02"
      ],
      "refs": [
        {
          "ticket_id": "tkt-schema-01",
          "kind": "rerun_of",
          "note": "カラム型修正後の再作成"
        }
      ],
      "status": "doing"
    },
    {
      "id": "tkt-migration-02",
      "task": "tsk-db-upgrade",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "objective": "確定スキーマを用いてマイグレーションを実行する",
      "jobs": [
        "job-run-migration-01"
      ],
      "depends": [
        "tkt-schema-01"
      ],
      "status": "blocked",
      "resume": "代替の先行Ticket tkt-schema-02 が completed になり depends を更新してスキーマ成果物の存在を確認する"
    }
  ]
}
```

#### パターン 3-B：先行 Ticket の完了（`completed`）に伴い後続 Ticket が `"todo"` へブロック解除された状態

```json
{
  "protocol": "TTJ/1.1.0",
  "tickets": [
    {
      "id": "tkt-schema-02",
      "task": "tsk-db-upgrade",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-02",
      "objective": "修正仕様に基づきスキーマ定義を確定する",
      "jobs": [
        "job-finalize-schema-02"
      ],
      "status": "completed",
      "artifacts": [
        "db/schema-v2.sql"
      ]
    },
    {
      "id": "tkt-migration-02",
      "task": "tsk-db-upgrade",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "objective": "確定スキーマを用いてマイグレーションを実行する",
      "jobs": [
        "job-run-migration-01"
      ],
      "depends": [
        "tkt-schema-02"
      ],
      "refs": [
        {
          "ticket_id": "tkt-schema-01",
          "kind": "related",
          "note": "旧前提 tkt-schema-01 の中止に伴い前提を tkt-schema-02 に更新済み"
        }
      ],
      "status": "todo"
    }
  ]
}
```

---

## UC-04: 外部待機（`waiting`）と一時停止（`paused`）の使い分け

### どのような場面か（シナリオ）
Job 実行中に外部システムの応答や承認を待つ計画内待機（`waiting`）と、オーナー（または運用責任者）の明示指示によって作業を一時保留する（`paused`）シナリオ。さらに、`waiting` 中にオーナー指示で `paused` へ移行し、その後 `paused` を解除した際にどのように状態を再評価するかを示す。

### 規格上のポイント（どの状態・フィールドを使うか）
- **`waiting` と `Run.wait`**：計画に含まれる外部応答・承認・イベント待ちは `status: "waiting"` とする。Job 実行に伴う待機制御情報（`job_id`, `target`, `condition`, `since`, `resume_point`, `timeout`, 解消時 `resolved`）は Job ではなく **`Run.wait`** に保持する（Run を持たない Task 単位の計画内待機や、`Run.wait` だけでは再開条件が分からない場合は `resume` に記載する）。待機対象や Worker 種別で状態遷移を分岐させてはならない。
- **`paused`（オーナー指示等による明示的な保留）**：運用判断やオーナー指示で一時停止する場合は `status: "paused"` とし、明示解除されるまで一切の実行再開を禁止する。
- **`paused` 解除時の再評価順序**：`waiting`（または `blocked`）の途中で `paused` にした場合も、未解消の `Run.wait` や課題はそのまま保持する。オーナー指示で `paused` を明示解除した際は、**保留前の状態を無条件に復元せず**、残存する `blocked`・`waiting`・`depends` を先に再評価する。外部応答がまだ未到着なら `paused → waiting` に戻り、待機条件がすべて解消済み（`Run.wait.resolved` 記録済み）なら `paused → doing`（未着手なら `todo`）へ復帰する。

### サンプル JSON パターン

#### パターン 4-A：外部応答待ちの `waiting`（`Run.wait` による待機制御）

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-release-prod-01",
    "task": "tsk-release-v110",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "本番環境へのデプロイを実施する",
    "jobs": [
      "job-apply-prod-deploy"
    ],
    "status": "waiting"
  },
  "job": {
    "id": "job-apply-prod-deploy",
    "objective": "本番反映スクリプトを実行しヘルスチェックを確認する",
    "criteria": [
      "承認ゲート通過後に本番反映が完了しヘルスチェック200 OKが記録されている"
    ],
    "status": "waiting"
  },
  "run": {
    "id": "run-prod-deploy-01",
    "ticket_id": "tkt-release-prod-01",
    "job": "job-apply-prod-deploy",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "started": "2026-02-01T12:00:00Z",
    "wait": {
      "job_id": "job-apply-prod-deploy",
      "target": "change-gate:chg-2026-0201",
      "condition": "変更管理システムで本番反映ウィンドウの承認通知を受領する",
      "since": "2026-02-01T12:03:00Z",
      "resume_point": "step-2-rollout-service",
      "timeout": "2026-02-01T15:00:00Z"
    }
  }
}
```

#### パターン 4-B：オーナー指示による一時停止（`paused`）と未解消 `Run.wait` の保持

```json
{
  "protocol": "TTJ/1.1.0",
  "task": {
    "id": "tsk-release-v110",
    "project": "pj-api-platform",
    "objective": "v1.1.0 を本番環境へリリースする",
    "criteria": [
      "本番デプロイと疎通確認が完了している"
    ],
    "status": "paused",
    "resume": "オーナーからリリース凍結解除の指示を受け、かつ変更管理ゲート chg-2026-0201 の承認を確認する"
  },
  "ticket": {
    "id": "tkt-release-prod-01",
    "task": "tsk-release-v110",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "本番環境へのデプロイを実施する",
    "jobs": [
      "job-apply-prod-deploy"
    ],
    "status": "paused",
    "resume": "オーナーの保留解除指示後、未解決の承認ゲート待機を再評価する"
  },
  "job": {
    "id": "job-apply-prod-deploy",
    "objective": "本番反映スクリプトを実行しヘルスチェックを確認する",
    "criteria": [
      "承認ゲート通過後に本番反映が完了しヘルスチェック200 OKが記録されている"
    ],
    "status": "paused",
    "resume": "保留解除および change-gate:chg-2026-0201 の承認成立を確認する"
  },
  "run": {
    "id": "run-prod-deploy-01",
    "ticket_id": "tkt-release-prod-01",
    "job": "job-apply-prod-deploy",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "started": "2026-02-01T12:00:00Z",
    "ended": "2026-02-01T12:10:00Z",
    "result": "interrupted",
    "wait": {
      "job_id": "job-apply-prod-deploy",
      "target": "change-gate:chg-2026-0201",
      "condition": "変更管理システムで本番反映ウィンドウの承認通知を受領する",
      "since": "2026-02-01T12:03:00Z",
      "resume_point": "step-2-rollout-service",
      "timeout": "2026-02-01T15:00:00Z"
    }
  }
}
```

---

## UC-05: Job の失敗（`failed`）と同一 Ticket・同一 Session 内での再試行

### どのような場面か（シナリオ）
Job の 1 回目の実行試行（`run-integration-01`）が一時的な通信エラーやタイムアウトで失敗（`Run.result: "failed"`）したものの、同じ作業計画（同一 Ticket・同一 Session・同一 Job）のまま回復可能であるため、失敗した旧 Run を閉じて新しい Run（`run-integration-02`）を追加・実行するシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **回復可能な中断と Job `status: "doing"` の維持**：Run が 1 回失敗・終了しただけで即座に Job や Ticket を `completed`・`canceled`・`blocked` にしてはならない（[規格本文](specification.md) §8.2）。同一計画・同一 Session で再試行できる間は Job・Ticket ともに `status: "doing"` を維持する。
- **旧 Run の閉鎖と新 Run の追加**：旧 Run（`run-integration-01`）の終了確定時に `result: "failed"`（および `ended`）を記録して閉じ、前の実行が停止（または残存副作用がない安全条件）していることを確認した上で、同一 Ticket（`tkt-verify-api-01`）・同一 Session（`ssn-wkr01-01`）で新しい `Run`（`run-integration-02`）を追加して再試行する。

### サンプル JSON パターン

#### パターン 5-A：1回目の Run（`run-integration-01`）が `failed` で閉鎖された直後

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-verify-api-01",
    "task": "tsk-auth-endpoint",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "ステージング環境で結合検証を実行する",
    "jobs": [
      "job-run-integration"
    ],
    "status": "doing"
  },
  "job": {
    "id": "job-run-integration",
    "objective": "結合テストスイートを実行し全件成功を確認する",
    "criteria": [
      "結合テスト全12ケースが成功し結果レポートが保存されている"
    ],
    "status": "doing",
    "resume": "一時的な接続タイムアウト解消を確認して同一Jobを新Runで再試行する"
  },
  "run": {
    "id": "run-integration-01",
    "ticket_id": "tkt-verify-api-01",
    "job": "job-run-integration",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "started": "2026-02-01T13:00:00Z",
    "ended": "2026-02-01T13:04:00Z",
    "result": "failed"
  }
}
```

#### パターン 5-B：同一 Ticket・同一 Session 内で新 Run（`run-integration-02`）を追加して再試行中

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-verify-api-01",
    "task": "tsk-auth-endpoint",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "ステージング環境で結合検証を実行する",
    "jobs": [
      "job-run-integration"
    ],
    "status": "doing"
  },
  "runs": [
    {
      "id": "run-integration-01",
      "ticket_id": "tkt-verify-api-01",
      "job": "job-run-integration",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T13:00:00Z",
      "ended": "2026-02-01T13:04:00Z",
      "result": "failed"
    },
    {
      "id": "run-integration-02",
      "ticket_id": "tkt-verify-api-01",
      "job": "job-run-integration",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T13:06:00Z"
    }
  ]
}
```

---

## UC-06: Worker / Session 再開不可時の旧 Ticket 中止と新規 Ticket への引き継ぎ

### どのような場面か（シナリオ）
作業中の Worker / Session が異常停止し再開不可となったため、旧 Ticket（および未完了 Job / Run）を中止・閉鎖し、同一 Task 配下に新規 Ticket（`refs` で旧 Ticket を参照）を起票して新しい Session で稼働させるシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **1つの Ticket の担当セッション不変の原則**：1つの Ticket の持ち主（`worker_id`・`session_id`）は途中で変わらない。担当 Session が再開不可になった場合は、同じ Ticket の持ち主を付け替えるのではなく、旧 Ticket を閉じて新規 Ticket として作り直し、新しい Session で稼働させる。
- **引き継ぎ前の停止・安全確認**：前の実行が稼働中または停止未確認の間は、後続 Ticket による引き継ぎ実行を開始してはならない（禁止事項 #13, #14）。前の実行が停止（または残存副作用がない安全条件）していることを確認できた場合のみ引き継ぎが許可される。
- **旧 Ticket の閉鎖と新規 Ticket の起票（別 Worker でも同一 Worker の新 Session でも同一方式）**：
  - 前の実行が停止（または残存副作用がない安全条件）していることを確認した上で、旧 Run を終了（`result: "canceled"` または `"interrupted"` 等、確認できれば `ended` を記録）し、旧 Ticket `tkt-deploy-01`（`worker_id: "wkr-01"`, `session_id: "ssn-wkr01-01"`）を `status: "canceled"`（`cancel_reason` 必須、未完了 Job も `canceled`）として閉鎖する。
  - 同一 Task 配下に新規 Ticket `tkt-deploy-02`（`refs` に `tkt-deploy-01` を指定）を起票し、新しい担当 Session（別 Worker `wkr-02` の `ssn-wkr02-01`、または同一 Worker `wkr-01` の新 Session `ssn-wkr01-02`）を設定して新しい `Run` を開始する。

### サンプル JSON パターン

#### パターン 6-A：別 Worker（`wkr-02` / `ssn-wkr02-01`）が新規 Ticket（`tkt-deploy-02`）で引き継ぐ場合

```json
{
  "protocol": "TTJ/1.1.0",
  "tickets": [
    {
      "id": "tkt-deploy-01",
      "task": "tsk-release-v110",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "objective": "ステージング環境へパッケージを配布し疎通確認を行う",
      "jobs": [
        "job-upload-pkg-01",
        "job-smoke-test-01"
      ],
      "status": "canceled",
      "cancel_reason": "ssn-wkr01-01 の停止および残存副作用がない安全条件を確認した上で旧Ticketを中止し、新規Ticket tkt-deploy-02 (wkr-02 / ssn-wkr02-01) へ引き継ぐ。",
      "artifacts": [
        "packages/app-v110.tar.gz"
      ]
    },
    {
      "id": "tkt-deploy-02",
      "task": "tsk-release-v110",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "objective": "配布済みパッケージを用いてステージング疎通確認を完了する",
      "jobs": [
        "job-smoke-test-02"
      ],
      "refs": [
        {
          "ticket_id": "tkt-deploy-01",
          "kind": "derived_from",
          "note": "job-upload-pkg-01 の成果物 packages/app-v110.tar.gz を引き継いで残工程を実行"
        }
      ],
      "status": "doing"
    }
  ],
  "jobs": [
    {
      "id": "job-upload-pkg-01",
      "objective": "ステージングへパッケージをアップロードする",
      "criteria": [
        "packages/app-v110.tar.gz のチェックサム一致が確認されている"
      ],
      "status": "completed",
      "result": "passed"
    },
    {
      "id": "job-smoke-test-01",
      "objective": "ステージングの疎通テストを実行する",
      "criteria": [
        "疎通テスト全項目が成功している"
      ],
      "status": "canceled",
      "result": "interrupted"
    },
    {
      "id": "job-smoke-test-02",
      "objective": "ステージングの疎通テストを実行する",
      "criteria": [
        "疎通テスト全項目が成功している"
      ],
      "status": "doing"
    }
  ],
  "runs": [
    {
      "id": "run-smoke-test-01",
      "ticket_id": "tkt-deploy-01",
      "job": "job-smoke-test-01",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T14:00:00Z",
      "ended": "2026-02-01T14:15:00Z",
      "result": "interrupted"
    },
    {
      "id": "run-smoke-test-02",
      "ticket_id": "tkt-deploy-02",
      "job": "job-smoke-test-02",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "started": "2026-02-01T14:21:00Z"
    }
  ]
}
```

#### パターン 6-B：同一 Worker（`wkr-01`）が新 Session（`ssn-wkr01-02`）で新規 Ticket（`tkt-deploy-02`）を起票して引き継ぐ場合

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-deploy-02",
    "task": "tsk-release-v110",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-02",
    "objective": "配布済みパッケージを用いてステージング疎通確認を完了する",
    "jobs": [
      "job-smoke-test-02"
    ],
    "refs": [
      {
        "ticket_id": "tkt-deploy-01",
        "kind": "derived_from",
        "note": "旧Session ssn-wkr01-01 の停止確認後に旧Ticket tkt-deploy-01 を閉じ、新Session ssn-wkr01-02 で残工程を引き継ぐ"
      }
    ],
    "status": "doing"
  },
  "run": {
    "id": "run-smoke-test-02",
    "ticket_id": "tkt-deploy-02",
    "job": "job-smoke-test-02",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-02",
    "started": "2026-02-01T14:21:00Z"
  }
}
```

---

## UC-07: Task 所属の揮発性 Handoff による引き継ぎ

### どのような場面か（シナリオ）
作業途中の文脈・停止位置・注意点を Task 所属の揮発性単一スロット `Handoff` に記録して後続 Session や他メンバーが状況を把握できるようにし、Task 完了時には揮発性 Handoff を破棄・消去して、今後再利用すべき長期知見だけを `Project.knowledge` へ昇格させるシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **Task 所属の揮発性単一スロット**：v1.1.0 では旧 `Ticket.handoffs` を廃止し、`Task` ごとに最大1つの揮発性 `Handoff` スロットを持つ。
- **閲覧権限と実行権限の分離**：Handoff は Task 参照権限を持つ複数 Session から閲覧できるが、Handoff を閲覧できることは Ticket の実行権限を意味しない（禁止事項 #11）。
- **正本保存の原則と Task 完了時の揮発・消去**：再開に必須の条件（`resume`, `Run.wait`, `cancel_reason`, `artifacts`）は必ず永続正本側に保存し、Handoff を正式履歴の正本にしてはならない（禁止事項 #15）。Task 完了時に Handoff スロットを消去しても正式履歴は一切失われず、再利用する恒久知見のみを選別して `Project.knowledge`（`status: "active"`）へ記録する。

### サンプル JSON パターン

#### パターン 7-A：作業進行中の Task 揮発性 `Handoff` スロットと永続正本

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-tune-cache-01",
    "task": "tsk-latency-improvement",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "キャッシュTTL設定を調整してP99レイテンシを改善する",
    "jobs": [
      "job-update-cache-config",
      "job-run-benchmark"
    ],
    "status": "doing",
    "artifacts": [
      "configs/cache-ttl-v2.yaml"
    ]
  },
  "handoff": {
    "task_id": "tsk-latency-improvement",
    "current_ticket_id": "tkt-tune-cache-01",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "current_job_id": "job-run-benchmark",
    "context": "TTLを300秒に変更した cache-ttl-v2.yaml を反映済み。現在ベンチマーク計測を実行中。",
    "stop_position": "job-run-benchmark のウォームアップ完了後・本計測ステップ",
    "blockers": [],
    "notes": [
      "TTL 600秒ではメモリ使用率が閾値を超えたため300秒を採用した。"
    ],
    "updated": "2026-02-01T15:10:00Z"
  }
}
```

#### パターン 7-B：Task 完了後（揮発性 Handoff 消去後）の永続正本と `Project.knowledge` への知見昇格

```json
{
  "protocol": "TTJ/1.1.0",
  "task": {
    "id": "tsk-latency-improvement",
    "project": "pj-api-platform",
    "objective": "認証・検索APIのP99レイテンシを50ms以下に改善する",
    "criteria": [
      "ベンチマーク計測でP99レイテンシが50ms以下であることが確認されている"
    ],
    "status": "completed"
  },
  "ticket": {
    "id": "tkt-tune-cache-01",
    "task": "tsk-latency-improvement",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "キャッシュTTL設定を調整してP99レイテンシを改善する",
    "jobs": [
      "job-update-cache-config",
      "job-run-benchmark"
    ],
    "status": "completed",
    "artifacts": [
      "configs/cache-ttl-v2.yaml",
      "reports/benchmark-20260201.json"
    ]
  },
  "knowledge": [
    {
      "id": "cache-ttl-guideline",
      "status": "active",
      "content": "APIレスポンスキャッシュの標準TTLは300秒とし、メモリ使用率80%未満を維持する",
      "basis": "tsk-latency-improvement のベンチマーク検証において、TTL 600秒ではメモリ逼迫が発生し、300秒でP99 38msとメモリ安全域を両立できたため。"
    }
  ]
}
```

---

## UC-08: 単独 Job の中止（`canceled`）と Task 全体の中止

### どのような場面か（シナリオ）
Ticket 内の特定の不要になった Job だけを `canceled` にして実行中の Run を閉鎖しつつ同一 Ticket・同一 Session で後続 Job を続行するケースと、方針変更等により Task 全体（および配下の未終端 Ticket・Job）を一括で `canceled` にするケースの対比シナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **単独 Job の中止（後続 Job を続ける場合）**：
  - 対象 Job を `status: "canceled"` に確定すると同時に、実行中の Run があれば終了（`Run.result: "canceled"` 等を記録して閉鎖）し、停止要求を発行する。
  - 後続 Job の新 Run を開始する前に、前の Job の実行が停止（または残存副作用がない安全条件）していることを確認する。全 Job が `completed` または `canceled` になれば Ticket は `status: "completed"` に集約される（ただし中止した Job の目標を達成したとはみなさない）。
- **Task 全体（または Ticket 全体）の中止**：
  - 親 Task を `status: "canceled"` にすると、配下の**未終端の Ticket・Job もすべて同時に `status: "canceled"`** に確定し、実行中の Run も閉鎖する（既に `completed` になっていた終端済みの子孫は変更せず保持する）。
  - 中止された各 Ticket には **`cancel_reason` の記録が必須**。Ticket 自身の中止は全 Job 終端による `completed` 集約より常に優先される。

### サンプル JSON パターン

#### パターン 8-A：単独 Job（`job-optional-seed`）のみを中止し、同一 Ticket・同一 Session で後続 Job（`job-verify-env`）を完了させたパターン

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-setup-preview-01",
    "task": "tsk-preview-env",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-01",
    "objective": "プレビュー環境を構築し疎通を確認する",
    "jobs": [
      "job-provision-infra",
      "job-optional-seed",
      "job-verify-env"
    ],
    "status": "completed"
  },
  "jobs": [
    {
      "id": "job-provision-infra",
      "objective": "プレビュー用コンテナを起動する",
      "criteria": [
        "コンテナがRunning状態である"
      ],
      "status": "completed",
      "result": "passed"
    },
    {
      "id": "job-optional-seed",
      "objective": "旧形式のデモ用シードデータを投入する",
      "criteria": [
        "シードデータが投入されている"
      ],
      "status": "canceled",
      "result": "canceled"
    },
    {
      "id": "job-verify-env",
      "objective": "プレビュー環境のヘルスチェックを実行する",
      "criteria": [
        "/healthz が 200 OK を返す"
      ],
      "status": "completed",
      "result": "passed"
    }
  ],
  "runs": [
    {
      "id": "run-optional-seed-01",
      "ticket_id": "tkt-setup-preview-01",
      "job": "job-optional-seed",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T16:10:00Z",
      "ended": "2026-02-01T16:11:00Z",
      "result": "canceled"
    },
    {
      "id": "run-verify-env-01",
      "ticket_id": "tkt-setup-preview-01",
      "job": "job-verify-env",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "started": "2026-02-01T16:12:00Z",
      "ended": "2026-02-01T16:15:00Z",
      "result": "succeeded"
    }
  ]
}
```

#### パターン 8-B：Task 全体を中止（`canceled`）し、未終端の子孫中止を同時確定したパターン

```json
{
  "protocol": "TTJ/1.1.0",
  "task": {
    "id": "tsk-legacy-export",
    "project": "pj-api-platform",
    "objective": "旧CSVエクスポート機能を追加する",
    "criteria": [
      "CSV出力APIとバッチ処理が稼働している"
    ],
    "status": "canceled"
  },
  "tickets": [
    {
      "id": "tkt-csv-spec-01",
      "task": "tsk-legacy-export",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "objective": "CSVフォーマット仕様を策定する",
      "jobs": [
        "job-define-csv-format"
      ],
      "status": "completed",
      "artifacts": [
        "docs/csv-spec.md"
      ]
    },
    {
      "id": "tkt-csv-impl-02",
      "task": "tsk-legacy-export",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "objective": "CSV出力バッチを実装する",
      "jobs": [
        "job-code-csv-batch",
        "job-test-csv-batch"
      ],
      "status": "canceled",
      "cancel_reason": "製品ロードマップ変更により旧CSVエクスポート機能の開発自体が中止（tsk-legacy-export canceled）となったため。"
    }
  ],
  "jobs": [
    {
      "id": "job-define-csv-format",
      "objective": "CSVフォーマット仕様書を作成する",
      "criteria": [
        "docs/csv-spec.md が作成されている"
      ],
      "status": "completed",
      "result": "passed"
    },
    {
      "id": "job-code-csv-batch",
      "objective": "CSV出力バッチを実装する",
      "criteria": [
        "バッチ処理コードがビルドを通過する"
      ],
      "status": "canceled",
      "result": "canceled"
    },
    {
      "id": "job-test-csv-batch",
      "objective": "CSV出力バッチを検証する",
      "criteria": [
        "検証テストが全件成功する"
      ],
      "status": "canceled"
    }
  ]
}
```

---

## UC-09: レビュー差し戻し・追加 Job の発生

### どのような場面か（シナリオ）
作成 Ticket（`tkt-draft-doc-01`）の完了後に別 Worker（`wkr-02`）がレビュー Ticket（`tkt-review-doc-01`）を実行した結果、修正要求（`changes_requested`）となり、指摘反映と再検証のための追加 Job と新 Run が必要になったシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **レビュー差し戻し時の `status` と `result` の分離**：レビュー Job が差し戻し結論を出した場合、レビューという作業自体は完了しているため `Job.status: "completed"`, `Job.result: "changes_requested"` として確定し、レビュー Ticket も `status: "completed"` に集約する（独立した `Request` オブジェクトは導入しない）。
- **確定後 Ticket への Job 追加禁止・終端 Ticket 再開禁止**：一度正式作成した Ticket の `jobs` 列に後から Job を追加したり、完了済みの作成 Ticket やレビュー Ticket を `doing` / `todo` に再開・巻き戻したりすることは**禁止**（禁止事項 #8, #9）。
- **同一 Task 配下への新 Ticket 起票による追加 Job・新 Run 対応**：差し戻し修正や追加で必要になった Job 列は、**同一 Task（`tsk-*`）配下に新しい Ticket（`tkt-revise-doc-02`）として起票**する（Task を新しく増殖させてはならない）。新 Ticket の `depends` にレビュー Ticket を指定し、`refs`（`kind: "fixes"` / `"derived_from"`）で初稿 Ticket の成果物とレビュー Ticket の指摘メモを参照した上で、新 Session（`ssn-wkr01-02`）で新 Run を実行する。

### サンプル JSON パターン

```json
{
  "protocol": "TTJ/1.1.0",
  "task": {
    "id": "tsk-security-policy",
    "project": "pj-api-platform",
    "objective": "APIセキュリティ運用規約を策定し承認を得る",
    "criteria": [
      "規約文書が作成され、レビュアーの承認（approved）を得ている"
    ],
    "status": "doing"
  },
  "tickets": [
    {
      "id": "tkt-draft-doc-01",
      "task": "tsk-security-policy",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "objective": "セキュリティ運用規約の初稿を作成する",
      "jobs": [
        "job-write-policy-01"
      ],
      "status": "completed",
      "artifacts": [
        "docs/security-policy-v1.md"
      ]
    },
    {
      "id": "tkt-review-doc-01",
      "task": "tsk-security-policy",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "objective": "セキュリティ運用規約の初稿をレビューする",
      "jobs": [
        "job-review-policy-01"
      ],
      "depends": [
        "tkt-draft-doc-01"
      ],
      "refs": [
        {
          "ticket_id": "tkt-draft-doc-01",
          "kind": "related",
          "note": "初稿 docs/security-policy-v1.md をレビュー"
        }
      ],
      "status": "completed",
      "artifacts": [
        "reviews/security-policy-review-01.md"
      ]
    },
    {
      "id": "tkt-revise-doc-02",
      "task": "tsk-security-policy",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-02",
      "objective": "レビュー指摘に基づき鍵ローテーション手順を追記・検証する",
      "jobs": [
        "job-add-rotation-section",
        "job-verify-policy-checklist"
      ],
      "depends": [
        "tkt-review-doc-01"
      ],
      "refs": [
        {
          "ticket_id": "tkt-draft-doc-01",
          "kind": "fixes",
          "note": "初稿 docs/security-policy-v1.md をベースに改訂"
        },
        {
          "ticket_id": "tkt-review-doc-01",
          "kind": "derived_from",
          "note": "changes_requested の指摘事項（鍵ローテーション手順の不足）を反映"
        }
      ],
      "status": "doing"
    }
  ],
  "jobs": [
    {
      "id": "job-review-policy-01",
      "objective": "初稿の記載漏れと基準適合性を検証し判定を記録する",
      "criteria": [
        "レビュー結果と判定（approved または changes_requested）が記録されている"
      ],
      "status": "completed",
      "result": "changes_requested",
      "artifacts": [
        "reviews/security-policy-review-01.md"
      ]
    },
    {
      "id": "job-add-rotation-section",
      "objective": "緊急時の鍵ローテーション手順節を追記する",
      "criteria": [
        "docs/security-policy-v2.md に鍵ローテーション手順が明記されている"
      ],
      "status": "doing"
    },
    {
      "id": "job-verify-policy-checklist",
      "objective": "全チェックリスト項目との整合を確認する",
      "criteria": [
        "チェックリスト全項目との照合が完了している"
      ],
      "status": "todo"
    }
  ],
  "run": {
    "id": "run-add-rotation-01",
    "ticket_id": "tkt-revise-doc-02",
    "job": "job-add-rotation-section",
    "worker_id": "wkr-01",
    "session_id": "ssn-wkr01-02",
    "started": "2026-02-01T17:00:00Z"
  }
}
```

---

## UC-10: Task 完了後の整理（Ticket 肥大化対策）

### どのような場面か（シナリオ）
作成・レビュー差し戻し・修正・再レビューを経て複数の Ticket が作られた長寿命 `Task` が目標を達成して完了する際、終了済み旧 Ticket のアーカイブ・折りたたみ、揮発性 Handoff の消去、および Task 完了判定を行ってクリーンな終了状態にするシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **終了済み旧 Ticket の即時アーカイブ・折りたたみ**：各 Ticket は終了（`completed` / `canceled`）時点で成果（`artifacts`）・結果（`result`）・中止理由（`cancel_reason`）を永続化する。次 Ticket が未作成でも終了済み Ticket は即座にアーカイブ・折りたたみ可能とする（[規格本文](specification.md) §4・§6）。
- **Ticket 肥大化対策と Task 完了判定**：
  - `refs`（`fixes`, `derived_from`, `rerun_of`, `related`）によって「初稿 → 差し戻しレビュー → 修正 → 承認レビュー」の系列を構造化し、終了済み旧 Ticket を折りたたんで管理負荷を抑える。
  - **完了 Ticket 数だけで Task の達成率や完了を決めてはならない**（[規格本文](specification.md) §2・§6）。全 Ticket が終端状態であり、差し戻しや中止で未達だった必須範囲が後続 Ticket で解決され、`Task.criteria` をすべて満たしていることを確認して `Task.status: "completed"` とする。
- **揮発性 Handoff の消去と長期知識の分離**：Task 完了時には全 Run が終了確定済みとなり、Task 所属の揮発性 `Handoff` スロットは長期記憶として残さず破棄・消去する。将来再利用する恒久知見のみを `Project.knowledge`（`status: "active"`、旧知識の置換時は旧記録を `"supersed"` にして `supersedes` で参照）に昇格させて保持する。

### サンプル JSON パターン

```json
{
  "protocol": "TTJ/1.1.0",
  "project": {
    "id": "pj-api-platform",
    "name": "API基盤開発",
    "knowledge": [
      {
        "id": "security-policy-v0-rule",
        "status": "supersed",
        "content": "署名鍵のローテーションは日次バッチで一括反映する",
        "basis": "初期設計時の暫定運用ルールとして採用していた。"
      },
      {
        "id": "security-policy-immediate-key-rotation",
        "status": "active",
        "content": "緊急時の署名鍵ローテーションは即時配信リストへ反映し5分以内に全ゲートウェイへ伝播させる",
        "basis": "tsk-security-policy のレビューおよび改訂で合意された新基準。",
        "supersedes": "security-policy-v0-rule"
      }
    ]
  },
  "task": {
    "id": "tsk-security-policy",
    "project": "pj-api-platform",
    "objective": "APIセキュリティ運用規約を策定し承認を得る",
    "criteria": [
      "規約文書が作成され、レビュアーの承認（approved）を得ている"
    ],
    "status": "completed"
  },
  "tickets": [
    {
      "id": "tkt-draft-doc-01",
      "task": "tsk-security-policy",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-01",
      "objective": "セキュリティ運用規約の初稿を作成する",
      "jobs": [
        "job-write-policy-01"
      ],
      "status": "completed",
      "artifacts": [
        "docs/security-policy-v1.md"
      ]
    },
    {
      "id": "tkt-review-doc-01",
      "task": "tsk-security-policy",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-01",
      "objective": "セキュリティ運用規約の初稿をレビューする",
      "jobs": [
        "job-review-policy-01"
      ],
      "depends": [
        "tkt-draft-doc-01"
      ],
      "refs": [
        {
          "ticket_id": "tkt-draft-doc-01",
          "kind": "related",
          "note": "初稿レビュー（changes_requested で終了・アーカイブ済み）"
        }
      ],
      "status": "completed",
      "artifacts": [
        "reviews/security-policy-review-01.md"
      ]
    },
    {
      "id": "tkt-revise-doc-02",
      "task": "tsk-security-policy",
      "worker_id": "wkr-01",
      "session_id": "ssn-wkr01-02",
      "objective": "レビュー指摘に基づき鍵ローテーション手順を追記・検証する",
      "jobs": [
        "job-add-rotation-section",
        "job-verify-policy-checklist"
      ],
      "depends": [
        "tkt-review-doc-01"
      ],
      "refs": [
        {
          "ticket_id": "tkt-draft-doc-01",
          "kind": "fixes",
          "note": "初稿を改訂し v2 を作成"
        },
        {
          "ticket_id": "tkt-review-doc-01",
          "kind": "derived_from",
          "note": "差し戻し指摘をすべて解消"
        }
      ],
      "status": "completed",
      "artifacts": [
        "docs/security-policy-v2.md"
      ]
    },
    {
      "id": "tkt-review-doc-02",
      "task": "tsk-security-policy",
      "worker_id": "wkr-02",
      "session_id": "ssn-wkr02-02",
      "objective": "改訂版セキュリティ運用規約の最終承認レビューを行う",
      "jobs": [
        "job-review-policy-02"
      ],
      "depends": [
        "tkt-revise-doc-02"
      ],
      "refs": [
        {
          "ticket_id": "tkt-revise-doc-02",
          "kind": "related",
          "note": "改訂版 docs/security-policy-v2.md を最終確認し approved"
        }
      ],
      "status": "completed",
      "artifacts": [
        "reviews/security-policy-review-02.md"
      ]
    }
  ]
}
```
