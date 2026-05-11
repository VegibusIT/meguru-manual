# 08. 運用・トラブルシュートガイド

📍 [目次](README.md) ▶ 08. 運用ガイド

このページの読者：MEGURU を**実際に動かす担当者**（DevOps / インフラ）。

---

## 8.1 環境一覧

| 環境 | URL | 用途 |
|---|---|---|
| ローカル開発 | `http://localhost:3000` | 個人検証 |
| Fly.io shadow | `https://meguru-shadow.fly.dev`（仮）| シャドウ検証 |
| Fly.io production | 未デプロイ | 本番運用（Phase 5）|

ホスティング：Fly.io 東京リージョン（`nrt`）。

設定：[fly.shadow.toml](../../fly.shadow.toml) / [fly.toml](../../fly.toml)

---

## 8.2 起動・停止・再起動

### ローカル

```bash
# 全部起動
cd /data/m2labo/meguru
docker-compose up -d              # Postgres + Redis
DATABASE_URL='...' cargo run --bin meguru-api      # API
DATABASE_URL='...' cargo run --bin meguru-worker   # worker（必要時）

# 停止
docker-compose stop
# プロセス停止は Ctrl+C
```

### Fly.io

```bash
# デプロイ
fly deploy -c fly.shadow.toml

# ログ
fly logs -c fly.shadow.toml

# 再起動
fly apps restart meguru-shadow

# シェル
fly ssh console -c fly.shadow.toml
```

---

## 8.3 監視ポイント

### Health Check

```bash
curl -fsS http://localhost:3000/health || echo "DOWN"
```

### Liveness / Readiness（K8s 風）

| Probe | 確認内容 |
|---|---|
| Liveness | `/health` が `{"status":"ok"}` |
| Readiness | + DB に SELECT 1 が通る |
| Startup | + マイグレーション完了 |

### メトリクス（Phase 2）

🟡 Prometheus エンドポイントは未実装。当面 Fly.io の組み込み metrics で代用。

### ログレベル

```bash
RUST_LOG='meguru_api=info'                       # 通常
RUST_LOG='meguru_api=debug,sqlx=warn'            # 詳細
RUST_LOG='meguru_bridge=trace'                   # bridge デバッグ
RUST_LOG='error'                                 # 最小限
```

---

## 8.4 バックアップ・リストア

### Postgres

```bash
# ダンプ
docker exec meguru_db_1 pg_dump -U meguru meguru > backup_$(date +%F).sql

# リストア
docker exec -i meguru_db_1 psql -U meguru -d meguru < backup_2026-05-12.sql
```

### Fly.io 上の DB

🟡 Fly Postgres を使う場合は Fly のスナップショット機能を併用。

```bash
fly postgres backup list -a meguru-db
fly postgres backup restore -a meguru-db --snapshot-id <id>
```

---

## 8.5 マイグレーション運用

新規マイグレーションを足すとき：

```bash
# 1. ファイルを作成
echo "ALTER TABLE ..." > migrations/004_xxx.sql

# 2. ローカルで適用テスト
cat migrations/004_xxx.sql | docker exec -i meguru_db_1 psql -U meguru -d meguru

# 3. shadow へデプロイ
fly deploy -c fly.shadow.toml
# release_command で自動適用される設定にしておく
```

> 🔴 **本番マイグレーションは必ず**:
> - ステージング（shadow）で先に流す
> - ロールバック SQL も書く
> - ダウンタイムが必要かどうか事前判断

---

## 8.6 トラブルシュート

### docker 関連

| 症状 | 確認・対処 |
|---|---|
| `docker-compose up -d` が `port 5432 already in use` | 他の Postgres が動いている。`sudo lsof -i :5432` で確認、`docker-compose down`、または `docker-compose.yml` の port を変える |
| `meguru_db_1` が即 Exit | ボリュームに壊れたデータ。`docker volume rm meguru_db_data` で再作成（**注意：データ消える**）|
| `psql: could not connect` | docker は動いているか `docker ps`、ポート公開されているか `docker port meguru_db_1` |

### API 関連

| 症状 | 確認・対処 |
|---|---|
| `cargo run` が `linker error` | `sudo apt install build-essential libssl-dev pkg-config` |
| API が `panic` で落ちる | RUST_BACKTRACE=1 で再実行。ログを GitHub Issue に貼る |
| 起動はするが /health が 503 | DB 接続失敗。`DATABASE_URL` を確認 |
| `EADDRINUSE: 0.0.0.0:3000` | 別プロセスがポート占有。`lsof -i :3000` |

### SQL 関連

| 症状 | 対処 |
|---|---|
| `permission denied for table shipments` | RLS ポリシー違反。`app.tenant_id` セッション変数が設定されているか確認 |
| `null value in column ... violates not-null constraint` | アプリ側のバリデーション漏れ。スキーマと照らす |
| `duplicate key value violates unique constraint` | 冪等性違反。`external_order_id` の重複チェック |

### bridge / probe 関連

→ [06_shadow_testing.md#トラブルシュート](06_shadow_testing.md#611-トラブルシュート) を参照。

---

## 8.7 緊急停止手順

### 全停止

```bash
# 1. API を Ctrl+C で止める
# 2. worker を Ctrl+C で止める
# 3. bridge を Ctrl+C で止める
# 4. SSH トンネルを切る
pkill -f 13307
# 5. Postgres を止める
cd /data/m2labo/meguru && docker-compose stop
```

### やさいバスへの影響を疑う場合

→ [06_shadow_testing.md#緊急停止手順](06_shadow_testing.md#610-緊急停止手順)

要点：
- `meguru_shadow@'10.1.1.%'` は SELECT のみ
- DROP USER で完全遮断、再作成可能

### Fly.io 上で異常

```bash
fly scale count 0 -a meguru-shadow   # 停止
fly logs -a meguru-shadow            # 原因調査
fly scale count 1 -a meguru-shadow   # 復旧
```

---

## 8.8 セキュリティ運用

### シークレットローテーション

| 対象 | 頻度 | 手順 |
|---|---|---|
| API キー | 半年 | 新キー発行 → 旧キー 30 日併存 → 削除 |
| JWT 署名鍵 | 1 年 | `JWT_SECRET` env を更新 → デプロイ |
| DB パスワード | 1 年 | DB ユーザー再作成 → Fly secret 更新 |
| bastion SSH 鍵 | 2 年 | 新鍵生成 → AWS console で公開鍵差し替え |
| `meguru_shadow` パスワード | 半年 | RDS で SET PASSWORD → ファイル更新 |

### 監査ログ

🟡 Phase 2。当面はアプリログ（`RUST_LOG=info`）を Fly.io 経由で確認。

### ペネトレーションテスト

[security-review](README.md) のスキルで自動レビューを定期実施推奨。

---

## 8.9 マニュアルのWeb公開手順

このマニュアルは **3 通り** の方法で公開可能。

### A. GitHub Pages（最速）

`VegibusIT/meguru-manual` リポジトリで Settings → Pages → Source = `main` / `/docs` を選ぶ。

→ `https://vegibusit.github.io/meguru-manual/` で公開。

### B. VitePress でリッチに

```bash
cd /data/m2labo/meguru/docs/manual
npm init -y
npm install -D vitepress
npx vitepress init
# .vitepress/config.ts でナビ設定
npx vitepress build
# 出力: .vitepress/dist
```

→ Fly.io 静的サイトとしてデプロイ、または `meguru-site` のサブパス。

### C. meguru-site に組み込む

`/data/m2labo/meguru-site/` に `manual/` パスを増設してマウント。既存の `index.html` から「マニュアル」ボタンで遷移。

---

## 8.10 オペレータ向け FAQ

**Q1. cargo build が遅い**
A. `cargo build --release` は初回 5〜10 分。`target/` をキャッシュ。CI では `actions/cache` を使う。

**Q2. ローカルで Redis を使ってないが必要？**
A. 現状は使っていない。docker-compose で起動はしているが API は接続していない。Phase 2 でセッション保管に使う予定。

**Q3. fly secrets と .env はどう使い分ける？**
A. `.env` はローカル開発のみ、`fly secrets` は Fly.io 環境のみ。`.env` を git にコミットしない。

**Q4. 複数テナントのデータを同時に見たい**
A. RLS で阻まれる。管理者用 SQL を直接叩く（DB ロール `meguru_admin` で）。

**Q5. ログを集約したい**
A. Fly.io ログを Datadog や Loki に転送可能。設定例は別途。

---

## 8.11 デプロイチェックリスト

新リリースを本番に出す前に：

- [ ] `cargo test` 全パス
- [ ] `cargo clippy --all -- -D warnings` 警告ゼロ
- [ ] [07_test_scenarios.md](07_test_scenarios.md) の Test 1〜11 をクリア
- [ ] マイグレーション SQL がロールバック可能か確認
- [ ] ステージング（shadow）で 24 時間稼働
- [ ] Diff Reporter で大きな乖離がない
- [ ] バックアップ取得
- [ ] Slack 通知が届くか確認
- [ ] ロールバック手順を共有

---

次：API の詳細仕様は [09_api_reference.md](09_api_reference.md)。
