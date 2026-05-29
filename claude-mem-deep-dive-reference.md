# claude-mem 仕組み解説（詳細・参考資料）

> 本ドキュメントは技術的な裏側まで踏み込んだ参考資料です。
> 非エンジニア向けの研修内容は `claude-mem-training.md` を参照してください。

---

## 0. claude-mem とは

Claude Code 用の永続メモリプラグイン。会話の内容を要約して永続データベースに保存し、新しいセッションで自動的に過去の文脈を呼び戻す。

- 配布元：`thedotmack/claude-mem` プラグイン
- 内部要約モデル：Claude Sonnet 4.5
- ストレージ：SQLite（プライマリ）+ Chroma（ベクトル検索）

---

## 1. 動作タイミング（hook 構成）

`hooks.json` で次のフックが登録されている：

| フック | 発火タイミング | 行う処理 |
|---|---|---|
| `SessionStart` | 新しいセッション開始時 | ワーカー起動／コンテキスト注入 |
| `UserPromptSubmit` | ユーザーがプロンプトを送信した時 | セッション初期化 |
| `PostToolUse` | Claude がツールを使うたび | `pending_messages` テーブルに enqueue（**type='observation'**） |
| `Stop` | Claude の応答が終わるたび | `pending_messages` に enqueue（**type='summarize'**）→ ワーカーが Sonnet 4.5 で要約 → 観測として保存 |
| `SessionEnd` | セッション終了時 | `POST 127.0.0.1:37777/api/sessions/complete` で完了マーク |

ポイント：

- **保存処理の本体は `Stop` フック**で走る。`PostToolUse` は素材をキューに積むだけ。
- `SessionEnd` は完了マークだけで、保存はもう終わっている。
- ワーカー (`worker-service.cjs`) は `127.0.0.1:37777` で常駐し、HTTP API で受け付ける。

---

## 2. ストレージ構造

```
~/.claude-mem/
├── claude-mem.db          (約19MB) ─ メイン SQLite DB
│   └── テーブル:
│       ├── observations        要約済み観測（要約 LLM の出力）
│       ├── observations_fts    FTS5 全文検索インデックス
│       ├── session_summaries   セッション要約
│       ├── user_prompts        ユーザー発言
│       ├── pending_messages    未処理キュー（observation / summarize）
│       └── sdk_sessions        セッション管理
├── chroma/chroma.sqlite3  (約98MB) ─ ベクトル DB（埋め込み）
├── settings.json          ─ 動作設定
├── logs/                  ─ フック実行ログ
└── supervisor.json        ─ ワーカー監視
```

**重要**：このストレージは **ユーザー単位でグローバル**。プロジェクトフォルダ内には何も保存されない。

---

## 3. プロジェクト識別ロジック

`scripts/context-generator.cjs` から minified 解読：

```js
function getProjectName(cwd) {
  if (!cwd) return "unknown-project";
  return path.basename(cwd);     // cwd の末端フォルダ名のみ
}
```

`observations.project` カラムにこの basename が入る。実 DB の値例：

```
source-code-review      167件
copy_site_by_claude     175件
mcp_client               91件
unknown-project          (ルートや空 cwd で起動した場合のごみ箱)
```

これにより以下が成り立つ：

| 状況 | 結果 |
|---|---|
| 同じフォルダで再起動 | 同一 project → 過去観測が継続される |
| basename が同じ別フォルダ（例：`~/work/foo` と `~/personal/foo`） | **同じレーンに混入する** |
| basename が違うフォルダ | 別レーン扱い。**自動注入では引かれない**（DB には残る） |
| フォルダリネーム | 旧名で残骸が残り、新名では引かれなくなる |
| `/` や空 cwd | `unknown-project` レーンに集約される |

---

## 4. SessionStart 時のコンテキスト注入

`SessionStart` フック内の `worker-service.cjs hook claude-code context` が以下を行う：

1. `process.cwd()` の basename を取得
2. SQLite から `WHERE project = ?` で**最新観測 50 件**を取得（`CLAUDE_MEM_CONTEXT_OBSERVATIONS=50`）
3. `session_summaries` の最新 10 件を取得（`CLAUDE_MEM_CONTEXT_SESSION_COUNT=10`）
4. `<claude-mem-context>...</claude-mem-context>` で囲んでセッション冒頭に注入

ここで使われる検索は **時系列クエリ（キー検索）のみ**。Chroma（ベクトル検索）は呼ばれない。
つまり **「RAG として意味的に類似メモを引いてくる」のは自動注入では行われない**。

---

## 5. 検索経路と横断性

| 経路 | 横断検索 | 検索方式 |
|---|---|---|
| **SessionStart 自動注入** | ❌（cwd basename で完全フィルタ） | SQLite キー検索 |
| `/mem-search` Skill | ⭕ project 未指定で全プロジェクト横断 | SQLite FTS5 |
| `smart-search` Skill | ⭕ | Chroma ベクトル検索（**ここが本来の RAG**） |
| `smart-outline` / `timeline` | ⭕ | SQLite 集計 |
| MCP `get_observations([ID])` | ⭕ ID 指定 | キー検索 |
| MCP `/api/search` | ⭕ | SQLite |
| MCP `/api/search/by-concept` | ⭕ | 概念タグ検索 |
| MCP `/api/search/by-file` | ⭕ | ファイルパス検索 |
| MCP `/api/search/by-type` | ⭕ | 観測タイプ検索 |

**まとめ：自動では同プロジェクトしか引かれないが、明示的な検索 API を呼べば全プロジェクト横断可能。**

---

## 6. 設定項目（`~/.claude-mem/settings.json`）

| キー | デフォルト | 意味 |
|---|---|---|
| `CLAUDE_MEM_MODEL` | `claude-sonnet-4-5` | 要約 LLM |
| `CLAUDE_MEM_CONTEXT_OBSERVATIONS` | `50` | 自動注入する観測数 |
| `CLAUDE_MEM_CONTEXT_SESSION_COUNT` | `10` | 自動注入するセッション要約数 |
| `CLAUDE_MEM_WORKER_PORT` | `37777` | ワーカーの HTTP ポート |
| `CLAUDE_MEM_SKIP_TOOLS` | `ListMcpResourcesTool,SlashCommand,Skill,TodoWrite,AskUserQuestion` | 観測対象外ツール |
| `CLAUDE_MEM_PROVIDER` | `claude` | 要約プロバイダ（claude/gemini/openrouter） |
| `CLAUDE_MEM_DATA_DIR` | `~/.claude-mem` | データ保存先 |
| `CLAUDE_MEM_MAX_CONCURRENT_AGENTS` | `2` | 同時実行 LLM 数 |
| `CLAUDE_MEM_EXCLUDED_PROJECTS` | `(空)` | 除外プロジェクト名 |
| `CLAUDE_MEM_CHROMA_ENABLED` | `true` | ベクトル DB 有効化 |

---

## 7. データフロー全体図

```
   [ユーザー入力]
        │
        ▼
   ┌────────────────────────┐
   │ Claude Code 本体        │
   │  ・ツール実行            │──── PostToolUse hook ──┐
   │  ・応答生成              │                       │
   │  ・応答完了              │──── Stop hook ────────┤
   └────────────────────────┘                       │
                                                    ▼
                                       ┌─────────────────────────┐
                                       │ pending_messages         │
                                       │  (SQLite キューテーブル)  │
                                       │  type='observation'      │
                                       │  type='summarize'        │
                                       └────────────┬─────────────┘
                                                    │
                                                    ▼
                                       ┌─────────────────────────┐
                                       │ worker-service.cjs       │
                                       │  ・キュー pull           │
                                       │  ・Sonnet 4.5 で要約     │
                                       │  ・観測テーブルへ INSERT │
                                       │  ・Chroma へ embed       │
                                       └────────────┬─────────────┘
                                                    │
                                                    ▼
                                       ┌─────────────────────────┐
                                       │ observations (SQLite)    │
                                       │ chroma.sqlite3 (Vector)  │
                                       └─────────────────────────┘
                                                    │
                                                    │  (次セッション起動時)
                                                    ▼
                                       ┌─────────────────────────┐
                                       │ SessionStart hook        │
                                       │  context-generator       │
                                       │  → <claude-mem-context>  │
                                       │     を冒頭に注入         │
                                       └─────────────────────────┘
```

---

## 8. 落とし穴チェックリスト

- [ ] basename が衝突するフォルダを併用していないか（`~/work/foo` と `~/personal/foo`）
- [ ] フォルダをリネームした後、旧名の観測を移行したか（`UPDATE observations SET project = 'new' WHERE project = 'old'`）
- [ ] ホーム直下や `/` から `claude` を起動して `unknown-project` を汚していないか
- [ ] ワーカー（`127.0.0.1:37777`）が落ちていないか（`curl http://127.0.0.1:37777/api/health`）
- [ ] DB ファイルがバックアップ対象に入っているか（`~/.claude-mem/claude-mem.db`）

---

## 9. メンテナンスコマンド例

```bash
# プロジェクト一覧と観測数
sqlite3 ~/.claude-mem/claude-mem.db "SELECT project, COUNT(*) FROM observations GROUP BY project ORDER BY 2 DESC"

# 観測のプロジェクト名一括移行（リネーム後）
sqlite3 ~/.claude-mem/claude-mem.db "UPDATE observations SET project='new-name' WHERE project='old-name'"

# ワーカー状態確認
curl -s http://127.0.0.1:37777/api/health

# 未処理キューの状態
sqlite3 ~/.claude-mem/claude-mem.db "SELECT status, COUNT(*) FROM pending_messages GROUP BY status"

# Web UI（リアルタイムメモリストリーム）
open http://localhost:37777
```

---

## 10. 結論

claude-mem は **「ユーザーグローバルなストアにフォルダ名でラベル付けした要約観測を蓄積し、新セッション開始時に同ラベルの最新観測を自動注入する」** 仕組み。ベクトル検索（RAG）は搭載しているが、**自動注入はベクトル検索ではなく時系列キー検索**で行う。横断検索は `mem-search` / `smart-search` などを明示的に呼んだ場合にのみ作動する。
