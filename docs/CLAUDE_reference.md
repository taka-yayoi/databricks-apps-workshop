# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) がこのリポジトリのコードを扱う際のガイダンスを提供します。

## プロジェクト概要

Databricks Apps上でOBO(On-Behalf-Of)ユーザー認証を使用したマルチファイルStreamlitアプリを構築するためのテンプレートです。

**核心原則**: アプリはサインインユーザーとして実行され、Unity Catalogの行/列レベルのセキュリティポリシーを尊重します。

## クイックスタート

```bash
# 依存関係のインストール
pip install -r requirements.txt

# ローカル実行（環境変数の設定またはローカル認証用の修正が必要）
streamlit run app.py
```

## アーキテクチャ

### ファイル構成

```
project/
├── app.py              # メインエントリポイント、ページ設定、ホームページ
├── app.yaml            # 実行コマンド + 環境変数マッピング
├── requirements.txt    # Python依存関係（バージョン固定必須）
├── utils.py            # 共有ユーティリティ（get_env(), get_user_token(), render_sidebar()等）
├── pages/              # Streamlitが自動検出するページディレクトリ
│   └── 0_empty.py      # 新規ページ作成用テンプレート
└── CLAUDE.md           # このファイル
```

### 認証モデル

Databricks SDKはデフォルトの`Config()`を使用し、注入されたアプリ認証情報を取得します。

**重要**: このアプリは2つの認証モードを使用します：

| 用途 | 認証方式 |
|------|----------|
| SQLクエリ | `Config().authenticate`（アプリレベル認証情報、credentials_provider経由） |
| Genie API呼び出し | 標準の`WorkspaceClient()`（GenieがOBOを内部処理） |

### ユーザー認可

Databricks Apps UIで以下のスコープを有効化する必要があります：
- `sql`
- `dashboards.genie`
- `iam.current-user:read`

ユーザートークンは`st.context.headers['X-Forwarded-Access-Token']`から`get_user_token()`で取得します。

## app.yaml設定

### 環境変数パターン

**重要**: リソースIDは絶対にハードコードしない。すべてのリソースIDはapp.yamlの環境変数から取得します。

```yaml
command: ["streamlit", "run", "app.py"]
env:
  # === データアクセス ===
  - name: SQL_WAREHOUSE_ID
    valueFrom: sql-warehouse              # Databricksリソース → valueFrom
  - name: UNITY_CATALOG_TABLE
    value: "catalog.schema.table"         # 直接値 → value
  - name: UNITY_CATALOG_VOLUME
    value: "catalog.schema.volume"

  # === AI/ML ===
  - name: MODEL_SERVING_ENDPOINT
    valueFrom: model-endpoint
  - name: VECTOR_SEARCH_ENDPOINT
    valueFrom: vector-search-endpoint
  - name: GENIE_SPACE_ID
    valueFrom: genie-space

  # === BI ===
  - name: DASHBOARD_EMBED_URL
    value: "https://<workspace-host>/embed/dashboardsv3/<dashboard-id>"

  # === ワークフロー ===
  - name: JOB_ID
    valueFrom: workflow-job

  # === シークレット ===
  - name: SECRET_SCOPE
    value: "my-secret-scope"
  - name: SECRET_KEY
    value: "my-api-key"
```

**パターンルール**:
- `valueFrom`: Databricksリソース（リソースIDにバインド）
- `value`: 直接値（URL、テーブル名、スコープ名）

### リソース追加手順

1. Databricks Apps UIでリソースを追加（キーを設定、例: `my-resource`）
2. app.yamlに環境変数を追加
3. コード内で`get_env("MY_RESOURCE_ID")`で参照

## コーディング規約

### 環境変数アクセス

```python
from utils import get_env

# ✅ 正しい方法（フレンドリーなエラー表示 + st.stop()）
resource_id = get_env("MY_RESOURCE_ID")

# ❌ 避けるべき方法
resource_id = os.getenv("MY_RESOURCE_ID")  # エラー時にクラッシュ
```

### エラーハンドリング

```python
from utils import get_env
import logging

logger = logging.getLogger(__name__)

try:
    resource_id = get_env("MY_RESOURCE_ID")
    # ... 処理 ...
except Exception as e:
    logger.error(f"処理に失敗しました: {e}", exc_info=True)
    st.error(f"処理に失敗しました。詳細: {e}")
```

### Streamlitパターン

| パターン | 用途 |
|----------|------|
| `st.set_page_config()` | 各ファイルの最初のStreamlitコマンド（必須） |
| `st.logo()` | サイドバーロゴ（utils.pyのrender_sidebar()内で呼び出し） |
| `@st.cache_resource` | コネクションオブジェクト用（utils.pyのsql_conn()） |
| `st.session_state` | ページリロード間でのデータ永続化 |
| `st.chat_message()` + `st.chat_input()` | チャットUI構築 |
| `components.iframe()` | 外部コンテンツ埋め込み（ダッシュボード等） |

**禁止事項**:
- `st.chat_message()`のネストは禁止（Streamlitエラーになる）

### ページ作成

1. `pages/0_empty.py`をコピーして新規ページを作成
2. ファイル名のプレフィックス番号で表示順序を制御（例: `1_`, `2_`, `3_`）
3. 最初の実ページ作成後は`pages/0_empty.py`を削除

### セッション状態

- `st.session_state`はアプリ内の全ページで共有される
- 会話ID、ユーザー選択などの永続化に使用

### ユーザー情報

`st.context.headers`から抽出（utils.pyの`render_user_badge()`参照）

## セキュリティ要件

### SQLインジェクション対策（必須）

```python
# ❌ 絶対に禁止（インジェクション脆弱性）
def get_data(table_name, filter_value):
    query = f"SELECT * FROM {table_name} WHERE col = '{filter_value}'"
    cursor.execute(query)

# ✅ パラメータ化クエリを使用
def get_data(filter_value):
    query = "SELECT * FROM catalog.schema.table WHERE col = :value"
    cursor.execute(query, {"value": filter_value})

# ✅ テーブル名が動的な場合はホワイトリスト検証
ALLOWED_TABLES = ["catalog.schema.table1", "catalog.schema.table2"]

def get_data(table_name, filter_value):
    if table_name not in ALLOWED_TABLES:
        raise ValueError(f"許可されていないテーブル: {table_name}")
    query = f"SELECT * FROM {table_name} WHERE col = :value"
    cursor.execute(query, {"value": filter_value})
```

### 入力バリデーション

```python
import re

def validate_input(user_input: str, max_length: int = 100) -> str:
    """ユーザー入力のバリデーション"""
    if not user_input:
        raise ValueError("入力が空です")
    if len(user_input) > max_length:
        raise ValueError(f"入力が長すぎます（最大{max_length}文字）")
    # 危険な文字のサニタイズ
    sanitized = re.sub(r'[<>"\']', '', user_input)
    return sanitized.strip()
```

### シークレット管理

- 生のシークレット値を環境変数に直接公開しない
- app.yamlで`valueFrom`を使用
- シークレットは定期的にローテーション（特にチームメンバー変更時）

## 運用要件

### ログ出力（stdout/stderr必須）

Databricksはstdout/stderrからログをキャプチャします。ローカルファイルへのログ出力は避けてください。

```python
import logging
import sys

# app.pyの先頭で設定
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout)
    ]
)
logger = logging.getLogger(__name__)

# 使用例
logger.info("アプリが起動しました")
logger.warning("警告メッセージ")
logger.error("エラーが発生しました", exc_info=True)
```

### グレースフルシャットダウン（SIGTERM対応）

アプリはSIGTERMシグナル受信後15秒以内にシャットダウンする必要があります。

```python
import signal
import sys
import logging

logger = logging.getLogger(__name__)

def graceful_shutdown(signum, frame):
    """SIGTERM受信時のクリーンアップ処理"""
    logger.info("シャットダウンシグナルを受信しました。クリーンアップを開始します...")
    
    # コネクションのクローズ
    if 'connection' in st.session_state:
        try:
            st.session_state.connection.close()
            logger.info("データベース接続をクローズしました")
        except Exception as e:
            logger.error(f"接続クローズ中にエラー: {e}")
    
    logger.info("シャットダウン完了")
    sys.exit(0)

# app.pyの先頭でシグナルハンドラを登録
signal.signal(signal.SIGTERM, graceful_shutdown)
```

### 依存関係バージョン固定（必須）

requirements.txtでは必ず正確なバージョンを指定してください。

```txt
# ✅ 正しい例（バージョン固定）
streamlit==1.45.0
databricks-sdk==0.55.0
databricks-sql-connector==4.0.0
pandas==2.2.3
plotly==5.24.0

# ❌ 避けるべき例
streamlit
databricks-sdk>=0.50.0
pandas
```

### 起動時間の最適化

コンテナ起動時間を最小化するため、初期化ロジックを軽量に保ちます。

```python
# ❌ 避けるべき例（起動時に重い処理）
import pandas as pd
from databricks import sql

# グローバルスコープでの重い初期化
connection = sql.connect(...)  # 起動を遅延させる
large_data = pd.read_csv("huge_file.csv")  # 起動を遅延させる

# ✅ 正しい例（遅延ロード）
@st.cache_resource
def get_connection():
    """必要になったときだけコネクション確立"""
    from databricks import sql
    from databricks.sdk.core import Config
    
    cfg = Config()
    return sql.connect(
        server_hostname=cfg.host,
        http_path=f"/sql/1.0/warehouses/{get_env('SQL_WAREHOUSE_ID')}",
        credentials_provider=lambda: cfg.authenticate
    )

# 使用時に初めて接続
if st.button("データ取得"):
    conn = get_connection()
    # ...
```

### 例外ハンドリング

グローバル例外ハンドリングを実装し、スタックトレースや機密データの公開を防ぎます。

```python
def main():
    try:
        # メインアプリロジック
        run_app()
    except Exception as e:
        logger.error(f"予期しないエラー: {e}", exc_info=True)
        st.error("予期しないエラーが発生しました。管理者に連絡してください。")
        # スタックトレースはログには出力するが、UIには表示しない

if __name__ == "__main__":
    main()
```

## データアクセスパターン

### テーブルスキーマの確認（重要）

**テーブルスキーマ、カラム名、フィルタ値を推測せず、必ず実テーブルを確認してから実装すること。**

```python
# スキーマ確認用クエリ
def get_table_schema(table_name: str):
    query = f"DESCRIBE TABLE {table_name}"
    return execute_query(query)

# サンプルデータ確認
def preview_data(table_name: str, limit: int = 5):
    query = f"SELECT * FROM {table_name} LIMIT {limit}"
    return execute_query(query)
```

### Databricksネイティブ機能の使用

App Computeは**UIレンダリング用に最適化**されています。重い処理はオフロードしてください。

| 処理タイプ | 推奨サービス |
|------------|--------------|
| クエリ・データセット | Databricks SQL |
| バッチ処理 | Databricks Jobs (Lakeflow) |
| AI推論 | Model Serving |

## デプロイ

### CLIでのデプロイ

```bash
databricks apps deploy <app-name> --source-code-path /Workspace/<path>
```

### デプロイ前チェックリスト

- [ ] requirements.txtのバージョンが固定されている
- [ ] app.yamlの環境変数が正しく設定されている
- [ ] SQLクエリがパラメータ化されている
- [ ] ログ出力がstdout/stderrに設定されている
- [ ] シグナルハンドラが実装されている
- [ ] 入力バリデーションが実装されている

## 参考リソース

修正時は以下を参照してください：

- [Databricks Apps ベストプラクティス](https://docs.databricks.com/aws/ja/dev-tools/databricks-apps/best-practices)
- [Databricks Apps 認証パターン](https://docs.databricks.com/aws/ja/dev-tools/databricks-apps/app-authorization)
- [Genie Conversation API](https://docs.databricks.com/api/workspace/genie)
- [Streamlit ドキュメント](https://docs.streamlit.io/)

## cookbook

`cookbook/streamlit/`ディレクトリにはDatabricks Apps CookbookのStreamlit向けサンプルがgit submoduleとして含まれています。
