# Claude Code × Databricks Apps ハンズオンガイド

## 概要

このハンズオンでは、Unity Catalogブラウザーアプリを開発しながら、Claude Codeの実践的な使い方を習得します。

**所要時間**: 60分

**ゴール**: 「Claude Codeと対話しながら開発する」体験を通じて、以下を習得する
- スラッシュコマンドの活用
- @ファイル参照
- 対話的なデバッグ
- 反復的な機能追加
- CLAUDE.md更新によるノウハウ蓄積

## 事前準備チェックリスト

ハンズオン開始前に以下を確認してください。

| 項目 | 確認コマンド | 期待される結果 |
|------|-------------|---------------|
| Claude Code | `claude --version` | バージョン番号が表示 |
| Databricks CLI | `databricks --version` | 0.250.0以上 |
| Databricks認証 | `databricks auth describe` | 認証情報が表示 |
| Python | `python --version` | 3.10以上 |
| Git | `git --version` | バージョン番号が表示 |

**SQL Warehouse ID**: ___________________(講師から提供)

## Phase 1: プロジェクトセットアップと基本生成(15分)

### Step 1-1: リポジトリのクローン(2分)

```bash
# ワークショップ用リポジトリをクローン
git clone https://github.com/taka-yayoi/databricks-apps-workshop.git
cd databricks-apps-workshop

# 構成確認
ls -la
```

**確認ポイント**: 以下のファイルが存在すること
- `CLAUDE.md` - ベストプラクティス
- `README.md` - 手順書
- `app.yaml.example` - 設定テンプレート

### Step 1-2: Claude Code起動とCLAUDE.md読み込み確認(3分)

```bash
# Claude Code起動
claude
```

**起動後、CLAUDE.mdが読み込まれているか確認**:

```
CLAUDE.mdの内容を要約して
```

**確認ポイント**:
- Claude CodeがCLAUDE.mdの内容(Databricks Apps、Streamlit、app.yamlのルールなど)を説明できること
- 説明できれば、CLAUDE.mdは正しく読み込まれている

```
/status
```

**確認ポイント**:
- 接続しているモデル名(databricks-claude-xxx)
- Thinking modeの状態

### Step 1-3: @参照とコンテキスト確認(2分)

```
@CLAUDE.md の内容からapp.yamlのルールだけ教えて
```

**学習ポイント**: `@ファイル名`で特定ファイルを明示的に参照できる

```
/context
```

**学習ポイント**: コンテキストウィンドウの使用状況を視覚的に確認できる

### Step 1-4: アプリの基本生成(8分)

以下のプロンプトを入力:

```
Unity Catalogブラウザーアプリを作成して。

要件:
- カタログ/スキーマ/テーブルをサイドバーで選択
- 選択したテーブルのデータをプレビュー(最大100行)
- Streamlit使用
- Databricks SDKの自動認証

ファイル構成:
- app.py
- app.yaml(CLAUDE.mdのルールに従う)
- requirements.txt(バージョン固定)

まず計画を説明してから、ファイルを作成して。
```

**観察ポイント**:
- Claude Codeが計画を説明するか
- CLAUDE.mdのルールが反映されているか(app.yamlの形式)
- 差分表示で変更内容を確認できるか

**差分が表示されたら**:
- `y`で承認、`n`で拒否、`e`で編集
- 今回は`y`で承認

**Phase 1完了チェック**:
- [ ] app.py が生成された
- [ ] app.yaml が正しい形式(commandがリスト、envがname/value)
- [ ] requirements.txt にバージョンが固定されている

## Phase 2: 意図的なエラーとデバッグ体験(15分)

このフェーズでは、**あえてエラーを発生させ、Claude Codeと対話して解決**します。

### Step 2-1: ローカル実行を試みる(3分)

```bash
# Claude Codeを一旦抜ける
/exit

# ローカル実行(--prepare-environmentで依存関係を自動インストール)
databricks apps run-local --prepare-environment
```

**予想されるエラー**: SQL Warehouse IDが設定されていない

```
Error: DATABRICKS_WAREHOUSE_ID is not set
```

### Step 2-2: Claude Codeに戻ってエラーを相談(5分)

```bash
claude
```

**エラーメッセージを貼り付けて相談**:

```
ローカル実行で以下のエラーが出た。原因と対処法を教えて。

Error: DATABRICKS_WAREHOUSE_ID is not set
```

**観察ポイント**:
- Claude Codeがエラー原因を特定できるか
- 具体的な修正提案があるか

### Step 2-3: 環境変数の設定(2分)

Claude Codeの提案に従って修正。おそらく以下のような指示:

```
@app.yaml にSQL Warehouse IDを追加して。
Warehouse IDは環境変数から読み取るように。
IDは: <講師から提供されたID>
```

### Step 2-4: 再度ローカル実行(5分)

```bash
# 一旦抜ける
/exit

# 再実行
databricks apps run-local --prepare-environment
```

**ブラウザで確認**: http://localhost:8000

**新たなエラーが出た場合**:
Claude Codeに戻って相談する。これが「対話的デバッグ」の体験。

```bash
claude

# エラーを貼り付け
> 以下のエラーが出た。修正して。
> <エラーメッセージ>
```

**Phase 2完了チェック**:
- [ ] ローカルでアプリが起動した
- [ ] カタログ一覧が表示された
- [ ] エラー→相談→修正のサイクルを体験した

## Phase 3: 反復的な機能追加(20分)

このフェーズでは、**小さな機能を複数回追加**し、反復開発のリズムを体験します。

### Step 3-1: 機能追加(1) - 統計表示(7分)

```bash
claude
```

```
@app.py に以下の機能を追加して。

- 数値カラムの基本統計(平均、中央値、標準偏差)を表示
- 表示/非表示を切り替えるチェックボックスを追加

差分を見せて。
```

**差分確認のポイント**:
- 変更箇所がハイライトされる
- 追加行数、削除行数を確認
- `y`で承認

**ローカルで確認**:
```bash
/exit
databricks apps run-local --prepare-environment
```

### Step 3-2: 機能追加(2) - グラフ表示(7分)

```bash
claude
```

```
@app.py に数値カラムのヒストグラムを追加して。

- カラム選択のセレクトボックス
- Plotlyでヒストグラム表示
- requirements.txtにplotlyを追加

必要な変更をすべて行って。
```

**学習ポイント**: 複数ファイルの同時変更

**確認**(requirements.txtが変更されたので再度--prepare-environment):
```bash
/exit
databricks apps run-local --prepare-environment
```

### Step 3-3: コンテキスト管理(3分)

```bash
claude
```

ここで**コンテキスト使用量を確認**:

```
/context
```

**観察ポイント**:
- コンテキストがどの程度消費されているか
- 長い開発セッションでは/clearが必要になる

もしコンテキストが50%以上使われていたら:

```
/clear
```

**注意**: /clearすると会話履歴は消えるが、CLAUDE.mdは再読み込みされる

### Step 3-4: 機能追加(3) - エラーハンドリング強化(3分)

```bash
# /clearした場合は再起動
claude
```

```
@app.py のエラーハンドリングを改善して。

- テーブルへのアクセス権限がない場合のエラーメッセージ
- SQL実行タイムアウトの処理
- ユーザーフレンドリーなエラー表示

CLAUDE.mdのセキュリティ要件も確認して対応して。
```

**Phase 3完了チェック**:
- [ ] 統計表示機能が動作する
- [ ] グラフ表示機能が動作する
- [ ] 3回の反復サイクル(指示→生成→確認)を体験した
- [ ] /contextでコンテキスト管理を意識した

## Phase 4: CLAUDE.md更新(5分)

このフェーズでは、**学んだことをCLAUDE.mdに追記**します。

### Step 4-1: 学んだことを追記

```
@CLAUDE.md に以下のセクションを追加して。

## 今回学んだこと

### SQL Warehouse関連
- DATABRICKS_WAREHOUSE_IDは環境変数で設定が必要
- statement_execution APIはwarehous_idが必須パラメータ

### Streamlit関連
- 重い処理には@st.cache_resourceを使用
- セレクトボックスの連鎖(カタログ→スキーマ→テーブル)パターン

追記の差分を見せて。
```

**学習ポイント**: CLAUDE.mdは「生きたドキュメント」。開発で学んだことを追記することで、次回以降の開発効率が上がる。

### Step 4-2: 追記内容の確認

```
@CLAUDE.md を表示して。追記された部分を確認したい。
```

**Phase 4完了チェック**:
- [ ] CLAUDE.mdに新しいセクションが追加された
- [ ] CLAUDE.md更新の意義を理解した

## Phase 5: デプロイ(5分)

### Step 5-1: デプロイ前の最終確認

```bash
# Claude Codeを終了
/exit

# ファイル一覧確認
ls -la

# app.yamlの内容確認
cat app.yaml
```

### Step 5-2: デプロイ実行

```bash
# アプリをデプロイ
databricks apps deploy uc-browser-<your-name>

# デプロイ状況確認
databricks apps describe uc-browser-<your-name>
```

### Step 5-3: 動作確認

デプロイ完了後、表示されたURLにアクセス:
```
https://<workspace>.cloud.databricks.com/apps/uc-browser-<your-name>
```

**Phase 5完了チェック**:
- [ ] デプロイが成功した
- [ ] ブラウザでアプリにアクセスできた
- [ ] ローカルと同じ機能が動作した

## トラブルシューティング

### よくあるエラーと対処

| エラー | 原因 | 対処 |
|--------|------|------|
| `streamlit: executable file not found` | 依存関係未インストール | `--prepare-environment`を付けて実行 |
| `ModuleNotFoundError: streamlit` | 依存関係未インストール | `--prepare-environment`を付けて実行 |
| CLAUDE.mdの内容を説明できない | 別ディレクトリで起動 | `cd databricks-apps-workshop`してから再起動 |
| `YAML parse error` | app.yamlの構文エラー | Claude Codeに「@app.yaml の構文を検証して」 |
| `Connection refused` | ポート競合 | 別のターミナルでアプリが動いていないか確認 |
| `Permission denied` on catalog | Unity Catalog権限不足 | 講師に相談、別のカタログを使用 |
| `Warehouse not found` | Warehouse ID間違い | IDを再確認 |
| コンテキスト超過 | 会話が長すぎ | `/clear`でリセット |

### Claude Codeがうまく動かないとき

1. `/status`で接続状態を確認
2. `/context`でコンテキスト使用量を確認
3. 問題が続く場合は`/exit`して再起動

## 発展課題(時間が余った場合)

### 課題1: 検索機能の追加

```
テーブル名で検索できる機能を追加して。
サイドバーにテキスト入力を追加し、部分一致で絞り込み。
```

### 課題2: お気に入り機能

```
よく使うテーブルを「お気に入り」として保存できる機能を追加して。
Streamlitのsession_stateを使用。
```

### 課題3: SQLクエリ実行機能

```
カスタムSQLクエリを入力して実行できる機能を追加して。
セキュリティに注意(CLAUDE.mdのルール参照)。
```

## 振り返り: 今日学んだClaude Codeスキル

| スキル | 使用場面 | コマンド/方法 |
|--------|---------|--------------|
| CLAUDE.md読み込み確認 | 起動直後 | 「CLAUDE.mdの内容を要約して」 |
| コンテキスト使用量確認 | セッション管理 | `/context` |
| ステータス確認 | 接続確認 | `/status` |
| ファイル参照 | 特定ファイルを指定 | `@ファイル名` |
| コンテキストクリア | 長いセッションのリセット | `/clear` |
| 差分確認 | 変更内容の確認 | 自動表示、y/n/eで応答 |
| 対話的デバッグ | エラー解決 | エラーを貼り付けて相談 |
| CLAUDE.md更新 | ノウハウ蓄積 | Claude Codeに追記を依頼 |

## 次のステップ

- **発展編ワークショップ**: Skills、MCP連携、MLflow Tracing
- **公式ドキュメント**: https://docs.anthropic.com/claude-code
- **Databricks Apps**: https://docs.databricks.com/apps
