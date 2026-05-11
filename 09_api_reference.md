# 09. API リファレンス

📍 [目次](README.md) ▶ 09. API リファレンス

このページの読者：MEGURU API を**直接叩く開発者**。

ベース URL（ローカル）：`http://localhost:3000`

---

## 9.1 認証

| 方式 | ヘッダ | 対象 |
|---|---|---|
| API Key | `X-API-Key: <key>` | 外部システム |
| JWT | `Authorization: Bearer <token>` | 管理者・ドライバー |
| 開発用バイパス | `X-API-Key: dev-noauth` | ローカル開発のみ |

公開エンドポイント（認証不要）：

- `GET /health`
- `POST /auth/login`

---

## 9.2 エンドポイント全一覧

| メソッド | パス | カテゴリ | 認証 |
|---|---|---|---|
| GET | `/health` | システム | 不要 |
| POST | `/auth/login` | 認証 | 不要 |
| GET | `/admin/tenants` | テナント | admin |
| POST | `/admin/tenants` | テナント | admin |
| PATCH | `/admin/tenants/:id` | テナント | admin |
| GET | `/stops?tenant_id=...` | バス停 | admin/shipper |
| POST | `/stops?tenant_id=...` | バス停 | admin |
| GET | `/routes?tenant_id=...` | ルート | admin |
| POST | `/routes?tenant_id=...` | ルート | admin |
| GET | `/routes/:id` | ルート | admin |
| PATCH | `/routes/:id` | ルート | admin |
| DELETE | `/routes/:id` | ルート | admin |
| GET | `/connections?tenant_id=...` | 接続 | admin |
| POST | `/connections?tenant_id=...` | 接続 | admin |
| POST | `/connections/bulk?tenant_id=...` | 接続 | admin |
| DELETE | `/connections/:id` | 接続 | admin |
| GET | `/drivers?tenant_id=...` | ドライバー | admin |
| POST | `/drivers?tenant_id=...` | ドライバー | admin |
| POST | `/shipments?tenant_id=...` | 荷物 | shipper/admin |
| GET | `/shipments?tenant_id=...` | 荷物 | shipper/admin |
| GET | `/shipments/:id` | 荷物 | shipper/admin |
| PATCH | `/shipments/:id/cancel` | 荷物 | shipper/admin |

---

## 9.3 共通

### エラーフォーマット

```json
{"error":"<人間が読めるメッセージ>"}
```

| HTTP | 意味 |
|---|---|
| 200 | 成功（GET / PATCH） |
| 201 | 作成成功 |
| 204 | 削除成功 |
| 400 | リクエストJSON不正 |
| 401 | 認証失敗 |
| 403 | 認可失敗（テナント越え等） |
| 404 | 見つからない |
| 422 | バリデーション・経路探索失敗 |
| 500 | サーバ内部エラー |

### 日付・時刻

- 日付：`YYYY-MM-DD`
- 時刻：`HH:MM:SS`
- タイムスタンプ：`YYYY-MM-DDTHH:MM:SS±ZZ:ZZ`（RFC 3339）

### UUID

すべて UUID v4。例：`8d94ed36-f5d5-4584-8dfb-b9f46d679a41`

---

## 9.4 システム

### GET /health

```http
GET /health
```

```json
{"status":"ok","version":"0.1.0"}
```

---

## 9.5 認証

### POST /auth/login

```http
POST /auth/login
Content-Type: application/json

{
  "tenant_id": "01a2b3c4-...",
  "role": "admin"     // "admin" | "driver"
}
```

```json
{"token":"eyJhbGc..."}
```

> 🟡 MVP では実際のパスワード検証なし。本番化時にユーザーテーブル＋bcrypt 等。

---

## 9.6 テナント（admin）

### POST /admin/tenants

```http
POST /admin/tenants
Content-Type: application/json
X-API-Key: <admin-key>

{
  "name": "〇〇配送",
  "plan": "starter"    // starter | standard | pro | enterprise
}
```

```json
{
  "id": "...",
  "name": "〇〇配送",
  "plan": "starter",
  "active": true,
  "created_at": "2026-05-12T..."
}
```

### GET /admin/tenants

→ テナント一覧。

### PATCH /admin/tenants/:id

```http
PATCH /admin/tenants/<id>
{"name":"新名","plan":"pro","active":false}
```

---

## 9.7 バス停

### POST /stops?tenant_id=&lt;id&gt;

```http
POST /stops?tenant_id=...
Content-Type: application/json

{
  "name":"農家A",
  "address":"千葉県千葉市…",
  "latitude":35.6134,
  "longitude":140.1688,
  "stop_type":"collection",
  "capacity_cases":100
}
```

| フィールド | 型 | 必須 |
|---|---|---|
| `name` | string | ✅ |
| `address` | string | ✅ |
| `latitude` | f64 (-90〜90) | ✅ |
| `longitude` | f64 (-180〜180) | ✅ |
| `stop_type` | enum: `collection`/`delivery`/`transit`/`both`/`garage` | ✅ |
| `capacity_cases` | int (>0) | ✅ |

```json
{
  "id":"...",
  "tenant_id":"...",
  "name":"農家A",
  "address":"千葉県千葉市…",
  "location":{"latitude":35.6134,"longitude":140.1688},
  "stop_type":"collection",
  "capacity_cases":100
}
```

### GET /stops?tenant_id=&lt;id&gt;

→ テナントの全バス停を配列で返す。

---

## 9.8 ルート

### POST /routes?tenant_id=&lt;id&gt;

```http
POST /routes?tenant_id=...
{
  "name":"千葉朝便",
  "days_of_week":["mon","wed","fri"],
  "departure_time":"07:00:00",
  "temperature":"refrigerated",
  "capacity_cases":80,
  "stops":["<stop_id_1>","<stop_id_2>","<stop_id_3>"]
}
```

`temperature`：`ambient` / `refrigerated` / `frozen`
`days_of_week`：曜日略号（小文字）配列

### GET /routes?tenant_id=&lt;id&gt;

### GET /routes/:id

### PATCH /routes/:id

```http
{"name":"新名","capacity_cases":100,"active":false}
```

### DELETE /routes/:id

→ HTTP 204。連鎖で未配送 shipment が再ルーティング対象に。

---

## 9.9 ストップ接続

### POST /connections?tenant_id=&lt;id&gt;

```http
POST /connections?tenant_id=...
{
  "from_stop_id":"...",
  "to_stop_id":"...",
  "days_of_week":["tue","thu","fri"],
  "transit_days":0,
  "distance_m":15000,
  "cost_jpy":300,
  "co2_g":2400,
  "active_from":"2026-04-01",
  "active_until":null
}
```

### POST /connections/bulk?tenant_id=&lt;id&gt;

```http
POST /connections/bulk?tenant_id=...
{
  "connections":[
    { ... }, { ... }, { ... }
  ]
}
```

### GET /connections?tenant_id=&lt;id&gt;

### DELETE /connections/:id

---

## 9.10 ドライバー

### POST /drivers?tenant_id=&lt;id&gt;

```http
POST /drivers?tenant_id=...
{"name":"山田太郎","phone":"090-1234-5678"}
```

### GET /drivers?tenant_id=&lt;id&gt;

---

## 9.11 荷物（Shipments）— 中核 API

### POST /shipments?tenant_id=&lt;id&gt;

```http
POST /shipments?tenant_id=...
{
  "origin_stop_id":"...",
  "destination_stop_id":"...",
  "scheduled_date":"2026-05-13",
  "cases":3,
  "container_size":"medium",
  "temperature":"refrigerated",
  "external_order_id":"yasai-12345"
}
```

| フィールド | 必須 |
|---|---|
| `origin_stop_id` | ✅ |
| `destination_stop_id` | ✅ |
| `scheduled_date` | ✅ |
| `cases` | ✅ |
| `container_size` | ✅ |
| `temperature` | 任意 |
| `external_order_id` | 推奨 |

**動作**：
1. テナントの全ストップ・接続・ルートを取得
2. `GraphDispatchEngine` を構築
3. ダイクストラで `origin → destination` の最短経路を計算
4. 経路をレグに分割
5. `shipments` + `shipment_legs` に INSERT

**レスポンス（201）**：

```json
{
  "id":"...",
  "tenant_id":"...",
  "external_order_id":"yasai-12345",
  "origin_stop_id":"...",
  "destination_stop_id":"...",
  "scheduled_date":"2026-05-13",
  "cases":3,
  "container_size":"medium",
  "temperature":"refrigerated",
  "status":"pending",
  "legs":[
    {"id":"...","leg_order":1,"route_id":"...","from_stop_id":"...","to_stop_id":"...","scheduled_date":"2026-05-13","status":"pending"},
    {"id":"...","leg_order":2,"route_id":"...","from_stop_id":"...","to_stop_id":"...","scheduled_date":"2026-05-13","status":"pending"}
  ]
}
```

**エラー**：

| HTTP | error | 原因 |
|---|---|---|
| 422 | `no path found` | 接続グラフに経路が存在しない |
| 422 | `invalid container_size` | enum 外 |
| 422 | `origin and destination are the same` | 同一バス停 |
| 422 | `cases must be positive` | 0 / 負数 |

### GET /shipments?tenant_id=&lt;id&gt;

### GET /shipments/:id

### PATCH /shipments/:id/cancel

| HTTP | 状況 |
|---|---|
| 200 | キャンセル成功 |
| 422 | 既に `picked_up` 以降で不可 |
| 404 | 見つからない |

---

## 9.12 enum リファレンス

### stop_type
```
collection | delivery | transit | both | garage
```

### container_size
```
small | medium | large | xlarge | xxlarge
```
（multiplier: 1.0 / 1.5 / 2.0 / 2.5 / 3.0）

### temperature
```
ambient | refrigerated | frozen
```

### shipment.status
```
pending | confirmed | picked_up | in_transit | delivered | cancelled | failed
```

### shipment_leg.status
```
pending | picked_up | in_transit | transit_ready | delivered | cancelled
```

### dispatch.status
```
scheduled | in_progress | completed | cancelled
```

### tenant.plan
```
starter | standard | pro | enterprise
```

### days_of_week
```
mon | tue | wed | thu | fri | sat | sun
```

---

## 9.13 OpenAPI / Swagger

🟡 自動生成スキーマは未提供。必要なら `utoipa` クレートで追加可能。

---

次：DB スキーマは [10_data_model.md](10_data_model.md)。
