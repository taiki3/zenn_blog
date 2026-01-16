---
title: "Claude Code を使う中で考えていること"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "ai", "cli", "productivity"]
published: true
---

## 前提

Claude Pro MaxプランもしくはEnterpriseプレミアムシートを契約していること。

## tl;dr

- ultrathinkはガンガン使う
- git worktreeで複数エージェントを並行稼働
- CLIツールを活用してコーディング以外もAIに任せる
- テストを書かせる
- デザインはv0などで時間をかけて行う
- Claude Code on the Webを活用しよう

## 指示の出し方

2026.1.11現在、2-3スレッドをOpus4.5で回し続けてもレートリミットにはそうそう達しない。
ultrathinkもガンガン使って大丈夫。明らかに簡単な変更以外は手癖でultrathinkさせよう。

工夫としては、やりたいことをチャット系のAIに相談し、壁打ちを経てから指示文をまとめさせるのが有効。

## git worktreeで複数エージェントを並行稼働

Claude Codeを複数同時に走らせたいとき、同じディレクトリで作業させるとコンフリクトが起きる。`git worktree`を使えば、1つのリポジトリから複数の作業ディレクトリを作成でき、それぞれで別のClaude Codeを動かせる。

### セットアップ

```bash
# メインのリポジトリがあるとして
cd ~/projects/my-app

# worktreeを作成（ブランチごとに別ディレクトリを用意）
git worktree add ../my-app-feature-a feature-a
git worktree add ../my-app-feature-b feature-b
git worktree add ../my-app-bugfix bugfix-123
```

これで以下のような構成になる：

```
~/projects/
├── my-app/           # メイン（mainブランチ）
├── my-app-feature-a/ # feature-aブランチ
├── my-app-feature-b/ # feature-bブランチ
└── my-app-bugfix/    # bugfix-123ブランチ
```

### 並行作業の実行

各ディレクトリで別々のターミナルを開き、それぞれでClaude Codeを起動する。

```bash
# ターミナル1
cd ~/projects/my-app-feature-a && claude

# ターミナル2
cd ~/projects/my-app-feature-b && claude

# ターミナル3
cd ~/projects/my-app-bugfix && claude
```

これで3つのタスクを同時並行で進められる。

### 作業完了後のクリーンアップ

```bash
# worktreeの一覧確認
git worktree list

# 不要になったworktreeを削除
git worktree remove ../my-app-feature-a
```

### 注意点

- 各worktreeは独立した作業ディレクトリなので、`node_modules`などの依存関係は個別にインストールが必要
- 同じブランチを複数のworktreeでチェックアウトすることはできない
- `.git`は共有されるため、コミット履歴はリアルタイムで同期される

## CLIツール

コーディング以外の作業もClaudeに任せたいとき、CLIツールの導入が非常に有効。
Claude CodeはBashツールを通じてあらゆるCLIを実行できるため、適切なツールを用意しておくことで作業の幅が大きく広がる。

### DuckDB

データ処理系のコードを書くときに人間がExcelで開いたりPythonで可視化するところ、LLMにはDuckDBでデータを覗かせるのが良い。

**用途**
- CSV/Parquet/JSONなど各種データファイルの分析
- データの前処理・変換スクリプトの検証
- SQLによる集計・フィルタリング

**使用例**
```bash
# CSVファイルの内容を確認
duckdb -c "SELECT * FROM 'data.csv' LIMIT 10"

# 集計クエリ
duckdb -c "SELECT category, COUNT(*) FROM 'sales.csv' GROUP BY category"

# 複数ファイルの結合
duckdb -c "SELECT * FROM 'users.csv' u JOIN 'orders.csv' o ON u.id = o.user_id"
```

Claude Codeにデータの中身を確認させながらコードを書かせることで、実データに即した実装が可能になる。

### gh (GitHub CLI)

GitHub上の作業を自動化・効率化できる。issueもActionsも人間が見に行く必要がなくなる。

**用途**
- Issue/PRの作成・確認・クローズ
- GitHub Actionsのログ確認・再実行
- Code Rabbitなどのレビューコメントの確認
- リポジトリ設定・環境変数の管理

**使用例**
```bash
# Issueの一覧確認
gh issue list

# PRのレビューコメントを確認
gh pr view 123 --comments

# Actionsの実行状況確認
gh run list

# 失敗したワークフローのログ確認
gh run view 12345 --log-failed

# 環境変数の設定
gh secret set API_KEY --body "xxx"
```

「このIssue対応して」と指示するだけで、Issueの内容を読み取り、実装し、PRを作成するまでを一気通貫で行える。

### Supabase CLI

Supabaseを使ったプロジェクトでは、データベースのマイグレーションや確認作業をClaude Codeに任せられる。

**用途**
- マイグレーションファイルの作成・適用
- データベーススキーマの確認
- Edge Functionsのデプロイ
- ローカル開発環境の管理

**使用例**
```bash
# マイグレーション作成
supabase migration new add_users_table

# ローカルDBにマイグレーション適用
supabase db reset

# スキーマの差分確認
supabase db diff

# Edge Functionのデプロイ
supabase functions deploy my-function
```

### Vercel CLI

Vercelへのデプロイやプレビュー環境の管理ができる。

**用途**
- プレビュー/本番デプロイ
- 環境変数の管理
- ビルドログの確認
- プロジェクト設定の変更

**使用例**
```bash
# プレビューデプロイ
vercel

# 本番デプロイ
vercel --prod

# 環境変数の設定
vercel env add NEXT_PUBLIC_API_URL

# デプロイ一覧の確認
vercel list
```

### その他おすすめのCLIツール

| ツール | 用途 |
|--------|------|
| `aws` / `gcloud` / `az` | クラウドリソースの管理 |
| `docker` | コンテナの操作・確認 |
| `kubectl` | Kubernetes操作 |
| `jq` | JSONデータの加工・整形 |
| `curl` / `httpie` | API動作確認 |
| `ffmpeg` | メディアファイルの変換 |

使う技術スタックに合わせてCLIツールを用意しておくと、Claude Codeの活躍の幅が大きく広がる。

## テスト駆動開発

コードの肥大化はAIでも避けられず、その生産性ゆえ人間以上に加速する。その対応としてテストの充実は必須。

テストがあることで、Claude Codeは変更後に自らテストを実行し、壊れていれば修正するというサイクルを回せる。人間のレビュー負荷を下げるためにも、テストカバレッジを高めておこう。

### 指示の出し方

**新機能を実装するとき**
```
ユーザー登録機能を実装して。先にテストを書いてから実装してね。
テストが通ることを確認してからコミットして。
```

**バグ修正のとき**
```
このバグを再現するテストを先に書いて、そのテストが通るように修正して。
```

**リファクタリングのとき**
```
この関数をリファクタリングして。既存のテストが全部通ることを確認しながら進めて。
```

### 効果的なパターン

- **「テストが通るまでやって」**: 実装→テスト実行→失敗→修正のループを自動で回してくれる
- **「テストも一緒に書いて」**: 実装と同時にテストを書かせることで、テストなしのコードが生まれにくい
- **「カバレッジを確認して」**: `jest --coverage`などを実行させ、テストの網羅性を意識させる

### CLAUDE.mdへの記載

プロジェクトのCLAUDE.mdにテストコマンドを明記しておくと、Claude Codeが自発的にテストを実行するようになる。

```markdown
## テスト
- `npm test`: 全テスト実行
- `npm run test:watch`: ウォッチモード
- `npm run test:coverage`: カバレッジレポート
```

テストが整備されていれば、「この機能を実装して」という指示だけで、実装→テスト→修正のサイクルを自走してくれる。

## デザイン

人間が触れる部分は人間が引き続き実施すべき。v0やFigma Make、Replitなどのサービスを使い、ユーザーインタビューを行いながら初期デザインを決める。

UIデザインは下記のようなスキルを使うのも有効かも。

https://github.com/Koomook/claude-frontend-skills

## Claude Code on the Web

ドキュメント整備、プランニングなどの計画を立てさせるなどはどんなプロジェクトでも有効。開発環境にSSHできなくてもAIを働かせ続けることができる。
スマホから確認・指示できるのも便利。
