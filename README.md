# openAI-like-APIs

OllamaとLiteLLM、PostgreSQLを用いてOpenAI互換のAPIを提供するリポジトリです。  
プロキシ環境下（`http://g3.konicaminolta.jp:8080/`）でも動作します。

---

## 前提条件

| 要件 | 詳細 |
|------|------|
| **Docker** | Docker Desktop / Docker Engine（Compose Plugin 含む） |
| **NVIDIA GPU** | VRAM 14 GB 以上の GPU（`nvidia-container-toolkit` インストール済み） |
| **NVIDIA Container Toolkit** | `sudo apt install nvidia-container-toolkit` などでインストール |

> GPU を使用しない場合は `docker-compose.yml` の `deploy` セクションと `OLLAMA_NUM_GPU` / `OLLAMA_MAX_VRAM` を削除し、CPU モードで起動できます。

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

> **⚠️ 重要**: `POSTGRES_PASSWORD` と `LITELLM_MASTER_KEY` は必ずプレースホルダーから実際の値に変更してください。  
> `POSTGRES_PASSWORD` が空のままだと PostgreSQL コンテナが起動直後にクラッシュします。

### 2. NVIDIA Container Toolkit のインストール（GPU を使用する場合）

```bash
sudo apt install nvidia-container-toolkit
sudo systemctl restart docker
```

> NVIDIA GPU がない環境で起動する場合は、`docker-compose.yml` の `ollama` サービスの `deploy` セクション全体と `OLLAMA_NUM_GPU` / `OLLAMA_MAX_VRAM` 行を削除してから起動してください。

### 3. Docker Compose でビルド・起動

```bash
docker compose up -d
```

### 4. Ollama モデルのダウンロード

`ollama-init` コンテナが Ollama 起動後に自動でデフォルトの3モデルを pull します。手動操作は不要です。

ダウンロードの進捗はログで確認できます：

```bash
docker compose logs -f ollama-init
```

> **プロキシ環境の場合**: モデルのダウンロードには外部への通信が必要です。  
> `.env` の `HTTP_PROXY` / `HTTPS_PROXY` が正しく設定されていれば自動的にプロキシを使用します。

モデルを追加で pull したい場合：

```bash
docker exec ollama ollama pull <モデル名>
```

---

## 使い方

LiteLLM は **ポート 4000** でOpenAI互換のエンドポイントを公開します。

### Chat Completions

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-change-me" \
  -d '{
    "model": "ollama/gemma3:12b",
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
    model="ollama/gemma3:12b",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(response.choices[0].message.content)
```

---

## モデルの追加

`litellm_config.yaml` の `model_list` に追加してください。

```yaml
model_list:
  - model_name: ollama/gemma3:12b
    litellm_params:
      model: ollama/gemma3:12b
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

---

## トラブルシューティング

### コンテナが起動しない・エラーが見えない場合

`-d` オプションなしで起動すると、すべてのログがターミナルに直接表示されます：

```bash
docker compose up
```

特定コンテナのログだけ確認したい場合：

```bash
docker compose logs ollama
docker compose logs postgres
docker compose logs litellm
```

### `ollama` が即座に Error になる場合

```
✘ Container ollama  Error  0.0s
```

この場合、コンテナ生成そのものが失敗しています。原因は `nvidia-container-toolkit` が未インストールであることがほとんどです。

```bash
# エラー詳細を確認
docker inspect ollama 2>/dev/null || echo "コンテナが作成されていません"

# nvidia-container-toolkitのインストール確認
nvidia-container-cli --version
```

**GPU なし環境で起動したい場合** は `docker-compose.yml` の `ollama` サービスから以下を削除してください：

```yaml
# 削除するブロック
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
environment:
  OLLAMA_NUM_GPU: "-1"
  OLLAMA_MAX_VRAM: "15032385536"
```

### `litellm` が `ollama` に接続できない場合

```bash
# ollama が正常に動いているか確認
docker exec ollama ollama list

# litellm から ollama へ疎通確認
docker exec litellm curl -f http://ollama:11434/
```

### `APIConnectionError: OllamaException` が返ってくる場合

```
litellm.APIConnectionError: OllamaException - .
```

モデルがまだダウンロードされていない可能性があります。`ollama-init` コンテナの完了を確認してください：

```bash
docker compose logs ollama-init
```

`ollama-init` がまだ動いている（もしくは失敗している）場合は、手動で pull してください：

```bash
docker exec ollama ollama pull gemma3:12b
docker exec ollama ollama pull llama3.2:11b
docker exec ollama ollama pull phi4
```

pull 完了後、litellm へのリクエストが通るようになります（litellm の再起動は不要です）。
