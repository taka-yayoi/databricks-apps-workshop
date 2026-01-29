# Databricks Apps + Streamlit プロジェクト

## プロジェクト概要

Unity Catalogの`samples`カタログを可視化するStreamlitダッシュボードアプリケーション。

**対象カタログ**: `samples`(tpch, nyctaxi等)

**samplesカタログを使う理由**: 
- 全ユーザーに読み取り権限があるため、デプロイ後もService Principal権限で問題なくアクセス可能
- 特別な権限設定が不要

## 技術スタック

- Python 3.11
- Streamlit
- Databricks SDK (databricks-sdk)
- Databricks SQL Connector (databricks-sql-connector)

**重要: Databricks AppsではSparkSessionは使用不可**

Databricks Appsはコンテナベースの軽量ランタイムで、Sparkドライバー/JVMが含まれていません。

```python
# ❌ 絶対にやってはいけない - JAVA_GATEWAY_EXITEDエラーになる
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("MyApp").getOrCreate()

# ✅ 代わりにDatabricks SQL Connectorを使用
from databricks import sql
from databricks.sdk.core import Config
```

データアクセスには必ずSQL Warehouse経由の`databricks-sql-connector`を使用してください。

## 開発コマンド

```bash
# ローカル実行(初回は--prepare-environmentで仮想環境を自動作成)
databricks apps run-local --prepare-environment

# 環境変数を追加してローカル実行(valueFromはローカルで解決できないため--envで渡す)
databricks apps run-local --prepare-environment --env DATABRICKS_WAREHOUSE_ID=xxx

# ログ確認
databricks apps logs <app-name>
```

**重要**: 
- `--prepare-environment`を付けないと、streamlit等の依存関係がインストールされていない状態で実行され、`executable file not found`エラーになる
- app.yamlで`valueFrom`を使用している環境変数は、ローカル実行時に`--env`フラグで明示的に渡す必要がある

## デプロイ(Databricks Asset Bundles)

デプロイにはDatabricks Asset Bundlesを使用する。リソース設定(SQL Warehouse等)もコードで管理できる。

### databricks.yml の例

```yaml
bundle:
  name: uc-browser

resources:
  apps:
    uc-browser:
      name: uc-browser-<your-name>
      source_code_path: .
      resources:
        - name: sql-warehouse
          sql_warehouse:
            id: <warehouse-id>
            permission: CAN_USE

variables:
  warehouse_id:
    description: SQL Warehouse ID
    default: <your-warehouse-id>
```

### デプロイコマンド

```bash
# デプロイ(初回はアプリ作成も含む)
databricks bundle deploy

# アプリの状態確認
databricks apps describe <app-name>

# アプリのURL確認
databricks apps get <app-name> --output json | jq -r '.url'
```

**Bundlesのメリット**:
- リソース設定(SQL Warehouse等)をコードで管理
- UIでの手動設定が不要
- 環境間(dev/staging/prod)の切り替えが容易

## app.yaml のルール

### command は必ずリスト形式(ポート/アドレス指定禁止)

```yaml
# ✅ 正しい
command:
  - streamlit
  - run
  - app.py

# ❌ 間違い: 文字列形式
command: "streamlit run app.py"

# ❌ 間違い: ポート/アドレス指定(Databricks Apps環境では自動設定される)
command:
  - streamlit
  - run
  - app.py
  - --server.port=8000      # 指定禁止
  - --server.address=0.0.0.0  # 指定禁止
```

**重要**: Databricks Apps環境では以下の環境変数が自動設定され、**オーバーライド禁止**:
- `STREAMLIT_SERVER_PORT`: `DATABRICKS_APP_PORT`に設定
- `STREAMLIT_SERVER_ADDRESS`: `0.0.0.0`に設定

ローカル実行時は`databricks apps run-local`がポート8000で自動起動するため、app.yamlでのポート指定は不要。

### env は name/value ペアのリスト

```yaml
# ✅ 正しい
env:
  - name: DATABRICKS_WAREHOUSE_ID
    valueFrom: sql-warehouse
  - name: LOG_LEVEL
    value: INFO

# ❌ 間違い
env:
  DATABRICKS_WAREHOUSE_ID: "xxx"
```

### valueFrom vs value の使い分け

| パターン | 用途 |
|----------|------|
| `valueFrom: sql-warehouse` | SQL Warehouse ID |
| `valueFrom: serving-endpoint-name` | Model Serving エンドポイント名 |
| `value: "固定値"` | 固定の設定値 |

**重要**: `valueFrom`を使用するには、Databricks Apps UIでリソースを設定する必要があります。
Compute > Apps > アプリ名 > Settings > Resources で SQL Warehouse等を追加してください。
リソース設定なしでは`valueFrom`が解決されず、アプリが動作しません。

## セキュリティ要件

### SQLクエリは必ずパラメータ化

```python
# ✅ 正しい
cursor.execute(
    "SELECT * FROM catalog.schema.table WHERE id = :id",
    {"id": user_input}
)

# ❌ 間違い（SQLインジェクションの危険）
cursor.execute(f"SELECT * FROM table WHERE id = {user_input}")
```

### 認証情報をハードコードしない

```python
# ✅ 正しい - SDKの自動認証を使用
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()

# ❌ 間違い
token = "dapi_xxxxxxxxxxxxx"
```

### databricks-sql-connectorの認証(重要)

**Databricks Apps環境での正しいパターン**:

```python
from databricks import sql
from databricks.sdk.core import Config
import os

@st.cache_resource
def get_sql_connection():
    cfg = Config()  # WorkspaceClientではなくConfigを使う
    return sql.connect(
        server_hostname=cfg.host,
        http_path=f"/sql/1.0/warehouses/{os.getenv('DATABRICKS_WAREHOUSE_ID')}",
        credentials_provider=lambda: cfg.authenticate,  # lambdaでラップ必須
    )
```

**よくある間違い**:

```python
# ❌ 間違い1: WorkspaceClient.config.authenticateを直接渡す
w = WorkspaceClient()
conn = sql.connect(
    server_hostname=w.config.host,
    credentials_provider=w.config.authenticate,  # エラー: 'dict' object is not callable
)

# ❌ 間違い2: lambdaなしで渡す
cfg = Config()
conn = sql.connect(
    credentials_provider=cfg.authenticate,  # エラー: 'NoneType' object is not callable
)

# ✅ 正しい: Configを使い、lambdaでラップする
cfg = Config()
conn = sql.connect(
    server_hostname=cfg.host,
    http_path=f"/sql/1.0/warehouses/{os.getenv('DATABRICKS_WAREHOUSE_ID')}",
    credentials_provider=lambda: cfg.authenticate,
)
```

**ポイント**:
- `WorkspaceClient`ではなく`databricks.sdk.core.Config`を使用
- `credentials_provider`には`lambda: cfg.authenticate`の形式で渡す(lambdaでラップ必須)
- `cfg.authenticate`は呼び出し時にヘッダー辞書を返すメソッド

## Unity Catalogアクセスパターン

### カタログ/スキーマ/テーブル一覧の取得

Unity Catalogのメタデータ取得には`WorkspaceClient`を使用します。

```python
from databricks.sdk import WorkspaceClient
import streamlit as st

@st.cache_resource
def get_workspace_client():
    return WorkspaceClient()

def get_catalogs():
    w = get_workspace_client()
    return [c.name for c in w.catalogs.list()]

def get_schemas(catalog_name):
    w = get_workspace_client()
    return [s.name for s in w.schemas.list(catalog_name=catalog_name)]

def get_tables(catalog_name, schema_name):
    w = get_workspace_client()
    return [t.name for t in w.tables.list(catalog_name=catalog_name, schema_name=schema_name)]
```

### テーブルデータの取得

テーブルのデータ取得には`databricks-sql-connector`を使用します。

```python
from databricks import sql
from databricks.sdk.core import Config
import os
import streamlit as st

@st.cache_resource
def get_sql_connection():
    cfg = Config()
    return sql.connect(
        server_hostname=cfg.host,
        http_path=f"/sql/1.0/warehouses/{os.getenv('DATABRICKS_WAREHOUSE_ID')}",
        credentials_provider=lambda: cfg.authenticate,
    )

def get_table_data(catalog, schema, table, limit=100):
    conn = get_sql_connection()
    with conn.cursor() as cursor:
        # パラメータ化クエリは識別子には使えないため、許可リストで検証
        query = f"SELECT * FROM `{catalog}`.`{schema}`.`{table}` LIMIT {limit}"
        cursor.execute(query)
        return cursor.fetchall_arrow().to_pandas()
```

**注意**: テーブル名などの識別子はSQLパラメータ化できないため、ユーザー入力をそのまま使う場合は許可リストで検証するか、選択式UIを使用してください。

## Streamlit のベストプラクティス

### st.set_page_config() は最初に呼び出す

```python
import streamlit as st

# 必ず最初に呼び出す
st.set_page_config(
    page_title="UC Browser",
    page_icon="📊",
    layout="wide"
)

# 他のインポートやコードはこの後
```

### 接続はキャッシュする

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.core import Config
from databricks import sql
import os

@st.cache_resource
def get_workspace_client():
    return WorkspaceClient()

@st.cache_resource
def get_sql_connection():
    cfg = Config()
    return sql.connect(
        server_hostname=cfg.host,
        http_path=f"/sql/1.0/warehouses/{os.getenv('DATABRICKS_WAREHOUSE_ID')}",
        credentials_provider=lambda: cfg.authenticate,
    )
```

## requirements.txt のルール

バージョンを必ず固定する:

```txt
streamlit==1.45.0
databricks-sdk==0.55.0
databricks-sql-connector==4.0.0
pandas==2.2.3
numpy>=1.26.0,<2.0.0
```

## ディレクトリ構造

```
project/
├── CLAUDE.md           # このファイル
├── databricks.yml      # Databricks Asset Bundles設定(デプロイ時に生成)
├── app.yaml            # Databricks Apps設定
├── app.py              # メインアプリケーション
├── requirements.txt    # 依存関係
└── README.md           # プロジェクト説明
```

## よくあるエラーと対処法

| エラー | 原因 | 対処法 |
|-------|------|--------|
| `JAVA_GATEWAY_EXITED` | SparkSessionを使用しようとした | Databricks AppsではSparkは使用不可。SQL Connectorを使用 |
| `streamlit: executable file not found` | 依存関係未インストール | `--prepare-environment`を付けて実行 |
| `valueFrom property and can't be resolved locally` | valueFromはローカルで解決不可 | `--env VAR_NAME=value`で環境変数を渡す |
| `'dict' object is not callable` | SQL Connectorの認証設定ミス | `credentials_provider=lambda: cfg.authenticate`の形式で渡す |
| `'NoneType' object is not callable` | SQL Connectorの認証設定ミス | `Config()`を使い、lambdaでラップする |
| `YAML parse error` | app.yamlの構文エラー | command/envの形式を確認 |
| `ModuleNotFoundError` | 依存関係不足 | requirements.txtを確認 |
| `Connection refused` (ローカル) | アプリが起動していない | `databricks apps run-local`を実行 |
| `401 Unauthorized` | 認証エラー | SDK自動認証の設定を確認 |
| `App deployment failed` (ファイル関連) | 10MB超のファイルがある | 大きなファイルを除外、.gitignoreを確認 |
| アプリが起動しない(デプロイ後) | app.yamlでポート/アドレス指定 | `--server.port`と`--server.address`を削除 |

## Databricks Apps の制限事項

| 制限 | 詳細 |
|------|------|
| **SparkSession使用不可** | コンテナにSparkランタイムなし。SQL Connector必須 |
| **ファイルサイズ** | 1ファイルあたり10MB以下 |
| **認証** | Databricksユーザー認証必須(匿名アクセス不可) |
| **ポート/アドレス指定禁止** | `STREAMLIT_SERVER_PORT`と`STREAMLIT_SERVER_ADDRESS`は自動設定。app.yamlでオーバーライド禁止 |
| **接続タイムアウト** | SQL Warehouseがアイドル停止すると接続が切れる可能性あり |
