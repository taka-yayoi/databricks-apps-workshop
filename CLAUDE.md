# Databricks Apps + Streamlit プロジェクト

## プロジェクト概要

Unity Catalogのデータを可視化するStreamlitダッシュボードアプリケーション。

## 技術スタック

- Python 3.11
- Streamlit
- Databricks SDK (databricks-sdk)
- Databricks SQL Connector (databricks-sql-connector)

## 開発コマンド

```bash
# ローカル実行
databricks apps run-local

# デプロイ
databricks apps deploy <app-name>

# ログ確認
databricks apps logs <app-name>
```

## app.yaml のルール

### command は必ずリスト形式

```yaml
# ✅ 正しい
command:
  - streamlit
  - run
  - app.py
  - --server.port=8000
  - --server.address=0.0.0.0

# ❌ 間違い
command: "streamlit run app.py"
```

### env は name/value ペアのリスト

```yaml
# ✅ 正しい
env:
  - name: DATABRICKS_WAREHOUSE_ID
    valueFrom: sql-warehouse-id
  - name: LOG_LEVEL
    value: INFO

# ❌ 間違い
env:
  DATABRICKS_WAREHOUSE_ID: "xxx"
```

### valueFrom vs value の使い分け

| パターン | 用途 |
|----------|------|
| `valueFrom: sql-warehouse-id` | SQL Warehouse ID |
| `valueFrom: serving-endpoint-name` | Model Serving エンドポイント名 |
| `value: "固定値"` | 固定の設定値 |

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
@st.cache_resource
def get_workspace_client():
    return WorkspaceClient()

@st.cache_resource
def get_sql_connection():
    return sql.connect(...)
```

## requirements.txt のルール

バージョンを必ず固定する:

```txt
streamlit==1.45.0
databricks-sdk==0.20.0
databricks-sql-connector==3.1.0
pandas==2.0.3
```

## ディレクトリ構造

```
project/
├── CLAUDE.md           # このファイル
├── app.yaml            # Databricks Apps設定
├── app.py              # メインアプリケーション
├── requirements.txt    # 依存関係
└── README.md           # プロジェクト説明
```

## よくあるエラーと対処法

| エラー | 原因 | 対処法 |
|-------|------|--------|
| `YAML parse error` | app.yamlの構文エラー | command/envの形式を確認 |
| `ModuleNotFoundError` | 依存関係不足 | requirements.txtを確認 |
| `Connection refused` | ポート不一致 | --server.port=8000を確認 |
| `401 Unauthorized` | 認証エラー | SDK自動認証の設定を確認 |
