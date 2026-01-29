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

**SQL Warehouse ID**: Databricksワークスペースで確認し、**必ずメモしておいてください**
```bash
# Databricks CLIで確認
databricks warehouses list
# または Databricks UI > SQL Warehouses > 対象のWarehouse > 詳細からIDをコピー
```

**メモしておく情報**:
- SQL Warehouse ID(例: `abc123def456`)
- Databricksメールアドレス(ワークスペースパスに使用)

## ハンズオンの実行環境について

**VSCodeは不要です**。このハンズオンはターミナルのみで完結します。

ただし、以下のような開発スタイルも可能です(参考情報):
- 複数ターミナルを開いて並列作業(1つでClaude Code、1つでアプリ実行)
- VSCode + Databricks拡張機能での開発
- VSCode内蔵ターミナルからClaude Code実行

本ハンズオンでは**1つのターミナル**で順番に進めます。

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

### ⚠️ 重要: Claude Codeを起動するディレクトリ

**Claude Codeはカレントディレクトリの`CLAUDE.md`を自動的に読み込みます。**

```bash
# ✅ 正しい: プロジェクトルートで起動
cd databricks-apps-workshop
claude

# ❌ 間違い: 別のディレクトリで起動
cd ~
claude  # → CLAUDE.mdが読み込まれない！
```

**別のディレクトリを操作したい場合**:
```bash
# 方法1: --add-dirでディレクトリを追加
claude --add-dir ../other-project

# 方法2: @で外部ファイルを参照
@/path/to/other/file.py を見て
```

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

**`/context`の出力の読み方**:

```
Context: ████████░░░░░░░░░░░░░░░░░░ 32% used (64K/200K tokens)
                                    ↑        ↑
                            使用中のトークン数  最大トークン数

Files in context:
• CLAUDE.md (2.1K tokens)     ← 読み込まれているファイル
• app.py (1.5K tokens)
• app.yaml (0.3K tokens)

Conversation: 15.2K tokens    ← 会話履歴のトークン数
```

**目安**:
- 50%以下: 十分な余裕あり
- 50-80%: 注意(長い出力を求めると溢れる可能性)
- 80%以上: `/clear`を検討

### Step 1-4: アプリの基本生成(8分)

以下のプロンプトを入力:

```
Unity Catalogのsamplesカタログブラウザーアプリを作成して。

要件:
- samplesカタログのスキーマ/テーブルをサイドバーで選択
- 選択したテーブルのデータをプレビュー(最大100行)
- Streamlit使用
- Databricks SDKの自動認証

ファイル構成:
- app.py
- app.yaml(CLAUDE.mdのルールに従う)
- requirements.txt(バージョン固定)

まず計画を説明してから、ファイルを作成して。
```

**なぜsamplesカタログか**: `samples`カタログ(tpch, nyctaxi等)は全ユーザーに読み取り権限があります。デプロイ後もService Principal権限で問題なくアクセスできます。

**観察ポイント**:
- Claude Codeが計画を説明するか
- CLAUDE.mdのルールが反映されているか(app.yamlの形式)
- 差分表示で変更内容を確認できるか

**差分が表示されたら**:
- 1: Yes(この変更を承認)
- 2: Yes, allow all edits during this session(shift+Tab)
- 3: No(拒否)
- Esc: キャンセル
- Tab: 修正を依頼
- 今回は`1`(Yes)で承認

### 軌道修正のテクニック

Claude Codeが期待通りに動かない場合の対処法:

| 操作 | 方法 | 用途 |
|------|------|------|
| **中断** | `Esc` | 処理を止めて方向転換 |
| **巻き戻し** | `Esc` を2回、または `/rewind` | 履歴を戻って別のアプローチを試す |
| **やり直し** | `/clear` | コンテキストをリセットして新鮮な状態から |

**ベストプラクティス**: 同じ問題で2回以上修正しても解決しない場合は、`/clear`して最初からやり直す方が速い。その際、失敗から学んだことを含めた**より具体的なプロンプト**で再開する。

### (参考) シェルコマンドの実行方法

本ハンズオンでは手堅く`/exit`でClaude Codeを抜けてからCLIを実行しますが、実際には抜けずに実行する方法もあります:

| 方法 | 説明 |
|------|------|
| 直接入力 | `databricks apps describe my-app` と入力するとClaudeが認識して実行 |
| `!`プレフィックス | `! databricks apps describe my-app` で即座にシェルに渡して実行 |

`run-local`のようにフォアグラウンドで動き続けるコマンドは`/exit`が必要ですが、`deploy`や`describe`のようにすぐ完了するコマンドはClaude Code内から実行できます。

### 具体的なプロンプトの重要性

曖昧な指示は曖昧な結果を生みます。

| ❌ 悪い例 | ✅ 良い例 |
|----------|----------|
| `テストを追加して` | `app.pyのテストを追加して。Warehouse接続エラーのケースをカバー。モックは使わないで` |
| `エラーを直して` | `このエラーを見て。@app.py の42行目が原因だと思う。CLAUDE.mdのセキュリティルールに従って修正して` |
| `UIを改善して` | `@app.py のサイドバーにテーブル検索機能を追加して。既存のst.selectboxパターンに合わせて実装して` |

**具体的に指示すべきこと**:
- 参照すべきファイルやパターン(@で指定)
- エッジケースや制約条件
- 使うべき/使わないべきライブラリ
- CLAUDE.mdのどのルールに従うか

**Phase 1完了チェック**:
- [ ] app.py が生成された
- [ ] app.yaml が正しい形式(commandがリスト、envがname/value)
- [ ] requirements.txt にバージョンが固定されている

## Phase 2: 意図的なエラーとデバッグ体験(15分)

このフェーズでは、**あえてエラーを発生させ、Claude Codeと対話して解決**します。

### Step 2-1: ローカル実行を試みる(3分)

**`databricks apps run-local`コマンドとは**:
- Databricks Appsをローカル環境でテスト実行するコマンド
- 本番デプロイ前の動作確認に使用
- ポート8000でアプリを起動(http://localhost:8000)

**主要オプション**:
| オプション | 説明 |
|-----------|------|
| `--prepare-environment` | requirements.txtから依存関係を自動インストール |
| `--env KEY=VALUE` | 環境変数を渡す(複数指定可) |
| `--debug` | デバッグモードで詳細ログを表示 |

```bash
# Claude Codeを一旦抜ける
/exit

# ローカル実行
databricks apps run-local --prepare-environment
```

**予想されるエラー**: app.yamlの`valueFrom`がローカルで解決できない

```
Error: DATABRICKS_WAREHOUSE_ID defined in app.yaml with valueFrom property and can't be resolved locally. Please set DATABRICKS_WAREHOUSE_ID environment variable in your terminal or using --env flag
```

**エラーの意味**: app.yamlで`valueFrom: sql-warehouse`と書くと、Databricks Apps環境では自動的にWarehouse IDが注入されます。しかしローカル実行では解決できないため、`--env`フラグで明示的に渡す必要があります。

### Step 2-2: Claude Codeに戻ってエラーを相談(5分)

```bash
# -c オプションで前の会話を継続
claude -c
```

**エラーメッセージを貼り付けて相談**:

```
ローカル実行で以下のエラーが出た。原因と対処法を教えて。

Error: DATABRICKS_WAREHOUSE_ID defined in app.yaml with valueFrom property and can't be resolved locally. Please set DATABRICKS_WAREHOUSE_ID environment variable in your terminal or using --env flag
```

**観察ポイント**:
- Claude Codeがエラー原因を特定できるか
- 具体的な修正提案があるか

### Step 2-3: --envフラグで環境変数を渡す(2分)

Claude Codeの提案に従って修正。`--env`フラグでWarehouse IDを渡します:

```bash
/exit

# --envフラグでWarehouse IDを渡してローカル実行
databricks apps run-local --prepare-environment --env DATABRICKS_WAREHOUSE_ID=<あなたのWarehouse ID>
```

**学習ポイント**: app.yamlの`valueFrom`はDatabricks Apps環境で自動解決されるが、ローカルでは`--env`で明示的に渡す必要がある

### Step 2-4: 動作確認(5分)

**ブラウザで確認**: http://localhost:8000

**新たなエラーが出た場合**:
Claude Codeに戻って相談する。これが「対話的デバッグ」の体験。

```bash
# -c オプションで前の会話を継続
claude -c

# エラーを貼り付け
> 以下のエラーが出た。修正して。
> <エラーメッセージ>
```

**学習ポイント**: `claude -c`で直前のセッションを継続できる。コンテキストが引き継がれるため、再度説明する必要がない。

**Phase 2完了チェック**:
- [ ] ローカルでアプリが起動した
- [ ] カタログ一覧が表示された
- [ ] エラー→相談→修正のサイクルを体験した

## Phase 3: 反復的な機能追加(20分)

このフェーズでは、**小さな機能を複数回追加**し、反復開発のリズムを体験します。

### Step 3-1: 機能追加(1) - 統計表示(7分)

```bash
# 前の会話を継続
claude -c
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
- `1`(Yes)で承認

**ローカルで確認**:
```bash
/exit
databricks apps run-local --prepare-environment
```

### Step 3-2: 機能追加(2) - グラフ表示(7分)

```bash
claude -c
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
claude -c
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
# /clearした場合は新規セッション、しなかった場合は-cで継続
claude -c  # または claude
```

```
@app.py のエラーハンドリングを改善して。think hardして考えてから実装して。

考慮すべき点:
- テーブルへのアクセス権限がない場合のエラーメッセージ
- SQL実行タイムアウトの処理
- ユーザーフレンドリーなエラー表示

CLAUDE.mdのセキュリティ要件も確認して対応して。
```

**Extended Thinkingについて**:

Extended Thinkingのトグル方法は**Claude Codeのバージョンによって異なります**:

| バージョン | トグル方法 |
|-----------|-----------|
| v2.0.x | `Tab` |
| v2.1.x以降 | `meta+t`(macOS: `Option+T`、Windows/Linux: `Alt+T`) |

```
# どちらかを押す → "Thinking on" と表示される → 深い思考が有効化
```

**ヒント**: `?`を押すと現在のバージョンで使えるショートカットが表示される

**強調フレーズの効果**:
| フレーズ | 効果 |
|---------|------|
| `think` | 基本的な思考を促す |
| `think hard` | より深い分析を促す |
| `think harder` / `ultrathink` | 最も深い思考を促す |

**使い分けの目安**:
- 簡単な修正 → フレーズなし
- 設計判断 → `think hard`
- 複雑なアーキテクチャ → `ultrathink`

**ソース**: [Claude Code Common Workflows](https://docs.anthropic.com/en/docs/claude-code/common-workflows)
> "The way you prompt for thinking results in varying levels of thinking depth"

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
- statement_execution APIはwarehouse_idが必須パラメータ

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

ローカルで動作確認できたアプリをDatabricks Appsにデプロイします。

### Step 5-1: デプロイの実行

Claude Codeに以下のプロンプトを入力:

```
Databricks Appsにデプロイして。
- アプリ名: uc-browser-<your-name>
- SQL Warehouse ID: <your-warehouse-id>

Databricks Asset Bundlesを使って、リソース設定も含めてデプロイして。
```

**Claude Codeが実行すること**:
1. `databricks.yml`(Bundle設定ファイル)を生成
2. SQL Warehouseリソースの設定を含める
3. `databricks bundle deploy`を実行
4. デプロイ完了を確認

**補足**: Databricks Asset Bundlesはインフラ設定をコードで管理するツールです。詳細を知る必要はありません。Claude Codeが適切な設定を生成してくれます。

### Step 5-2: 動作確認

デプロイ完了後、Claude Codeが表示するURLにアクセス:
```
https://<workspace>.cloud.databricks.com/apps/uc-browser-<your-name>
```

または、以下のプロンプトでURLを確認:
```
デプロイしたアプリのURLを教えて
```

### Step 5-3: (オプション)手動デプロイの場合

Claude Codeを使わずに手動でデプロイする場合:

```bash
# Claude Codeを終了
/exit

# Bundle設定を確認
cat databricks.yml

# デプロイ
databricks bundle deploy

# アプリの状態確認
databricks apps describe uc-browser-<your-name>
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
| `valueFrom property and can't be resolved locally` | valueFromはローカルで解決不可 | `--env DATABRICKS_WAREHOUSE_ID=xxx`で渡す |
| CLAUDE.mdの内容を説明できない | 別ディレクトリで起動 | 下記「ディレクトリ問題」参照 |
| `YAML parse error` | app.yamlの構文エラー | Claude Codeに「@app.yaml の構文を検証して」 |
| `Connection refused` | ポート競合 | 別のターミナルでアプリが動いていないか確認 |
| `Permission denied` on catalog | Unity Catalog権限不足 | 講師に相談、別のカタログを使用 |
| `Warehouse not found` | Warehouse ID間違い | IDを再確認 |
| コンテキスト超過 | 会話が長すぎ | `/clear`でリセット |
| アプリが起動しない(デプロイ後) | app.yamlでポート指定 | `--server.port`と`--server.address`を削除 |

### ディレクトリ問題: CLAUDE.mdが読み込まれない

**症状**: Claude Codeが「CLAUDE.mdの内容を要約して」に答えられない

**原因**: Claude Codeをプロジェクトディレクトリ以外で起動している

**解決方法**:
```bash
# 1. 現在のセッションを終了
/exit

# 2. プロジェクトディレクトリに移動
cd databricks-apps-workshop

# 3. 再起動
claude
```

**確認方法**:
```bash
# 起動前にCLAUDE.mdの存在を確認
ls -la CLAUDE.md
```

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
| セッション継続 | /exit後に会話を継続 | `claude -c` |
| Extended Thinking | 複雑な計画や設計 | `meta+t`または`Tab`でトグル(バージョンによる)、`think hard`等のフレーズ |
| コンテキスト使用量確認 | セッション管理 | `/context` |
| ステータス確認 | 接続確認 | `/status` |
| ファイル参照 | 特定ファイルを指定 | `@ファイル名` |
| 外部ディレクトリ参照 | プロジェクト外のファイル | `claude --add-dir ../other` |
| コンテキストクリア | 長いセッションのリセット | `/clear` |
| **中断** | 処理を止めて方向転換 | `Esc` |
| **巻き戻し** | 別のアプローチを試す | `Esc`を2回、または`/rewind` |
| 差分確認 | 変更内容の確認 | 自動表示、1/2/3で応答、Tabで修正依頼 |
| 対話的デバッグ | エラー解決 | エラーを貼り付けて相談 |
| CLAUDE.md更新 | ノウハウ蓄積 | Claude Codeに追記を依頼、または`#`キー |

**重要な原則**(公式ベストプラクティスより):
- **反復が改善を生む**: 最初の出力が良くても、2-3回の反復で大幅に改善されることが多い
- **早めの軌道修正**: 間違った方向に進んでいると気づいたら、すぐにEscで中断
- **具体的なプロンプト**: 曖昧な指示は曖昧な結果を生む

## 次のステップ

- **発展編ワークショップ**: Skills、MCP連携、MLflow Tracing
- **公式ドキュメント**: https://docs.anthropic.com/en/docs/claude-code
- **公式ベストプラクティス**: https://www.anthropic.com/engineering/claude-code-best-practices
- **Databricks Apps**: https://docs.databricks.com/apps
