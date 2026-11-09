# Metabase (Docker Compose)

Docker Compose で [Metabase](https://www.metabase.com/) を起動するための構成です。

- **Metabase**: BI ツール本体（`http://localhost:3000`）
- **PostgreSQL 17**: Metabase のアプリケーション DB（設定・質問・ダッシュボードの保存先）
- **Ollama 連携**: ホスト上の Ollama（OpenAI 互換 API）を Metabase の AI 機能から利用

## 必要なもの

- Docker / Docker Compose v2
- （AI 機能を使う場合）ホストで起動している [Ollama](https://ollama.com/)

## セットアップ

1. `.env` を作成します。

   ```sh
   cp .env.example .env
   ```

2. `.env` に値を設定します。

   ```sh
   # 例: ランダムな値を生成して設定
   echo "POSTGRES_PASSWORD=$(openssl rand -hex 24)" >  .env
   echo "MB_ENCRYPTION_SECRET_KEY=$(openssl rand -base64 32)" >> .env
   chmod 600 .env
   ```

   | 変数 | 必須 | 説明 |
   | --- | --- | --- |
   | `POSTGRES_PASSWORD` | ✓ | アプリケーション DB（PostgreSQL）のパスワード |
   | `MB_ENCRYPTION_SECRET_KEY` | ✓ | DB 接続情報などを暗号化するキー（16 文字以上） |
   | `METABASE_VERSION` | | Metabase イメージのタグ（既定: `v0.65.3.6`） |
   | `METABASE_PORT` | | ホスト側の公開ポート（既定: `3000`） |
   | `OLLAMA_BASE_URL` | | Ollama の API URL。末尾の `/v1` が必要（既定: `http://host.docker.internal:11434/v1`） |
   | `MB_LLM_ALLOWED_NETWORKS` | | LLM の接続先として許可するネットワーク（既定: `allow-private`） |

   > [!WARNING]
   > `MB_ENCRYPTION_SECRET_KEY` は一度設定したら変更しないでください。変更すると保存済みの接続情報を復号できなくなります。安全な場所にバックアップしておくことを推奨します。

3. 起動します。

   ```sh
   docker compose up -d
   ```

   初回起動には 1〜2 分かかります。状態は次で確認できます。

   ```sh
   docker compose ps          # metabase が healthy になれば準備完了
   docker compose logs -f metabase
   ```

4. ブラウザで <http://localhost:3000> を開き、初期設定（管理者アカウントの作成など）を行います。

## Ollama との連携

Metabase コンテナからは `host.docker.internal` 経由でホスト上の Ollama に接続します（Linux でも解決できるよう `extra_hosts` に `host-gateway` を設定済み）。

- Ollama がコンテナから到達できるアドレスで待ち受けている必要があります。`127.0.0.1` のみで待ち受けている場合は、`OLLAMA_HOST=0.0.0.0` などを設定して Ollama を起動してください。
- Ollama はプライベート IP 上にあるため、`MB_LLM_ALLOWED_NETWORKS=allow-private` を設定しています（既定値の `external-only` では接続できません）。
- 別ホストの Ollama を使う場合は `.env` に `OLLAMA_BASE_URL` を設定してください。

接続確認:

```sh
docker compose exec metabase curl -s http://host.docker.internal:11434/v1/models
```

## よく使うコマンド

```sh
docker compose up -d       # 起動
docker compose down        # 停止（データは保持）
docker compose restart metabase
docker compose logs -f     # ログ表示
```

### アップデート

`.env` の `METABASE_VERSION`（または `compose.yml` の既定値）を変更してから実行します。

```sh
docker compose pull
docker compose up -d
```

アップデート前にはバックアップを取得してください。

### バックアップ / リストア

アプリケーション DB は名前付きボリューム `metabase_postgres-data` に保存されています。

```sh
# バックアップ
docker compose exec -T postgres pg_dump -U metabase -d metabase -Fc > metabase-$(date +%Y%m%d).dump

# リストア
docker compose exec -T postgres pg_restore -U metabase -d metabase --clean --if-exists < metabase-YYYYMMDD.dump
```

> [!CAUTION]
> `docker compose down -v` はボリュームを削除するため、すべての設定・質問・ダッシュボードが失われます。

## 構成メモ

- Metabase のポートは `127.0.0.1` にのみ公開しています。外部から公開する場合はリバースプロキシ（HTTPS）経由にしてください。
- タイムゾーンは `Asia/Tokyo` に設定しています。
- `.env` は `.gitignore` 済みです。コミットしないでください。
