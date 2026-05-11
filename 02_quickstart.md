# 02. クイックスタート（30 分で荷物を 1 件流す）

📍 [目次](README.md) ▶ 02. クイックスタート

このページの目標：**MEGURU をローカルで起動して、テナント作成→バス停登録→ルート登録→荷物作成 まで一気通貫で動かす**こと。所要 30 分。

🎥 **動画候補**：このセクション全体（端末操作のみ、5〜7 分）

---

## 2.1 必要なもの

| 項目 | バージョン | 確認コマンド |
|---|---|---|
| OS | macOS / Linux / WSL2 | `uname -a` |
| Docker | 24+ | `docker --version` |
| docker-compose | v2+ | `docker-compose --version` |
| Rust | 1.78+ | `rustc --version` |
| `curl` | 任意 | `curl --version` |
| `jq`（推奨） | 任意 | `jq --version` |

```bash
$ rustc --version
rustc 1.78.0 (9b00956e5 2024-04-29)
$ docker --version
Docker version 27.0.3, build 7d4bcd8
```

---

## 2.2 リポジトリ取得とディレクトリ移動

```bash
# 既にチェックアウト済みなら skip
git clone <meguru-repo-url> /data/m2labo/meguru
cd /data/m2labo/meguru
```

以降のコマンドは **すべて `/data/m2labo/meguru/` で実行** してください。

---

## 2.3 Step 1: PostgreSQL + Redis を起動

```bash
docker-compose up -d
docker ps --filter name=meguru
```

📸 **期待出力**：[assets/captures/05_ssh_tunnel.png](assets/captures/05_ssh_tunnel.png)

```
CONTAINER ID   IMAGE             ...   STATUS         PORTS
xxxxxxxxxxxx   postgres:16       ...   Up X seconds   0.0.0.0:5432->5432/tcp
xxxxxxxxxxxx   redis:7           ...   Up X seconds   0.0.0.0:6379->6379/tcp
```

うまく起動しない場合は [08_operator_guide.md#docker関連](08_operator_guide.md#docker関連) を参照。

---

## 2.4 Step 2: マイグレーション

```bash
cat migrations/001_initial.sql | docker exec -i meguru_db_1 psql -U meguru -d meguru
cat migrations/002_add_edge_cost_dimensions.sql | docker exec -i meguru_db_1 psql -U meguru -d meguru
cat migrations/003_bridge_support.sql | docker exec -i meguru_db_1 psql -U meguru -d meguru
```

確認：

```bash
docker exec -i meguru_db_1 psql -U meguru -d meguru -c "\dt"
```

`tenants`, `stops`, `routes`, `shipments` などのテーブルが並べば OK。

---

## 2.5 Step 3: 環境変数

```bash
cp .env.example .env
```

クイックスタートでは編集不要（開発用デフォルトでそのまま動く）。

---

## 2.6 Step 4: API サーバー起動

```bash
DATABASE_URL='postgres://meguru:meguru_dev@localhost:5432/meguru' \
RUST_LOG='meguru_api=info' \
cargo run --bin meguru-api
```

別ターミナルで：

```bash
curl -s http://localhost:3000/health
```

📸 **期待出力**：[assets/captures/01_health_check.png](assets/captures/01_health_check.png)

```json
{"status":"ok","version":"0.1.0"}
```

✅ ここまでで「サーバが動く」状態は完成。

---

## 2.7 Step 5: テナント作成

> 以降のコマンドは別ターミナル（API サーバを止めずに）で実行してください。

```bash
curl -s -X POST http://localhost:3000/admin/tenants \
  -H 'Content-Type: application/json' \
  -H 'X-API-Key: dev-noauth' \
  -d '{"name":"クイックスタート用テナント","plan":"starter"}' | jq
```

期待出力：

```json
{
  "id": "01a2b3c4-...",
  "name": "クイックスタート用テナント",
  "plan": "starter",
  "active": true,
  "created_at": "2026-05-12T..."
}
```

ここで返ってきた `id` を **以降 `$TENANT` として保存** します。

```bash
TENANT='01a2b3c4-...'   # ← 実際の id をコピー
```

> 💡 jq があれば一行で取れます：
> ```bash
> TENANT=$(curl -s -X POST http://localhost:3000/admin/tenants \
>   -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
>   -d '{"name":"qs","plan":"starter"}' | jq -r .id)
> echo $TENANT
> ```

---

## 2.8 Step 6: バス停を 3 つ作る

最小構成： **農家A（集荷）→ 中継X（積替）→ レストランB（配達）**

```bash
# 農家A
FARM=$(curl -s -X POST "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"農家A","address":"千葉県千葉市…","latitude":35.61,"longitude":140.10,"stop_type":"collection","capacity_cases":100}' \
  | jq -r .id)

# 中継X
HUB=$(curl -s -X POST "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"中継X","address":"千葉県千葉市美浜区…","latitude":35.64,"longitude":140.04,"stop_type":"transit","capacity_cases":500}' \
  | jq -r .id)

# レストランB
REST=$(curl -s -X POST "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"レストランB","address":"東京都江東区…","latitude":35.66,"longitude":139.80,"stop_type":"delivery","capacity_cases":50}' \
  | jq -r .id)

echo "FARM=$FARM HUB=$HUB REST=$REST"
```

確認：

```bash
curl -s "http://localhost:3000/stops?tenant_id=$TENANT" -H 'X-API-Key: dev-noauth' | jq '.[].name'
```

📸 **期待出力**：[assets/captures/04_api_list_stops.png](assets/captures/04_api_list_stops.png)

```
"農家A"
"中継X"
"レストランB"
```

---

## 2.9 Step 7: ルート 2 本を作る

```bash
# 朝便：農家A → 中継X
MORNING=$(curl -s -X POST "http://localhost:3000/routes?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{
    "name":"朝便",
    "days_of_week":["mon","tue","wed","thu","fri"],
    "departure_time":"08:00:00",
    "temperature":"refrigerated",
    "capacity_cases":50,
    "stops":["'"$FARM"'","'"$HUB"'"]
  }' | jq -r .id)

# 午後便：中継X → レストランB
EVENING=$(curl -s -X POST "http://localhost:3000/routes?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{
    "name":"午後便",
    "days_of_week":["mon","tue","wed","thu","fri"],
    "departure_time":"13:00:00",
    "temperature":"refrigerated",
    "capacity_cases":50,
    "stops":["'"$HUB"'","'"$REST"'"]
  }' | jq -r .id)
```

---

## 2.10 Step 8: ストップ接続（グラフの辺）を作る

```bash
curl -s -X POST "http://localhost:3000/connections/bulk?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{
    "connections":[
      {"from_stop_id":"'"$FARM"'","to_stop_id":"'"$HUB"'","days_of_week":["mon","tue","wed","thu","fri"],"transit_days":0,"active_from":"2026-01-01"},
      {"from_stop_id":"'"$HUB"'","to_stop_id":"'"$REST"'","days_of_week":["mon","tue","wed","thu","fri"],"transit_days":0,"active_from":"2026-01-01"}
    ]
  }' | jq
```

---

## 2.11 Step 9: 荷物を 1 件作る（**ここで配車エンジンが動く**）

```bash
curl -s -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{
    "origin_stop_id":"'"$FARM"'",
    "destination_stop_id":"'"$REST"'",
    "scheduled_date":"2026-05-13",
    "cases":3,
    "container_size":"medium",
    "temperature":"refrigerated",
    "external_order_id":"QS-001"
  }' | jq
```

期待出力（抜粋）：

```json
{
  "id": "f1e2d3c4-...",
  "tenant_id": "01a2b3c4-...",
  "external_order_id": "QS-001",
  "origin_stop_id": "...",          // FARM
  "destination_stop_id": "...",     // REST
  "scheduled_date": "2026-05-13",
  "cases": 3,
  "status": "pending",
  "legs": [
    {"leg_order": 1, "route_id": "...", "from_stop_id": "...", "to_stop_id": "..."},  // 農家A → 中継X
    {"leg_order": 2, "route_id": "...", "from_stop_id": "...", "to_stop_id": "..."}   // 中継X → レストランB
  ]
}
```

🎉 **2 レグに自動分割された！** ダイクストラ法が「直行ルートはないが、中継Xを介せば届く」と判断したわけです。

---

## 2.12 Step 10: 荷物を確認・キャンセル

```bash
# 一覧
curl -s "http://localhost:3000/shipments?tenant_id=$TENANT" -H 'X-API-Key: dev-noauth' | jq

# 特定の荷物
SHIPMENT_ID='f1e2d3c4-...'
curl -s "http://localhost:3000/shipments/$SHIPMENT_ID" -H 'X-API-Key: dev-noauth' | jq

# キャンセル（status が picked_up より前のときだけ通る）
curl -s -X PATCH "http://localhost:3000/shipments/$SHIPMENT_ID/cancel" \
  -H 'X-API-Key: dev-noauth' -w "\nHTTP %{http_code}\n"
```

---

## 2.13 ここまでで触れた API のおさらい

| メソッド | パス | 用途 |
|---|---|---|
| GET | `/health` | 死活確認 |
| POST | `/admin/tenants` | テナント作成 |
| POST | `/stops?tenant_id=...` | バス停作成 |
| GET | `/stops?tenant_id=...` | バス停一覧 |
| POST | `/routes?tenant_id=...` | ルート作成 |
| POST | `/connections/bulk?tenant_id=...` | 接続一括登録 |
| POST | `/shipments?tenant_id=...` | 荷物作成（配車エンジン起動）|
| GET | `/shipments?tenant_id=...` | 荷物一覧 |
| GET | `/shipments/:id` | 荷物詳細 |
| PATCH | `/shipments/:id/cancel` | 荷物キャンセル |

全エンドポイントの詳細は [09_api_reference.md](09_api_reference.md)。

---

## 2.14 つまづいたら

| 症状 | 参照先 |
|---|---|
| `curl: (7) Failed to connect to localhost port 3000` | [08_operator_guide.md#port-3000](08_operator_guide.md#トラブルシュート) |
| `connection refused on 127.0.0.1:5432` | docker が落ちている → `docker-compose up -d` |
| 401 Unauthorized | `X-API-Key: dev-noauth` を付け忘れ |
| 422 Unprocessable Entity（shipment 作成時） | 経路が見つからない → ストップ接続が抜けている可能性 |

---

次：

- 運営事業者として全体設定したい → [03_admin_guide.md](03_admin_guide.md)
- 外部システム連携したい → [04_shipper_guide.md](04_shipper_guide.md)
- すべてのテストパターンを潰したい（**やさいバス担当者**）→ [07_test_scenarios.md](07_test_scenarios.md)
