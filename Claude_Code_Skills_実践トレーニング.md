---
marp: true
theme: default
paginate: true
---

# Claude Code Skills 実践トレーニング

2026-04-07

---

## 目次

1. トレーニングの目的
2. スキルの概要
3. 実物で見てみよう：公式 internal-comms スキル
4. ハンズオン：skillsをインストールして使う
5. internal-comms スキルの仕組み解説
6. スキル活用のTips
7. 他の仕組みとの使い分け（CLAUDE.md / MCP / Subagents / Plugin）
8. 参考リンク

---

## トレーニングの目的

本トレーニングの背景やゴール、アジェンダを連携します。

---

## 背景

Skills が非常に盛り上がっています 🔥🔥

- Claude Code (AIエージェント)の動きを **自分好みにカスタマイズ** できる拡張機能
- 手軽に作れて、手軽に共有できます

---

## トレーニングで期待する成果

- **このトレーニング**: `Skills` のセットアップ完了 + 基本的な使い方の理解
- **今後(短期)**: 日常業務で `Skills` を使った効率化を試す人が増える
- **今後(長期)**: チーム内での活用事例・ベストプラクティス・スタンダードが蓄積されていく

---

## トレーニングの流れ

**座学**
- スキルの概要
- スキルの自動発火
- スキルの仕組み
- 活用のTips

**ハンズオン**
- 実際に `Skills` を動かしてみる

---

## スキルの概要

スキルの概要とフォルダ構成を理解します。

---

## スキルの定義

以下、公式の定義です。

> エージェントに新しい機能と専門知識を与えるための、シンプルでオープンなフォーマット。
>
> Agent Skills は、エージェントが発見して使用できる指示、スクリプト、リソースのフォルダです。これにより、エージェントはタスクをより正確かつ効率的に実行できます。 – Overview - Agent Skills

---

## スキルのポイント

ポイントは **フォルダ** であることです。

- 指示（ `SKILL.md` ）だけでなく、スクリプトやリファレンスもまとめて置ける
- エージェントが使うべきスキルを **自動で発見** し、指示に沿って進めつつ、フォルダ内のスクリプトやリファレンスを適宜活用してくれる

---

## Skills が生まれた理由: コンテキストウィンドウ問題

Claude Code のセッションでは、平均して **コンテキストウィンドウの約 65% がシステムプロンプトや過去のやり取りで埋まっている** ことが分かっています。

- 残り 35% で実際のタスクをこなさなければならない
- CLAUDE.md に何でも詰め込むと、さらにこの余裕が減る
- 長い指示を毎回読み込むのはトークンの無駄遣い

→ **「必要なときに、必要な知識だけを読み込む」** 仕組みが求められた

---

## Skills の答え: Progressive Disclosure

Skills はこの問題を **分割した情報を段階的に読む** ことで解決します。

| ステップ | 読み込む量 |
|---|---|
| 常時 | `name` + `description` だけ（数行） |
| スキル発火時 | `SKILL.md` 本文（〜500行） |
| 必要に応じて | `references/` 等の追加ファイル |

CLAUDE.md に全部書く場合と比べて、**使わないスキルの知識はコンテキストを消費しない**。これが Skills というフォーマットが生まれた根本的な理由です。

---

## スキルの変遷

もともと Anthropic が Claude Code 向けに開発した仕組みです。

- Anthropic がオープン標準としてリリース
- 現在は Claude Code 以外のエージェント製品にも採用が広がっている
- エコシステム全体からの貢献を受け付けている

つまり、Claude Code 専用の仕組みではなく **エージェント共通のスキルフォーマット** になりつつあります。

---

## スキルのフォルダ構成

スキルのフォルダ構成は以下のようなものです。

```
my-skill/
├── SKILL.md          # 必須: 指示 + メタデータ
├── scripts/          # 任意: スクリプト
├── references/       # 任意: リファレンス
└── assets/           # 任意: テンプレートやリソース
```

SKILL.md のみ必須です。（= **SKILL.md だけでもOK** ）

---

## 実物で見てみよう: 公式 internal-comms スキル

Anthropic が公開している公式スキルの1つ「internal-comms」を見て、構造を理解しましょう。

社内向けに作られたスキルですが、**今回は社外向けニュースレター作成に応用** していきます。

GitHub: [anthropics/skills](https://github.com/anthropics/skills) > [skills/internal-comms/](https://github.com/anthropics/skills/tree/main/skills/internal-comms)

```
skills/internal-comms/
├── SKILL.md                              ← メインの指示書（ルーター役）
├── examples/
│   ├── 3p-updates.md                     ← Progress / Plans / Problems 更新
│   ├── company-newsletter.md             ← 全社ニュースレター ★今回これを使う
│   ├── faq-answers.md                    ← FAQ 回答
│   └── general-comms.md                  ← その他のコミュニケーション
└── LICENSE.txt
```

---

## internal-comms スキル: frontmatter を読む

SKILL.md の冒頭部分（frontmatter）を見てみましょう。

```yaml
---
name: internal-comms
description: A set of resources to help me write all kinds
  of internal communications, using the formats that my
  company likes to use. Claude should use this skill
  whenever asked to write some sort of internal
  communications (status reports, leadership updates,
  3P updates, company newsletters, FAQs, incident
  reports, project updates, etc.).
license: Complete terms in LICENSE.txt
---
```

---

## frontmatter のポイント

**`name`** : スキルの識別名。スラッシュコマンド `/internal-comms` としても使える

**`description`** : Claude が「このスキルを使うべきか？」を判断する基準

description の書き方が重要です:

- **やること**: 各種社内コミュニケーション文書の作成
- **発火条件**: 「status report」「newsletter」「FAQ」等のキーワード
- **想定ユースケースを列挙**: 3P updates / newsletters / FAQs / incident reports …

→ Claude が「使うべきか迷ったら使う」ように、**ユースケースを具体的に並べる**のがコツ

---

## internal-comms スキル: 本文を読む

frontmatter の下に、Markdown で指示が書かれています。

```markdown
## When to use this skill
To write internal communications, use this skill for:
- 3P updates (Progress, Plans, Problems)
- Company newsletters
- FAQ responses
- ...

## How to use this skill
1. **Identify the communication type** from the request
2. **Load the appropriate guideline file** from `examples/`
3. **Follow the specific instructions** in that file
```

→ 本文は「いつ使うか + どのファイルを開くか」のルーティングに徹している

---

## internal-comms スキル: 詳細は examples/ に分割

SKILL.md はあくまで **ルーター**。詳しい執筆ガイドは `examples/` 配下の各ファイルにあります。

| ファイル | 役割 |
|---|---|
| `SKILL.md` | 全体ルーティング（どの examples を開くか判断） |
| `examples/3p-updates.md` | Progress/Plans/Problems 形式の更新 |
| **`examples/company-newsletter.md`** | **全社ニュースレター（今日これを使う）** |
| `examples/faq-answers.md` | FAQ 回答 |
| `examples/general-comms.md` | その他のコミュニケーション |

→ Claude は最初は SKILL.md だけを読み、必要になった examples ファイルだけを追加で読む（Progressive Disclosure）

---

## examples/company-newsletter.md を覗いてみる

リクエストが「ニュースレター作って」だった場合、Claude はこのファイルだけを開きます。

```markdown
## Instructions
You are being asked to write a company-wide newsletter
update. You are meant to summarize the past week/month
of a company in the form of a newsletter ...
It should be maybe ~20-25 bullet points long. It will
be sent via Slack and email, so make it consumable
for that.

Ideally it includes the following attributes:
- Lots of links: pulling documents from Google Drive ...
- Short and to-the-point: each bullet should probably
  be no longer than ~1-2 sentences
- Use the "we" tense, as you are part of the company.
```

→ 「**何を書くか**」「**何文字くらいか**」「**どんなトーンで**」までが具体的に指示されている

---

## company-newsletter.md: ツール連携の指示も入っている

執筆に必要な情報の集め方まで書かれています。

```markdown
## Tools to use
If you have access to the following tools, please try
to use them.

- Slack: look for messages in channels with lots of
  people, with lots of reactions or responses ...
- Email: look for things from executives that discuss
  company-wide announcements
- Calendar: if there were meetings with large attendee
  lists, particularly things like All-Hands meetings ...
- Documents: if there were new docs published in the
  last week or two that got a lot of attention ...
- External press: if you see references to articles
  or press we've received over the past week ...
```

→ MCP連携している Slack / Gmail / Calendar / Drive を **自動で巡回して素材を集める** ところまで指示できる

---

## company-newsletter.md: テンプレートまで提供

最後に、出力フォーマットの雛形まで書かれています。

```markdown
## Example Formats

:megaphone: Company Announcements
- Announcement 1
- Announcement 2

:dart: Progress on Priorities
- Area 1
    - Sub-area 1

:pillar: Leadership Updates
- Post 1

:thread: Social Updates
- Update 1
```

→ 絵文字付きセクション分けまで標準化されているので、毎回ブレないアウトプットが返ってくる

---

## ハンズオン

実際にニュースレター作成スキルを使ってニュースレターを作成してみます。

---

## 事前準備

- Claude Code がインストール済みであることを前提とします。

- チャット形式のClaude Codeの画面だとコマンドが反応しないことがあるため、ターミナルを使います。

- ハンズオンでは Anthropics公式skillsを使います。

---

## VScodeでプロジェクトを開く

1. エクスプローラーで、今回の研修で利用するためのフォルダを新規の作成する。
2. VScodeでそのフォルダを開く

---

## Vscodeのターミナルパネルを開いてclaude コマンドを実行

vscodeの右上のアイコンのターミナルパネルボタンを押すか、メニューにある「ターミナル -> 新しいターミナル」をクリック

```ターミナル上でコマンドを実行
claude
```

---

## マーケットプレイスからAnthropics公式skillsをインストール

1. ターミナル上のClaude Code で `/plugin` を実行
2. "Marketplaces" タブ まで左右キーボードで移動して選択
3. 「GitHub repo, URL, or path...」の入力欄に`anthropics/skills` を入力して実行
4. "Discover"タグまで左右のキーボードで移動して、Install Pluginsの中から"example-skills"を上下キーボどで選択して実行
5. ３つ目の「Install for you, in this repo only (local scope)」を選択して実行

---

## skillsのインストール先（スコープ）

プラグインをインストールする際、以下の3つのスコープから選択します。用途に応じて使い分けてください。

| 選択肢 | スコープ | 保存先 | 有効範囲 |
|---|---|---|---|
| **Install for you** | user scope | `~/.claude/` | 自分のすべてのプロジェクトで使える |
| **Install for all collaborators on this repository** | project scope | `.claude/` （リポジトリ内、Git管理対象） | このリポジトリを使うチーム全員に共有される |
| **Install for you, in this repo only** | local scope | `.claude/` （リポジトリ内だが `.gitignore` 対象） | 自分だけ、かつこのリポジトリでのみ使える |

```
> Install for you (user scope)
　└ 自分専用・全プロジェクトで利用可。個人の汎用スキルを入れる場合に選択。
> Install for all collaborators on this repository (project scope)
　└ チーム共有・Gitにコミットされる。プロジェクト固有でメンバー全員に配りたい場合に選択。
> Install for you, in this repo only (local scope)
　└ 自分専用・このリポジトリ限定。試験的に使いたい、他メンバーに影響させたくない場合に選択。
> Back to plugin list
　└ 前のプラグイン一覧に戻る。
```

---

### 💡 使い分けの目安

- **迷ったら `user scope`** — 個人学習・検証目的ならこれ
- **チームで統一したい** → `project scope`（`.claude/` をGit管理に含める必要あり）
- **一時的に試したい／個人設定を他メンバーに押し付けたくない** → `local scope`

今回のトレーニングでは、受講者ごとに独立して操作するため **`Install for you, in this repo only (local scope)`** を選択します。

---

## スキルはどこにインストールされているのか？

先ほどのインストール先にスキルはありますか？

---

## スキルはどこに置かれる？

`/plugin` でインストールしたスキルは、必ず **プラグインキャッシュ領域** に展開されます。

```
~/.claude/plugins/cache/<marketplace>/<plugin>/<hash>/skills/
```

これは「マーケットプレイスが管理する領域」で、自動更新・バージョン管理・一括アンインストールが効きます。一方、自分で作った／改造したスキルは **ユーザー領域** や **プロジェクト領域** に直接置きます。

| 配置場所 | インストール方法 | 用途 |
|---|---|---|
| `~/.claude/plugins/cache/...` | `/plugin` で取得 | 公式・配布スキルをそのまま使う（自動更新） |
| `~/.claude/skills/<name>/` | フォルダを手動配置 | 個人用にカスタマイズ・自作 |
| `<project>/.claude/skills/<name>/` | フォルダを手動配置 | チームでGit共有 |

**使い分けの目安**

- **そのまま使う** → `/plugin`（キャッシュ場所は気にしない）
- **改造して使いたい** → キャッシュから `cp -r` でユーザー領域へコピーしてから編集
- **チームで共有したい** → プロジェクトの `.claude/skills/` に置いてGit管理


---

## 【ハンズオン】　Claude Codeでskillsをプロジェクトフォルダにコピーして使う

```Claude Code プロンプト
先ほどpluginでインストールしたinternal-commsをプロジェクトフォルダのskillsにコピーして
```

コピーされたskillsを探してみて下さい。

---

## スキルの構造まとめ

公式 internal-comms スキルから分かること:

1. **frontmatter** で「いつ使うか」をユースケース列挙で明確にする
2. **SKILL.md 本文** はルーティング（どの examples を開くか）に徹する
3. **詳細は examples/ に分割** して、必要な時だけ読ませる
4. **執筆ガイドライン**（文量・トーン・情報源・テンプレート）を自然言語で書けば、Claude はそれに従う
5. **公式スキルをベースに自社仕様にアレンジ** すれば、ゼロから作らなくていい

→ プログラミング不要。Markdown が書ければスキルは作れる

---

## 【ハンズオン】　skillsを使ってニュースレターを書いてみる

```プロンプト
新しい人事評価クラウドScaleをリリースしました。次のURLからサービス紹介用のニュースレター用ファイルを作成して下さい。
画像もそのまま参考画像として取得して埋め込んでください。
ショートコードを使用しないでUnicode絵文字を使用して下さい。
◆プレスリリース：https://prtimes.jp/main/html/rd/p/000000002.000179065.html
◆サービスページ：https://colorfulbox.co.jp/scale
```

---

## 何が起きているか

1. "ニュースレター"というキーワードから`Skill(example-skills:internal-comms)` の description 基準でスキル発火
2. `SKILL.md` 本文を読み込み
3. リファレンスにあるテンプレートファイル（examples/company-newsletter.md）を読み込み
4. テンプレート（examples/company-newsletter.md）に沿ったニュースレターを生成

---

## internal-comms スキルの中身を見てみる

ハンズオンで使った `internal-comms` スキルの実体は、たった3つの要素で構成されています。

```
.claude/skills/internal-comms/
├── SKILL.md          # スキルのエントリーポイント
├── examples/         # ドキュメント種別ごとのテンプレート
│   ├── 3p-updates.md
│   ├── company-newsletter.md
│   ├── faq-answers.md
│   └── general-comms.md
└── LICENSE.txt
```

---

## SKILL.md の中身（internal-comms 抜粋）

```markdown
---
name: internal-comms
description: A set of resources to help me write all kinds of internal communications...
  Claude should use this skill whenever asked to write some sort of internal
  communications (status reports, leadership updates, 3P updates, company
  newsletters, FAQs, incident reports, project updates, etc.).
---

## When to use this skill
- 3P updates / Company newsletters / FAQ responses / Status reports ...

## How to use this skill
1. Identify the communication type from the request
2. Load the appropriate guideline file from the `examples/` directory:
    - examples/company-newsletter.md - For company-wide newsletters
    - examples/3p-updates.md - For Progress/Plans/Problems team updates
    ...
3. Follow the specific instructions in that file
```

ポイント:
- **frontmatter** の `description` で「いつ使うか」をユースケース列挙で明示 → Claude が自動発火の判断に使う
- **本文** は「どの examples ファイルを開けばよいか」のルーティングに徹している
- 詳細な書き方ルール（文量・トーン・テンプレート）は `examples/*.md` 側に分割

---

## スキルの動作（internal-comms の場合）

スキルは **段階的な情報開示 (Progressive Disclosure)** で動きます。今回のニュースレター作成では、こう動いていました。

1. **起動時**: 全スキルの `name` / `description` だけを確認（軽量）
2. **発火判定**: 「ニュースレター」というキーワードが `internal-comms` の description にマッチ → スキル本文をロード
3. **ルーティング**: `SKILL.md` の指示に従い、`examples/company-newsletter.md` だけを読み込み
4. **生成**: そのテンプレートに沿ってニュースレターを出力

→ `3p-updates.md` や `faq-answers.md` は **読み込まれない**。必要なときに必要なものだけ開くことで、コンテキストウィンドウを圧迫しません。

---

## 補足: スキルの置き場所

ユーザースコープとプロジェクトスコープがあります。

| スコープ | パス |
|---|---|
| ユーザー | `~/.claude/skills/<skill-name>/SKILL.md` |
| プロジェクト | `.claude/skills/<skill-name>/SKILL.md` |


---

## スラッシュコマンド と スキル

Claude Codeにはスラッシュコマンドという仕組みがあります。

スラッシュコマンドとは、スキルなどを自動発火させるのをCladue Code + キーワードに任せるさせるのではなく、直接指定して発火（起動）させる仕組みです。

```プロンプト
/
```

"/"と打ってみて下さい。
そして、先ほどコピーした"/internal-comms"というスラッシュコマンド（スキル）が存在するか？確認してみて下さい。

---

## 補足: スラッシュコマンドとの関係

もともと Claude Code には `/command` : スラッシュコマンドがありました。 `.claude/commands/<name>.md` に Markdown で指示を書く仕組みです。

**v2.1.3 で Slash Commands は Skills に統合されました。**

> **カスタムスラッシュコマンドはスキルにマージされました。** `.claude/commands/review.md` のファイルと `.claude/skills/review/SKILL.md` のスキルの両方が `/review` を作成し、同じように機能します。既存の `.claude/commands/` ファイルは引き続き機能します。

出典: https://code.claude.com/docs/ja/skills

---

## 補足：　commands/ と skills/ の違い

内部的には同じシステムだが、Skills にしかない機能があります。

|  | `.claude/commands/` | `.claude/skills/` |
|---|---|---|
| `/name` で手動呼び出し | ✅ | ✅ |
| Claude が自動判断で発動 | ✅ | ✅ |
| サポートファイル（複数ファイル） | ❌（単一ファイル） | ✅（同ディレクトリに配置可能） |
| Plugin に含められるか | ❌ | ✅ |

→ **今後は `.claude/skills/` を使うのが推奨**

出典: Claude Code の拡張機能を整理する - Zenn

---

## スキル活用のTips

スキルを作る・育てるときに役立つポイントを紹介します。

---

## SKILL.md は 500 行以下に保つ

> 💡 `SKILL.md` を 500 行以下に保ちます。詳細なリファレンス資料を別のファイルに移動します。

出典: https://code.claude.com/docs/ja/skills

SKILL.md はスキルの **エントリーポイント** です。概要やナビゲーションとしましょう。詳細は `references/` 等に分割します。

```
my-skill/
├── SKILL.md          # 概要 + ナビゲーション
├── references/       # 詳細リファレンス
├── examples.md       # 使用例
└── scripts/
    └── helper.sh
```

---

## `context: fork` でサブエージェント実行

frontmatter に `context: fork` を付けると、スキルがサブエージェントとして隔離実行されます。

メインの会話コンテキストを汚しません。

---

## context:fork 活用例

```
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:
1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

- 出典: code.claude - スキルをサブエージェントで実行する

---

## Tips: サブエージェントとは？

特定のタスクを処理する特化した AI アシスタント。**独自のコンテキストウィンドウ** で実行され、メインの会話履歴にはアクセスしない。完了後、結果だけがメイン会話に返される。

— Sub-agents - Claude Code 公式ドキュメント

---

## スキルが発火しないとき

`description` を見直しましょう。Claude が「いつ使うべきか」を判断できる記述になっていますか？

それでもうまく行かない場合、最終手段は **手動実行** です。

- `/skill-name` でスラッシュコマンドとして直接実行
- 例: `/fix-issue 123`

---

## セキュリティに注意

スキルの入手元は信頼できるソースに限定しましょう。

- **自作** のスキル
- **社内** で管理・レビューされたスキル
- 信頼できる公開スキル（例: anthropics/claude-code-skills）

第三者のスキルを使う場合は **スキルの中身をすべて確認** してください（SKILL.md、スクリプト、リファレンス等）。判断できない場合は使わない。

- 参考: あなたの拾ってきた野良（マーケット）Skills、セキュリティトラブルを発生させていませんか？ - Zenn

---

## スキルを作るスキル

公式の skill-creator を導入すると、スキルの作成・改善を対話的に進められます。

```
> /skill-creator
● スキルクリエイターへようこそ！
  1. 新しいスキルを作成したい
  2. 既存のスキルを改善したい
  3. スキルのテスト・評価を実行したい
  4. スキルの説明文（トリガー）を最適化したい
```

---

## 他の仕組みとの使い分け

Skills は CLAUDE.md や MCP、サブエージェントとの使い分けは？

出典: *Claude Agent Skills Explained - YouTube*

---

## Skills vs CLAUDE.md

|  | Skills | CLAUDE.md |
|---|---|---|
| 役割 | 専門的なタスクの **「実行方法」** を教える | **プロジェクト固有の情報** をClaudeに伝える |
| スコープ | どのプロジェクトでも使えるポータブルな専門知識 | 特定リポジトリに紐づく（技術スタック、規約等） |

---

## Skills vs MCP Servers

|  | Skills | MCP Servers |
|---|---|---|
| 役割 | データを **「どう扱うべきか」** を教える | 外部データソースへの **「接続」** を提供 |
| 例 | クエリ最適化パターンを教える | GitHubやDBへのアクセスを可能にする |

---

## Skills vs Subagents

|  | Skills | Subagents |
|---|---|---|
| 性質 | **ポータブルな専門知識** | 固定化された業務ノウハウ・業務フローを定義する **特化型AIアシスタント** |
| 特徴 | どのエージェントでも使用可能 | 固定の役割（議事録作成者、UIレビュアー等） |

---

## Plugin とは

Skills・Agents・Hooks・MCP サーバーを **まとめて配布可能な形にパッケージ化** したものが Plugin です。

```
Plugin（配布パッケージ）
  ├── Skills（スキル）
  ├── Agents（カスタムエージェント）
  ├── Hooks（ライフサイクルフック）
  └── MCP サーバー
```

GitHub リポジトリ経由でインストール・バージョン管理ができます。

出典: Claude Code の拡張機能を整理する - Zenn

---

## Plugin のフォルダ構成

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       ← プラグイン定義
├── skills/
│   └── review/SKILL.md   ← スキル
├── agents/
│   └── debugger.md       ← カスタムエージェント
├── hooks/
│   └── hooks.json        ← ライフサイクルフック
└── .mcp.json              ← MCP サーバー設定
```

---

## 個人 Skill vs Plugin

|  | 個人 Skill (`~/.claude/skills/`) | Plugin |
|---|---|---|
| 管理場所 | ホームディレクトリに直置き | 独立したパッケージ |
| 共有 | 手動コピー | インストールで配布 |
| バージョン管理 | なし | あり |
| 含められるもの | Skill のみ | Skill + Agent + Hooks + MCP |

→ チームで共有したい設定は **Plugin として GitHub リポジトリで管理** するのがベストプラクティス


## ハンズオン２

サンプルの議事メモからクライアントとの共有用の議事録をを作成するスキルを作成してみよう。

## サンプルの議事メモ

以下のmarkdownファイルを使って下さい。

"Claude_code_Skills_実践トレーニング_ハンズオン2_サンプル議事メモ.md"
