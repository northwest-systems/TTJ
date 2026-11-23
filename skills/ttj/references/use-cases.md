# TTJ 1.1.0 代表ユースケースとサンプルパターン（UC-01〜UC-10）

本文書は、[規格本文](specification.md) および [JSON参照形式](json-format.md) を実運用へ適用する際に迷いやすい代表的な **10場面（UC-01〜UC-10）** について、**「どのような場面か（シナリオ）」「規格上のポイント（どの状態・フィールドを使うか）」「サンプル JSON パターン」** をまとめた実践リファレンスである。

すべての JSON サンプルは文書ルートに `"protocol": "TTJ/1.1.0"` を持ち、確定済みの ID 接頭辞（`pj-`・`tsk-`・`tkt-`・`job-`・`run-`・`wkr-`）に従う完全な JSON として記述している。

---

## 目次

1. [UC-01: 基本の実行フロー](#uc-01-基本の実行フロー)
2. [UC-02: 複数 Worker による並列実行](#uc-02-複数-worker-による並列実行)
3. [UC-03: Ticket 間の依存関係（`depends_on` / `depends`）とブロック解除](#uc-03-ticket-間の依存関係depends_on--dependsとブロック解除)
4. [UC-04: 外部待機（`waiting`）と一時停止（`paused`）の使い分け](#uc-04-外部待機waitingと一時停止pausedの使い分け)
5. [UC-05: Job の失敗（`failed`）と同一 Claim 内での再試行](#uc-05-job-の失敗failedと同一-claim-内での再試行)
6. [UC-06: Worker 無応答・中断時の安全な巻き取り（Takeover）](#uc-06-worker-無応答中断時の安全な巻き取りtakeover)
7. [UC-07: Task 所属の揮発性 Handoff による引き継ぎ](#uc-07-task-所属の揮発性-handoff-による引き継ぎ)
8. [UC-08: 単独 Job の中止（`canceled`）と Task 全体の中止](#uc-08-単独-job-の中止canceledと-task-全体の中止)
9. [UC-09: レビュー差し戻し・追加 Job の発生](#uc-09-レビュー差し戻し追加-job-の発生)
10. [UC-10: Task 完了後の整理（Ticket 肥大化対策）](#uc-10-task-完了後の整理ticket-肥大化対策)

---

## UC-01: 基本の実行フロー

### どのような場面か（シナリオ）
Project 配下に長寿命の成果目標である `Task` を作成し、単一 Worker（`wkr-01`）が担当する `Ticket` と最小実行単位の `Job` 列に分解して正式作成する。その後、`wkr-01` の `Session` が `Execution Claim` を取得して `Run` を開始し、確認可能な証拠に基づいて作業完了（done：`status: "completed"` / `Run.result: "succeeded"`）まで一方向に進める最も基本的なシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **正式作成と初期状態**：Task・Ticket・Job の初期 `status` はすべて `"todo"`。Ticket 正式作成時に論理的な初期 `Execution Claim`（`state: "unassigned"`, `session_id: null`, `active_run_id: null`, `generation: 1`, `run_seq: 0`）が成立し、確定後は Ticket の `owner_id` と `jobs` 配列（順序固定）を変更しない。
- **Execution Claim 取得と Run 開始（開始前照合）**：`Session.worker_id == Ticket.owner_id` を満たす生存 Session が Claim を取得すると `state: "active"`, `generation: 1 → 2` となり、初回 `Run`（`run-*`）を結合すると `active_run_id: "run-..."`, `run_seq: 0 → 1` となる。実作業開始と同時に Job・Ticket・`todo` の親 Task を一括で `status: "doing"` に遷移させ、`Run.started` と `Run.activity: "alive"` を記録する。
- **完了（done / succeeded）と `status` / `result` 分離**：TTJ 1.1.0 では「作業工程の完了（done）」を Task・Ticket・Job の `status: "completed"` で表し、Job の業務判定を `Job.result`（例：`"passed"`）、実行試行の成功を `Run.result: "succeeded"`（`activity: "stopped"`, `ended` 記録）として明確に分離する。
- **試行閉鎖と Claim 解放**：Run 終了確定と同時に `claim.active_run_id` を `null` に戻して `run_seq` を `+1`（`1 → 2`）し、全 Job 終端により Ticket が `"completed"` になると同時に Claim を `state: "released"`（`generation: 2 → 3`, `session_id: null`）へ解放する。

### サンプル JSON パターン

#### パターン 1-A：正式作成直後（`todo` / Claim `unassigned`）

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
    "id": "session-wkr01-01",
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
    "owner_id": "wkr-01",
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
  ],
  "claim": {
    "task_id": "tsk-auth-endpoint",
    "ticket_id": "tkt-impl-auth-01",
    "owner_id": "wkr-01",
    "session_id": null,
    "active_run_id": null,
    "generation": 1,
    "run_seq": 0,
    "state": "unassigned"
  }
}
```

#### パターン 1-B：Claim 取得・最初の Job の Run 実行中（`doing`）

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
    "owner_id": "wkr-01",
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
  "claim": {
    "task_id": "tsk-auth-endpoint",
    "ticket_id": "tkt-impl-auth-01",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-01",
    "active_run_id": "run-code-handler-01",
    "generation": 2,
    "run_seq": 1,
    "state": "active",
    "acquired": "2026-02-01T09:05:00Z"
  },
  "run": {
    "id": "run-code-handler-01",
    "ticket_id": "tkt-impl-auth-01",
    "job": "job-code-handler-01",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-01",
    "generation": 2,
    "started": "2026-02-01T09:05:00Z",
    "activity": "alive",
    "observed": "2026-02-01T09:10:00Z"
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
    "owner_id": "wkr-01",
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
  "claim": {
    "task_id": "tsk-auth-endpoint",
    "ticket_id": "tkt-impl-auth-01",
    "owner_id": "wkr-01",
    "session_id": null,
    "active_run_id": null,
    "generation": 3,
    "run_seq": 4,
    "state": "released"
  },
  "runs": [
    {
      "id": "run-code-handler-01",
      "ticket_id": "tkt-impl-auth-01",
      "job": "job-code-handler-01",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T09:05:00Z",
      "ended": "2026-02-01T09:20:00Z",
      "activity": "stopped",
      "observed": "2026-02-01T09:20:00Z",
      "result": "succeeded"
    },
    {
      "id": "run-unit-test-01",
      "ticket_id": "tkt-impl-auth-01",
      "job": "job-unit-test-01",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T09:21:00Z",
      "ended": "2026-02-01T09:30:00Z",
      "activity": "stopped",
      "observed": "2026-02-01T09:30:00Z",
      "result": "succeeded"
    }
  ]
}
```

---

## UC-02: 複数 Worker による並列実行

### どのような場面か（シナリオ）
同一の `Task` 配下に、互いに実行依存を持たない独立した複数の `Ticket`（バックエンド実装 `tkt-backend-01` とフロントエンド実装 `tkt-frontend-02`）を作成し、`wkr-01` と `wkr-02` がそれぞれの Session で並行して `Execution Claim` を取得し `Run` を同時実行するシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **単一 Owner 原則と Ticket 分割**：1つの Ticket に複数 Owner を設定することは禁止されているため、複数 Worker（`wkr-01`, `wkr-02`）が分担する場合は Worker ごとに別 Ticket を作成し、それぞれ単一不変の `owner_id` を設定する。
- **Ticket 単位の Execution Claim**：`Execution Claim` の一意キーは `ticket_id` である。同一 Task 配下であっても、別 Ticket であれば独立した Claim を持ち、同時に `state: "active"`・`active_run_id: "run-..."` となって並列実行できる。
- **並列実行の非阻害**：一方の Ticket が `waiting` や `blocked` になったり、Task 所属の単一スロット `Handoff` が一方の Ticket 文脈を表示していたりしても、無関係な独立 Ticket の開始・並行実行を遮ったり直列化したりしてはならない（[規格本文](specification.md) §6・§8.2・§9.1・禁止事項 #17）。

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
      "id": "session-wkr01-01",
      "worker_id": "wkr-01",
      "started": "2026-02-01T10:00:00Z"
    },
    {
      "id": "session-wkr02-01",
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
      "owner_id": "wkr-01",
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
      "owner_id": "wkr-02",
      "objective": "検索UIコンポーネントを実装する",
      "jobs": [
        "job-impl-search-ui"
      ],
      "depends": [],
      "status": "doing"
    }
  ],
  "claims": [
    {
      "task_id": "tsk-search-feature",
      "ticket_id": "tkt-backend-01",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "active_run_id": "run-search-api-01",
      "generation": 2,
      "run_seq": 1,
      "state": "active",
      "acquired": "2026-02-01T10:01:00Z"
    },
    {
      "task_id": "tsk-search-feature",
      "ticket_id": "tkt-frontend-02",
      "owner_id": "wkr-02",
      "session_id": "session-wkr02-01",
      "active_run_id": "run-search-ui-01",
      "generation": 2,
      "run_seq": 1,
      "state": "active",
      "acquired": "2026-02-01T10:01:30Z"
    }
  ],
  "runs": [
    {
      "id": "run-search-api-01",
      "ticket_id": "tkt-backend-01",
      "job": "job-impl-search-api",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T10:01:00Z",
      "activity": "alive",
      "observed": "2026-02-01T10:05:00Z"
    },
    {
      "id": "run-search-ui-01",
      "ticket_id": "tkt-frontend-02",
      "job": "job-impl-search-ui",
      "owner_id": "wkr-02",
      "session_id": "session-wkr02-01",
      "generation": 2,
      "started": "2026-02-01T10:01:30Z",
      "activity": "alive",
      "observed": "2026-02-01T10:05:00Z"
    }
  ]
}
```

---

## UC-03: Ticket 間の依存関係（`depends_on` / `depends`）とブロック解除

### どのような場面か（シナリオ）
後続 Ticket が先行 Ticket の完了を前提としている場面。先行 Ticket が未完了（あるいは先行 Ticket が中止となり継続不能な `blocked` 状態）のため後続 Ticket が開始を阻まれている状態から、先行 Ticket（または代替 Ticket）の `completed` 確定によってブロックが解除され、後続 Ticket が実行可能な `"todo"` へ遷移するシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **実行依存フィールド名 `depends`**：他ツールの `depends_on` に相当する実行依存は、TTJ 1.1.0 では `Ticket.depends`（前提 Ticket ID の配列 `["tkt-..."]`）として記述する。情報共有専用の `refs` とは明確に区別し、循環依存は厳禁とする。
- **先行 Ticket 未完了時の状態（計画内待機 `waiting` と計画外障害 `blocked`）**：
  1. **通常の先行 Ticket 完了待ち**：先行 Ticket が進行中（`todo` / `doing` 等）の間、後続 Ticket は計画内の前提待機として `status: "waiting"` とし、先行 Ticket が `status: "completed"` になった時点で依存評価操作により `waiting → todo` へ解除される。
  2. **先行 Ticket の中止や成果物欠落による `blocked` とその解除**：先行 Ticket（例：`tkt-schema-01`）が `status: "canceled"` になった場合など、現行計画のままでは継続不能になったとき、後続 Ticket（`tkt-migration-02`）は `status: "blocked"` に遷移し `resume` に解除条件を永続記録する。その後、代替の先行 Ticket（`tkt-schema-02`）が完了し、未着手の後続 Ticket の `depends` 更新と課題解消が確認された時点で `blocked → todo` へ遷移してブロック解除される（[規格本文](specification.md) §5.2・§7）。
- **`active` Claim 先行取得時の依存評価**：後続 Ticket が未着手（現在 Job 未着手かつ `active_run_id == null`）であれば、Claim が `unassigned` の場合だけでなく、Owner の Session が `active` Claim を先行取得している場合でも依存評価操作で `todo` へ解除できる（ただし依存評価自体は Run を自動開始しない）。

### サンプル JSON パターン

#### パターン 3-A：先行 Ticket 未完了・中止により後続 Ticket が `blocked`（または `waiting`）で待機中

```json
{
  "protocol": "TTJ/1.1.0",
  "tickets": [
    {
      "id": "tkt-schema-01",
      "task": "tsk-db-upgrade",
      "owner_id": "wkr-01",
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
      "owner_id": "wkr-01",
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
      "owner_id": "wkr-02",
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
      "owner_id": "wkr-01",
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
      "owner_id": "wkr-02",
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
  ],
  "claim": {
    "task_id": "tsk-db-upgrade",
    "ticket_id": "tkt-migration-02",
    "owner_id": "wkr-02",
    "session_id": "session-wkr02-01",
    "active_run_id": null,
    "generation": 2,
    "run_seq": 0,
    "state": "active",
    "acquired": "2026-02-01T11:00:00Z"
  }
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
    "owner_id": "wkr-01",
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
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-01",
    "generation": 2,
    "started": "2026-02-01T12:00:00Z",
    "activity": "alive",
    "observed": "2026-02-01T12:05:00Z",
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
    "owner_id": "wkr-01",
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
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-01",
    "generation": 2,
    "started": "2026-02-01T12:00:00Z",
    "ended": "2026-02-01T12:10:00Z",
    "activity": "stopped",
    "observed": "2026-02-01T12:10:00Z",
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

## UC-05: Job の失敗（`failed`）と同一 Claim 内での再試行

### どのような場面か（シナリオ）
Job の 1 回目の実行試行（`run-integration-01`）が一時的な通信エラーやタイムアウトで失敗（`Run.result: "failed"`）したものの、同じ作業計画（同一 Ticket・同一 Job）のまま回復可能であるため、同一 `Execution Claim`（同じ `session_id`・同じ `generation`）内で旧 Run を閉鎖し、`run_seq` を進めて新 Run（`run-integration-02`）で再試行するシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **回復可能な中断と Job `status: "doing"` の維持**：Run が 1 回失敗・終了しただけで即座に Job や Ticket を `completed`・`canceled`・`blocked` にしてはならない（[規格本文](specification.md) §8.2「回復可能な中断」）。同一計画で再試行できる間は Job・Ticket ともに `status: "doing"` を維持する。
- **旧 Run の閉鎖と `run_seq` 加算**：旧 Run（`run-integration-01`）の終了確定時に `result: "failed"`（および `ended`）を必須記録し、同時にその試行権限を即座に閉じる（`claim.active_run_id = null`, `claim.run_seq: 1 → 2`）。なお、旧 Run の `activity` が `unknown` の場合でも、同一 Claim 内の通常再試行に限り、外部フェンシングで旧 Run の外部影響を遮断でき残存副作用がない安全条件を満たせば安全閉鎖できる。
- **新 Run の開始前照合と古い前提の並行再試行拒否**：同じ `active` Claim（`generation: 2`）で新 Run `run-integration-02` を開始する際、`claim.active_run_id == null` かつ要求の `expected_run_seq` が現行 `claim.run_seq`（`2`）と一致することを検証し、原子的に `claim.active_run_id = "run-integration-02"`, `claim.run_seq: 2 → 3` へ更新する。これにより、旧状態（`expected_run_seq: 0` 等）を前提に重複発行された遅延リトライ要求や閉じた旧 `run-integration-01` からの更新は確実に拒否される。

### サンプル JSON パターン

#### パターン 5-A：1回目の Run（`run-integration-01`）が `failed` で閉鎖された直後（`active_run_id: null`, `run_seq: 2`）

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-verify-api-01",
    "task": "tsk-auth-endpoint",
    "owner_id": "wkr-01",
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
  "claim": {
    "task_id": "tsk-auth-endpoint",
    "ticket_id": "tkt-verify-api-01",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-01",
    "active_run_id": null,
    "generation": 2,
    "run_seq": 2,
    "state": "active",
    "acquired": "2026-02-01T13:00:00Z"
  },
  "run": {
    "id": "run-integration-01",
    "ticket_id": "tkt-verify-api-01",
    "job": "job-run-integration",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-01",
    "generation": 2,
    "started": "2026-02-01T13:00:00Z",
    "ended": "2026-02-01T13:04:00Z",
    "activity": "stopped",
    "observed": "2026-02-01T13:04:00Z",
    "result": "failed"
  }
}
```

#### パターン 5-B：同一 Claim 内で `run_seq` を進めて新 Run（`run-integration-02`）を結合し再試行中（`run_seq: 3`）

```json
{
  "protocol": "TTJ/1.1.0",
  "claim": {
    "task_id": "tsk-auth-endpoint",
    "ticket_id": "tkt-verify-api-01",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-01",
    "active_run_id": "run-integration-02",
    "generation": 2,
    "run_seq": 3,
    "state": "active",
    "acquired": "2026-02-01T13:00:00Z"
  },
  "runs": [
    {
      "id": "run-integration-01",
      "ticket_id": "tkt-verify-api-01",
      "job": "job-run-integration",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T13:00:00Z",
      "ended": "2026-02-01T13:04:00Z",
      "activity": "stopped",
      "observed": "2026-02-01T13:04:00Z",
      "result": "failed"
    },
    {
      "id": "run-integration-02",
      "ticket_id": "tkt-verify-api-01",
      "job": "job-run-integration",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T13:06:00Z",
      "activity": "alive",
      "observed": "2026-02-01T13:08:00Z"
    }
  ]
}
```

---

## UC-06: Worker 無応答・中断時の安全な巻き取り（Takeover）

### どのような場面か（シナリオ）
作業中の Worker / Session が無応答または中断したため、別の Worker（または同一 Worker の別 Session）が安全に作業を巻き取る（Takeover）シナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **Heartbeat 途絶（`unknown`）と停止確認済み（`stopped`）の厳格な区別**：Heartbeat 途絶やタイムアウトだけでは `Run.activity` は `"unknown"` となる。`alive` または `unknown` の Run が残存している間は巻き取りを**絶対に行ってはならない**（禁止事項 #13, #14, #15）。独立した監視機構により対象 Run の実行主体・配下コンテキストの完全停止が確認されて `Run.activity: "stopped"` が記録され、かつ `active_run_id == null` および残存副作用がない安全条件が成立した場合のみ巻き取りが許可される。
- **切替時の競合境界**：停止・安全確認時点の `(expected_generation, expected_run_seq)` が現行 Claim の `(generation, run_seq)` と一致することを原子的に検証する（確認後に旧 Session で新 Run が介入した場合は拒否）。
- **引き継ぎの 2 パターン**：
  1. **別 Worker への巻き取り（新 Ticket への単一原子確定）**：`Ticket.owner_id` は変更禁止のため、別 Worker（`wkr-01` → `wkr-02`）が引き継ぐ場合は、途中状態を一切公開しない**単一の原子的確定**として、(a) 旧 Ticket `tkt-deploy-01` を `status: "canceled"`（`cancel_reason` 必須・`owner_id` 不変・未終端 Job も `canceled`）へ終端化、(b) 旧 Claim を `state: "revoked"`（`generation: 2 → 3`, `session_id: null`, `active_run_id: null`、再取得永久禁止）へ失効し再送用 `takeover` オブジェクトを記録、(c) `wkr-02` を `owner_id` とする後続 Ticket `tkt-deploy-02`（`refs` に `tkt-deploy-01` を記録）とその初期 Claim（`session-wkr02-01` が即時取得する場合は `generation: 2`, `state: "active"`）を一体確定する。
  2. **同一 Worker の別 Session による同一 Ticket 上の安全移譲（CAS 移譲）**：同じ `wkr-01` の新 Session（`session-wkr01-02`）が同一 Ticket の未完了 Job からそのまま継続する場合は、全過去 Run `stopped`・`active_run_id == null`・安全条件成立のもと、原子的 CAS で同一 Ticket の Claim を `state: "active"`, `session_id: "session-wkr01-02"`, `generation: 2 → 3` へ更新する。

### サンプル JSON パターン

#### パターン 6-A：別 Worker（`wkr-02`）への新 Ticket 単一原子巻き取り確定後の状態

```json
{
  "protocol": "TTJ/1.1.0",
  "tickets": [
    {
      "id": "tkt-deploy-01",
      "task": "tsk-release-v110",
      "owner_id": "wkr-01",
      "objective": "ステージング環境へパッケージを配布し疎通確認を行う",
      "jobs": [
        "job-upload-pkg-01",
        "job-smoke-test-01"
      ],
      "status": "canceled",
      "cancel_reason": "session-wkr01-01 の異常停止確認および環境安全確認完了に伴い、op-takeover-deploy-01 により後続 Ticket tkt-deploy-02 (wkr-02) へ単一原子巻き取りを実施。",
      "artifacts": [
        "packages/app-v110.tar.gz"
      ]
    },
    {
      "id": "tkt-deploy-02",
      "task": "tsk-release-v110",
      "owner_id": "wkr-02",
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
  "claims": [
    {
      "task_id": "tsk-release-v110",
      "ticket_id": "tkt-deploy-01",
      "owner_id": "wkr-01",
      "session_id": null,
      "active_run_id": null,
      "generation": 3,
      "run_seq": 4,
      "state": "revoked",
      "revoked": "2026-02-01T14:20:00Z",
      "takeover": {
        "operation_id": "op-takeover-deploy-01",
        "claimant_session_id": "session-wkr02-01",
        "target_owner_id": "wkr-02",
        "successor_ticket_id": "tkt-deploy-02"
      }
    },
    {
      "task_id": "tsk-release-v110",
      "ticket_id": "tkt-deploy-02",
      "owner_id": "wkr-02",
      "session_id": "session-wkr02-01",
      "active_run_id": "run-smoke-test-02",
      "generation": 2,
      "run_seq": 1,
      "state": "active",
      "acquired": "2026-02-01T14:20:00Z"
    }
  ],
  "runs": [
    {
      "id": "run-smoke-test-01",
      "ticket_id": "tkt-deploy-01",
      "job": "job-smoke-test-01",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T14:00:00Z",
      "ended": "2026-02-01T14:15:00Z",
      "activity": "stopped",
      "observed": "2026-02-01T14:18:00Z",
      "result": "interrupted"
    },
    {
      "id": "run-smoke-test-02",
      "ticket_id": "tkt-deploy-02",
      "job": "job-smoke-test-02",
      "owner_id": "wkr-02",
      "session_id": "session-wkr02-01",
      "generation": 2,
      "started": "2026-02-01T14:21:00Z",
      "activity": "alive",
      "observed": "2026-02-01T14:22:00Z"
    }
  ]
}
```

#### パターン 6-B：同一 Worker（`wkr-01`）の別 Session（`session-wkr01-02`）へ同一 Ticket 上で CAS 移譲した場合の Claim

```json
{
  "protocol": "TTJ/1.1.0",
  "claim": {
    "task_id": "tsk-release-v110",
    "ticket_id": "tkt-deploy-01",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-02",
    "active_run_id": "run-smoke-test-02",
    "generation": 3,
    "run_seq": 5,
    "state": "active",
    "acquired": "2026-02-01T14:20:00Z"
  }
}
```

---

## UC-07: Task 所属の揮発性 Handoff による引き継ぎ

### どのような場面か（シナリオ）
作業途中の文脈・停止位置・注意点を Task 所属の揮発性単一スロット `Handoff` に記録して後続 Session や他メンバーが状況を把握できるようにし、Task 完了時には揮発性 Handoff を破棄・消去して、今後再利用すべき長期知見だけを `Project.knowledge` へ残すシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **Task 所属の揮発性単一スロット**：v1.1.0 では `Ticket.handoffs` を廃止し、`Task` ごとに最大1つの揮発性 `Handoff` スロットを持つ。必須項目は `task_id`、揮発スロット書込連番 `revision`、およびスロット消失をまたいで鮮度比較できる永続正本側の単調確定版 `source_revision`。
- **更新 3 経路の分離と閲覧・実行の分離**：Handoff の更新は (1) 現行結合 Run 起因（`claim.active_run_id == run_id` 非 `null` 一致）、(2) Run 非紐付けの `active` Claim 保持 Session 更新、(3) 認可された管理再生成（Claim 不要・書込み時点でも `source_revision` が最新であることを検証・実行権非付与）の 3 経路に限定する。また、Handoff を閲覧できることは実行権（Claim）を意味しない（禁止事項 #11）。
- **正本保存の原則と Task 完了時の揮発・消去**：再開に必須の条件（`resume`, `Run.wait`, `cancel_reason`, `artifacts`）は必ず永続正本側に保存し、Handoff を正式履歴の正本にしてはならない（禁止事項 #16）。Task 完了時に Handoff スロットが揮発・消去されても正式履歴は一切失われず、再利用する知見のみを選別して `Project.knowledge`（`status: "active"`）へ記録する。

### サンプル JSON パターン

#### パターン 7-A：作業進行中の Task 揮発性 `Handoff` スロットと永続正本

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-tune-cache-01",
    "task": "tsk-latency-improvement",
    "owner_id": "wkr-01",
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
    "revision": 4,
    "source_revision": 11,
    "current_ticket_id": "tkt-tune-cache-01",
    "owner_id": "wkr-01",
    "active_session_id": "session-wkr01-01",
    "current_job_id": "job-run-benchmark",
    "source_generation": 2,
    "activity": "alive",
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
    "owner_id": "wkr-01",
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
Ticket 内の特定の不要になった Job だけを `canceled` にして試行権限を即時失効させつつ後続 Job を続行するケースと、方針変更等により Task 全体（および配下の未終端 Ticket・Job）を一括で `canceled` にするケースの対比シナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **単独 Job の中止（後続 Job を続ける場合）**：
  - 対象 Job を `status: "canceled"` に確定すると**同時に**、結合中 Run があればその試行権限を即座に無効化・閉鎖（`Run.result: "canceled"`, `claim.active_run_id = null`, `claim.run_seq + 1`）し、外部効果確定側でも当該 `Run.id` の書込みを即時拒否可能にして停止要求を発行する。
  - このとき **Ticket 全体の `Execution Claim` は失効（`revoked`）させず `state: "active"` を維持**できる。
  - 後続 Job の新 Run を開始する前に、前の Job の実行に対する「停止確認（`stopped`）**または**残存副作用がない安全条件」を別途確認する。全 Job が `completed` または `canceled` になれば Ticket は `status: "completed"` に集約される（ただし中止した Job の目標を達成したとはみなさない）。
- **Task 全体（または Ticket 全体）の中止**：
  - 親 Task を `status: "canceled"` にすると、配下の**未終端の Ticket・Job もすべて同時に `status: "canceled"`** に確定する（既に `completed` になっていた終端済みの子孫は変更せず保持する）。
  - 中止された各 Ticket には **`cancel_reason` の記録が必須**。
  - 対象範囲の未失効 `Execution Claim`（`unassigned`・`active`・`released`）はすべて **`state: "revoked"`**（`generation + 1`, `session_id: null`, `active_run_id: null`、再取得永久禁止）として失効させる（既に `revoked` 済みの Claim は二重失効しない）。Ticket 自身の中止は全 Job 終端による `completed` 集約より常に優先される。

### サンプル JSON パターン

#### パターン 8-A：単独 Job（`job-optional-seed`）のみを中止し、同一 Claim で後続 Job（`job-verify-env`）を完了させたパターン

```json
{
  "protocol": "TTJ/1.1.0",
  "ticket": {
    "id": "tkt-setup-preview-01",
    "task": "tsk-preview-env",
    "owner_id": "wkr-01",
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
  "claim": {
    "task_id": "tsk-preview-env",
    "ticket_id": "tkt-setup-preview-01",
    "owner_id": "wkr-01",
    "session_id": null,
    "active_run_id": null,
    "generation": 3,
    "run_seq": 6,
    "state": "released"
  },
  "runs": [
    {
      "id": "run-optional-seed-01",
      "ticket_id": "tkt-setup-preview-01",
      "job": "job-optional-seed",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T16:10:00Z",
      "ended": "2026-02-01T16:11:00Z",
      "activity": "stopped",
      "observed": "2026-02-01T16:11:00Z",
      "result": "canceled"
    },
    {
      "id": "run-verify-env-01",
      "ticket_id": "tkt-setup-preview-01",
      "job": "job-verify-env",
      "owner_id": "wkr-01",
      "session_id": "session-wkr01-01",
      "generation": 2,
      "started": "2026-02-01T16:12:00Z",
      "ended": "2026-02-01T16:15:00Z",
      "activity": "stopped",
      "observed": "2026-02-01T16:15:00Z",
      "result": "succeeded"
    }
  ]
}
```

#### パターン 8-B：Task 全体を中止（`canceled`）し、未終端の子孫中止と Claim `revoked` を同時確定したパターン

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
      "owner_id": "wkr-01",
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
      "owner_id": "wkr-02",
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
  ],
  "claim": {
    "task_id": "tsk-legacy-export",
    "ticket_id": "tkt-csv-impl-02",
    "owner_id": "wkr-02",
    "session_id": null,
    "active_run_id": null,
    "generation": 3,
    "run_seq": 2,
    "state": "revoked",
    "revoked": "2026-02-01T16:30:00Z"
  }
}
```

---

## UC-09: レビュー差し戻し・追加 Job の発生

### どのような場面か（シナリオ）
作成 Ticket（`tkt-draft-doc-01`）の完了後に別 Worker（`wkr-02`）がレビュー Ticket（`tkt-review-doc-01`）を実行した結果、修正要求（`changes_requested`）となり、指摘反映と再検証のための追加 Job と新 Run が必要になったシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **レビュー差し戻し時の `status` と `result` の分離**：レビュー Job が差し戻し結論を出した場合、レビューという作業自体は完了しているため `Job.status: "completed"`, `Job.result: "changes_requested"` として確定し、レビュー Ticket も `status: "completed"` に集約する（独立した `Request` オブジェクトは導入しない）。
- **確定後 Ticket への Job 追加禁止・終端 Ticket 再開禁止**：一度正式作成した Ticket の `jobs` 列に後から Job を追加したり、完了済みの作成 Ticket やレビュー Ticket を `doing` / `todo` に再開・巻き戻したりすることは**禁止**（禁止事項 #8, #9）。
- **同一 Task 配下への新 Ticket 起票による追加 Job・新 Run 対応**：差し戻し修正や追加で必要になった Job 列は、**同一 Task（`tsk-*`）配下に新しい Ticket（`tkt-revise-doc-02`）として起票**する（Task を新しく増殖させてはならない）。新 Ticket の `depends` にレビュー Ticket を指定し、`refs`（`kind: "fixes"` / `"derived_from"`）で初稿 Ticket の成果物とレビュー Ticket の指摘メモを参照した上で、新 Ticket の Claim を取得して新 Run を実行する。

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
      "owner_id": "wkr-01",
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
      "owner_id": "wkr-02",
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
      "owner_id": "wkr-01",
      "objective": "レビュー指摘に基づきトークン失効手順を追記・検証する",
      "jobs": [
        "job-add-revocation-section",
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
          "note": "changes_requested の指摘事項（トークン失効手順の不足）を反映"
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
      "id": "job-add-revocation-section",
      "objective": "緊急時のトークン失効手順節を追記する",
      "criteria": [
        "docs/security-policy-v2.md にトークン失効手順が明記されている"
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
  "claim": {
    "task_id": "tsk-security-policy",
    "ticket_id": "tkt-revise-doc-02",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-02",
    "active_run_id": "run-add-revocation-01",
    "generation": 2,
    "run_seq": 1,
    "state": "active",
    "acquired": "2026-02-01T17:00:00Z"
  },
  "run": {
    "id": "run-add-revocation-01",
    "ticket_id": "tkt-revise-doc-02",
    "job": "job-add-revocation-section",
    "owner_id": "wkr-01",
    "session_id": "session-wkr01-02",
    "generation": 2,
    "started": "2026-02-01T17:00:00Z",
    "activity": "alive",
    "observed": "2026-02-01T17:05:00Z"
  }
}
```

---

## UC-10: Task 完了後の整理（Ticket 肥大化対策）

### どのような場面か（シナリオ）
作成・レビュー差し戻し・修正・再レビューを経て複数の Ticket が作られた長寿命 `Task` が目標を達成して完了する際、終了済み Ticket のアーカイブ・折りたたみ、Execution Claim の解放、揮発性 Handoff の整理、および Task 完了判定を行ってクリーンな終了状態にするシナリオ。

### 規格上のポイント（どの状態・フィールドを使うか）
- **終了済み Ticket の即時アーカイブと Claim 解放**：各 Ticket は終了（`completed` / `canceled`）時点で成果（`artifacts`）・結果（`result`）・中止理由（`cancel_reason`）を永続化し、`Execution Claim` を非 `active`（完了時は `state: "released"`、中止・巻き取り時は `state: "revoked"`、いずれも `session_id: null`, `active_run_id: null`）へ解放・失効させる。次 Ticket が未作成でも終了済み Ticket は即座にアーカイブ・折りたたみ可能とする（[規格本文](specification.md) §4・§6）。
- **Ticket 肥大化対策と Task 完了判定**：
  - `refs`（`fixes`, `derived_from`, `rerun_of`, `related`）によって「初稿 → 差し戻しレビュー → 修正 → 承認レビュー」の系列を構造化し、終了済み Ticket を折りたたんで管理負荷を抑える。
  - **完了 Ticket 数だけで Task の達成率や完了を決めてはならない**（[規格本文](specification.md) §2・§6）。全 Ticket が終端状態であり、差し戻しや中止で未達だった必須範囲が後続 Ticket で解決され、`Task.criteria` をすべて満たしていることを確認して `Task.status: "completed"` とする。
- **揮発状態のクリーンアップと長期知識の分離**：Task 完了時には全 Claim が `released` / `revoked`（`session_id: null`, `active_run_id: null`）、全 Run が `activity: "stopped"` となり、Task 所属の揮発性 `Handoff` スロットは長期記憶として残さず破棄・消去する。将来再利用する知見のみを `Project.knowledge`（`status: "active"`、旧知識の置換時は旧記録を `"supersed"` にして `supersedes` で参照）に保持する。

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
        "content": "トークン失効は日次バッチで一括反映する",
        "basis": "初期設計時の暫定運用ルールとして採用していた。"
      },
      {
        "id": "security-policy-immediate-revocation",
        "status": "active",
        "content": "緊急時のトークン失効は即時失効リストへ反映し5分以内に全ゲートウェイへ伝播させる",
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
      "owner_id": "wkr-01",
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
      "owner_id": "wkr-02",
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
      "owner_id": "wkr-01",
      "objective": "レビュー指摘に基づきトークン失効手順を追記・検証する",
      "jobs": [
        "job-add-revocation-section",
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
      "owner_id": "wkr-02",
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
  ],
  "claims": [
    {
      "task_id": "tsk-security-policy",
      "ticket_id": "tkt-draft-doc-01",
      "owner_id": "wkr-01",
      "session_id": null,
      "active_run_id": null,
      "generation": 3,
      "run_seq": 2,
      "state": "released"
    },
    {
      "task_id": "tsk-security-policy",
      "ticket_id": "tkt-review-doc-01",
      "owner_id": "wkr-02",
      "session_id": null,
      "active_run_id": null,
      "generation": 3,
      "run_seq": 2,
      "state": "released"
    },
    {
      "task_id": "tsk-security-policy",
      "ticket_id": "tkt-revise-doc-02",
      "owner_id": "wkr-01",
      "session_id": null,
      "active_run_id": null,
      "generation": 3,
      "run_seq": 4,
      "state": "released"
    },
    {
      "task_id": "tsk-security-policy",
      "ticket_id": "tkt-review-doc-02",
      "owner_id": "wkr-02",
      "session_id": null,
      "active_run_id": null,
      "generation": 3,
      "run_seq": 2,
      "state": "released"
    }
  ]
}
```
