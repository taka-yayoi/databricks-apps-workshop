# Claude Code × Databricks Apps ワークショップ

Claude Codeを使ってDatabricks Appsを開発するハンズオンワークショップの教材リポジトリです。

## 前提条件

以下がインストール・設定済みであることを確認してください:

- Node.js v18以上
- Python 3.10以上
- Claude Code CLI (`npm install -g @anthropic-ai/claude-code`)
- Databricks CLI v0.250.0以上
- Databricks認証設定済み (`~/.claude/settings.json`)

### 環境確認コマンド

```bash
# Claude Code
claude --version

# Databricks CLI
databricks --version

# Python
python --version

# 認証設定
cat ~/.claude/settings.json
```

## ハンズオン手順

**詳細な手順は [handson_guide.md](handson_guide.md) を参照してください。**

以下は概要です。

### Step 1: リポジトリのクローン

```bash
git clone https://github.com/taka-yayoi/databricks-apps-workshop.git
cd databricks-apps-workshop
```

### Step 2: CLAUDE.mdの確認

このリポジトリにはDatabricks Apps開発のベストプラクティスが記載された`CLAUDE.md`が含まれています。

```bash
cat CLAUDE.md
```

**確認ポイント**:
- app.yamlの書き方ルール(command、env形式)
- セキュリティ要件(SQLパラメータ化)
- Streamlitのベストプラクティス

### Step 3: Claude Codeの起動

```bash
claude
```

起動時に`CLAUDE.md`が自動的に読み込まれます。

### Step 4: Unity Catalog Browserの生成

Claude Codeに以下の指示を入力:

```
Unity Catalogブラウザーアプリを作成して。

要件:
- Unity Catalogのカタログ/スキーマ/テーブルをブラウズできる
- 選択したテーブルのデータをプレビューできる(最大100行)
- 数値カラムの基本統計量を表示
- Streamlitを使用
- Databricks SDKの自動認証を使用

ファイル構成:
- app.py: メインアプリケーション
- app.yaml: Databricks Apps設定
- requirements.txt: 依存関係(バージョン固定)

まず計画を立ててから、実装に進んで。
```

### Step 5: ローカル実行

```bash
databricks apps run-local
```

ブラウザで http://localhost:8000 を開いて動作確認。

### Step 6: デプロイ

```bash
# アプリ名は各自ユニークな名前に変更
databricks apps deploy uc-browser-<your-name>
```

## リポジトリ構成

```
databricks-apps-workshop/
├── CLAUDE.md                 # ベストプラクティス(Claude Code用)
├── README.md                 # このファイル
├── handson_guide.md          # 詳細ハンズオン手順
├── app.yaml.example          # 設定ファイルの雛形
├── requirements.txt.example  # 依存関係の雛形
└── docs/
    └── CLAUDE_reference.md   # フル版ベストプラクティス(参考資料)
```

## 参考資料

- [Claude Code ドキュメント](https://docs.anthropic.com/claude-code)
- [Databricks Apps ドキュメント](https://docs.databricks.com/apps)
- [Streamlit ドキュメント](https://docs.streamlit.io)

## トラブルシューティング

### Claude Codeが起動しない

```bash
# グローバルパスを確認
npm list -g @anthropic-ai/claude-code

# 再インストール
npm install -g @anthropic-ai/claude-code
```

### Databricks認証エラー

```bash
# 認証設定を確認
cat ~/.claude/settings.json

# Databricks CLIで認証テスト
databricks auth describe
```

### ローカル実行でエラー

```bash
# デバッグモードで実行
databricks apps run-local --debug

# 環境準備を含めて実行
databricks apps run-local --prepare-environment
```

## ライセンス

MIT License
