# 06. シャドウ検証ガイド（M2Labo 内部）

📍 [目次](README.md) ▶ 06. シャドウ検証ガイド

このページの読者：**M2Labo 内部** で MEGURU のシャドウ検証を回す担当者。

> ⚠️ シャドウ検証 = やさいバス本番 DB を **読み取り専用** でミラーし、MEGURU の予測と実績を毎日比較する仕組み。本番運用ではありません。

🔴 **絶対条件**：やさいバス本番 DB には **一切書き込まない**。`meguru_shadow` ユーザーには `GRANT SELECT` のみ付与しています。

🎥 **動画候補**：6.4 起動手順（SSH トンネル張りから probe まで、3〜5 分）

---

## 6.1 全体構成図

```mermaid
flowchart TB
    Y[(やさいバス本番<br/>Aurora MySQL<br/>vegibus_new)]
    Bastion[bastion EC2<br/>35.73.227.214]
    SSH[ローカル端末<br/>SSHトンネル :13307]
    BR[meguru-bridge<br/>1分ごと差分pull]
    API[meguru-api]
    DB[(meguru-shadow<br/>Postgres :5432)]
    DIFF[Diff Reporter<br/>毎晩比較]
    SL[Slack]

    Y -- TLS+RDS auth --> Bastion
    Bastion -- TLS --> SSH
    SSH -- mysql 13307 --> BR
    BR -- HTTP POST --> API
    API -- INSERT --> DB
    DIFF -- 比較 --> DB
    DIFF -- 比較 --> Y
    DIFF -- 通知 --> SL
```

書き戻し矢印が **どこにもない** ことを確認してください。

---

## 6.2 構成要素

| 名前 | 場所 | 役割 |
|---|---|---|
| やさいバス Aurora | AWS RDS / Aurora MySQL 5.7 互換 | やさいバス本番。**触らない**。 |
| bastion EC2 | `35.73.227.214` / `ubuntu@` | SSH 踏み台。鍵: [/data/m2labo/vegibus-201909.pem](../../../vegibus-201909.pem) |
| `meguru_shadow` ユーザー | RDS 上 / `@'10.1.1.%'` | read-only 専用 |
| meguru-shadow Postgres | ローカル docker / `meguru_db_1` | 検証データの置き場 |
| meguru-api | ローカル / port 3000 | REST API |
| meguru-bridge-probe | one-shot バイナリ | スモークテスト |
| meguru-bridge (daemon) | 🟡 開発中 | 常駐差分 sync |

---

## 6.3 認証情報の置き場所

🔴 **禁止事項**：パスワード・API キーを git にコミットしない／コマンド直書きしない（`MYSQL_PWD` env / .env / Fly secrets を使う）。

| 情報 | 保管場所 |
|---|---|
| `meguru_shadow` パスワード | `/data/m2labo/.meguru_shadow.pwd`（chmod 600）|
| bastion 秘密鍵 | `/data/m2labo/vegibus-201909.pem`（chmod 600）|
| meguru-shadow Postgres pass | `meguru_dev`（開発用ハードコード）|
| やさいバス admin | 別管理。ローテーション要件あり |

---

## 6.4 起動手順（毎回この順）

### Step 1: Postgres を起動

```bash
cd /data/m2labo/meguru
docker-compose up -d db
docker ps --filter name=meguru
```

📸 [assets/captures/05_ssh_tunnel.png](assets/captures/05_ssh_tunnel.png)

### Step 2: やさいバス RDS への SSH トンネル

```bash
ssh -i /data/m2labo/vegibus-201909.pem \
    -L 13307:vegibus-database.cluster-cc43iwcqgrea.ap-northeast-1.rds.amazonaws.com:3306 \
    -N ubuntu@35.73.227.214 &
```

確認：

```bash
nc -z localhost 13307 && echo OK
```

### Step 3: meguru-api を起動

```bash
cd /data/m2labo/meguru
DATABASE_URL='postgres://meguru:meguru_dev@localhost:5432/meguru' \
RUST_LOG='meguru_api=info' \
cargo run --bin meguru-api
```

別ターミナル：

```bash
curl -s http://localhost:3000/health
```

📸 [assets/captures/01_health_check.png](assets/captures/01_health_check.png)

### Step 4: probe（スモークテスト）

```bash
cd /data/m2labo/meguru
SHADOW_PWD=$(cat /data/m2labo/.meguru_shadow.pwd) \
DATABASE_URL='postgres://meguru:meguru_dev@localhost:5432/meguru' \
YASAIBUS_DATABASE_URL="mysql://meguru_shadow:${SHADOW_PWD}@127.0.0.1:13307/vegibus_new" \
PROBE_BUS_AREA_ID=32 \
PROBE_TENANT_ID='8d94ed36-f5d5-4584-8dfb-b9f46d679a41' \
MEGURU_API_BASE_URL='http://127.0.0.1:3000' \
MEGURU_API_KEY='dev-noauth' \
RUST_LOG=info \
cargo run --bin meguru-bridge-probe
```

📸 [assets/captures/02_probe_full_run.png](assets/captures/02_probe_full_run.png)

期待されるハイライト：

- `count=158` — やさいバス本番から 158 バス停を read
- `by_type={"both":143,"garage":5,"transit":5,"delivery":3,"collection":2} unmappable=0` — マッピング 100%
- `posted=10 failed=0` — MEGURU API への POST 全件成功
- `mapped_status=Cancelled` / `Delivered` — order_status 変換 OK
- `containers=["Largex1"]` — container_size 変換 OK

### Step 5: DB 上の着地確認

```bash
docker exec -i meguru_db_1 psql -U meguru -d meguru \
  -c "SELECT id, name, stop_type FROM stops WHERE tenant_id='8d94ed36-f5d5-4584-8dfb-b9f46d679a41' ORDER BY id LIMIT 10;"
```

📸 [assets/captures/03_meguru_db_stops.png](assets/captures/03_meguru_db_stops.png)

---

## 6.5 環境変数リファレンス

| 変数 | 用途 | 例 |
|---|---|---|
| `DATABASE_URL` | meguru-shadow Postgres DSN | `postgres://meguru:meguru_dev@localhost:5432/meguru` |
| `YASAIBUS_DATABASE_URL` | やさいバス Aurora DSN（SSH 経由）| `mysql://meguru_shadow:***@127.0.0.1:13307/vegibus_new` |
| `MEGURU_API_BASE_URL` | bridge → meguru-api の宛先 | `http://127.0.0.1:3000` |
| `MEGURU_API_KEY` | API キー | `dev-noauth` |
| `BRIDGE_TENANT_ID` | bridge daemon が同期する MEGURU テナント | `8d94ed36-f5d5-4584-8dfb-b9f46d679a41` |
| `BRIDGE_BUS_AREA_ID` | やさいバス側のエリア ID | `32` |
| `BRIDGE_POLL_SECS` | bridge daemon の sync 間隔 | `300`（5 分）|
| `SLACK_WEBHOOK_URL` | エラー＆ Diff 通知先 | `https://hooks.slack.com/...` |
| `RUST_LOG` | ログレベル | `info`、`meguru_bridge=debug,sqlx=warn` 等 |

---

## 6.6 やさいバス エリア ID 一覧

```sql
SELECT bus_area_id, COUNT(*) AS stops
FROM   mtb_bus_stop WHERE delete_flag=0
GROUP  BY bus_area_id ORDER BY stops DESC LIMIT 5;
```

| bus_area_id | エリア | stops |
|---|---|---|
| 32 | 千葉 | 158 |
| 17 | ? | 96 |
| 1 | ? | 93 |
| 7 | ? | 93 |
| 13 | ? | 64 |

シャドウ検証は **エリア 32（千葉）から** が推奨。

---

## 6.7 データの流れ（注文 1 件）

```
[やさいバス] dtb_order.id = 434685
   ├─ farmer_bus_stop_id = 10
   ├─ buyer_bus_stop_id = 2203
   ├─ init_pickup_datetime = 2026-05-04 13:30:00
   ├─ order_status_id = 12 (取引終了)
   └─ dtb_order_container.container_size_id = 3 (Large)
        ↓ bridge が SELECT
[meguru-bridge] mapping.rs で
   ├─ status: 12 → ShipmentStatus::Delivered
   └─ container: size_id=3 → ContainerSize::Large
        ↓ POST /shipments
[meguru-api] /shipments handler
   ↓
[meguru DB] shipments テーブルに INSERT
   └─ external_order_id="434685" で冪等
```

完全なカラム対応は [docs/bridge_mapping.md](../bridge_mapping.md)。

---

## 6.8 シャドウ検証 Phase ロードマップ

| Phase | 内容 | 必要な追加実装 |
|---|---|---|
| Phase 0 | スモーク（10 stops 通過）| 完了 ✅ |
| **Phase 1 ★ 現在地** | 1 エリア全体の stops + connections + orders 取込 | `/stops/bulk` 実装、connections/routes/pricing SELECT |
| Phase 2 | 24 時間連続稼働 + Slack エラー通知 | `meguru-bridge` バイナリで daemon 化 |
| Phase 3 | Diff Reporter 本実装 | `jobs/diff.rs` の TODO を埋める |
| Phase 4 | 全エリア展開・Fly.io 移管 | [fly.shadow.toml](../../fly.shadow.toml) でデプロイ |
| Phase 5 | やさいバス本番から MEGURU 本番に切替 | カットオーバー手順書（別途）|

---

## 6.9 Diff Reporter（毎晩比較）

> 🟡 スケルトンのみ。`crates/meguru-worker/src/jobs/diff.rs` の TODO を実装する必要あり。

ねらい：

```
やさいバスの実際の配送実績 vs MEGURU の予測

┌────────────────────┐    ┌────────────────────┐
│ やさいバス実績      │    │ MEGURU 予測         │
│ - 注文Aは午後便で  │    │ - 注文Aは朝便で    │
│   1日で配送       │ vs │   0日で配送       │
└────────────────────┘    └────────────────────┘
            ↓                       ↓
       一致／不一致を集計
            ↓
        Slack 通知
```

3 つの観点：

1. **Routing**：選んだルートが一致するか
2. **Pricing**：料金計算が一致するか
3. **Capacity**：容量超過の判定が一致するか

---

## 6.10 緊急停止手順

何か怪しいことが起きたら、以下を順に実行：

```bash
# 1. bridge / probe を Ctrl+C で止める
#    → シャドウ書き込みを止める

# 2. SSH トンネルを切る
pkill -f 13307
#    → やさいバスへの物理アクセスを遮断

# 3. （影響を疑う場合のみ）meguru_shadow ユーザーを削除
#    bastion 経由で admin で接続
#    DROP USER 'meguru_shadow'@'10.1.1.%';
#    → 完全にロールバック可能。再作成可能。

# 4. meguru-shadow Postgres を停止
docker-compose stop db
```

⚠️ **やさいバス本番テーブル（dtb_* / mtb_*）には bridge が一切 INSERT/UPDATE/DELETE しません**。`meguru_shadow` ユーザーには `GRANT SELECT` しかないので、コードバグでも書き込みは MySQL レベルで弾かれます。**多層防御** の設計。

---

## 6.11 トラブルシュート

| 症状 | 原因と対処 |
|---|---|
| `YASAIBUS_DATABASE_URL must be set` | env 変数 export 漏れ。6.4 Step 4 を見直す |
| `could not find Cargo.toml` | `/data/m2labo/meguru` で実行する（その上の `/data/m2labo` ではない）|
| `connection refused on 127.0.0.1:13307` | SSH トンネル切れ。6.4 Step 2 再実行 |
| `connection refused on 127.0.0.1:5432` | Postgres docker 落ち。`docker-compose up -d db` |
| `connection refused on 127.0.0.1:3000` | meguru-api 未起動。6.4 Step 3 を別ターミナルで |
| `unmappable: N` が大量 | フラグ全 0 が "可視性" であって "type" ではない仕様。意図通り |
| MySQL Access denied | (1) パスワードファイル読めるか (2) bastion グローバル IP 変化 (3) `meguru_shadow@'10.1.1.%'` 存在確認 |

---

## 6.12 関連ドキュメント

- [docs/MANUAL.md](../MANUAL.md) — 旧マニュアル（このページの前身）
- [docs/bridge_mapping.md](../bridge_mapping.md) — カラム対応表
- [crates/meguru-bridge/](../../crates/meguru-bridge/) — bridge ソース
- [crates/meguru-bridge/src/bin/probe.rs](../../crates/meguru-bridge/src/bin/probe.rs) — probe バイナリ
- [migrations/003_bridge_support.sql](../../migrations/003_bridge_support.sql) — bridge_* 補助テーブル

---

次：やさいバス担当者向けの **全テストパターン** は [07_test_scenarios.md](07_test_scenarios.md)。
