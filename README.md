# openAI-like-APIs

OllamaとLiteLLM、PostgreSQLを用いてOpenAI互換のAPIを提供するリポジトリです。  
プロキシ環境下（`http://g3.konicaminolta.jp:8080/`）でも動作します。

---

## 構成

| サービス | イメージ | 役割 |
|---------|---------|------|
| **ollama** | `ollama/ollama:latest` | ローカルLLM推論エンジン |
| **postgres** | `postgres:15-alpine` | LiteLLMのデータ永続化（ログ・ユーザー管理） |
| **litellm** | `ghcr.io/berriai/litellm:main-latest` | OpenAI互換プロキシ（Ollama へのリクエストをルーティング） |

---

## セットアップ

### 1. 環境変数の設定

```bash
cp .env.example .env
```

`.env` を編集してプロキシ設定とシークレットキーを確認・変更してください。

```dotenv
HTTP_PROXY=http://g3.konicaminolta.jp:8080/
HTTPS_PROXY=http://g3.konicaminolta.jp:8080/
NO_PROXY=localhost,127.0.0.1,ollama,postgres,litellm

# 強力なランダムキーを生成して設定する (例: openssl rand -hex 32)
LITELLM_MASTER_KEY=sk-REPLACE_WITH_RANDOM_KEY

# 強力なランダムパスワードを設定する (例: openssl rand -hex 16)
POSTGRES_PASSWORD=REPLACE_WITH_STRONG_PASSWORD
```

### 2. Docker Compose でビルド・起動

```bash
docker compose up -d
```

### 3. Ollama モデルのダウンロード

コンテナ起動後、利用したいモデルを pull します。

```bash
# 例: Llama 3
docker exec ollama ollama pull llama3

# 例: Mistral
docker exec ollama ollama pull mistral
```

> **プロキシ環境の場合**: モデルのダウンロードには外部への通信が必要です。  
> `.env` の `HTTP_PROXY` / `HTTPS_PROXY` が正しく設定されていれば自動的にプロキシを使用します。

---

## 使い方

LiteLLM は **ポート 4000** でOpenAI互換のエンドポイントを公開します。

### Chat Completions

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-change-me" \
  -d '{
    "model": "ollama/llama3",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### モデル一覧

```bash
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer sk-change-me"
```

### OpenAI Python クライアントからの利用

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-change-me",
    base_url="http://localhost:4000/v1",
)

response = client.chat.completions.create(
    model="ollama/llama3",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(response.choices[0].message.content)
```

---

## モデルの追加

`litellm_config.yaml` の `model_list` に追加してください。

```yaml
model_list:
  - model_name: ollama/llama3
    litellm_params:
      model: ollama/llama3
      api_base: http://ollama:11434
```

---

## Python ライブラリ（requirements.txt）

```
litellm[proxy]  # LiteLLMプロキシ本体
psycopg2-binary # PostgreSQL接続ドライバ
prisma          # LiteLLMが使用するORMクライアント
```

開発環境で直接 LiteLLM を実行する場合：

```bash
pip install -r requirements.txt
```

---

## サービスの停止

```bash
docker compose down

# ボリューム（DBデータ・モデル）も含めて削除する場合
docker compose down -v
```
