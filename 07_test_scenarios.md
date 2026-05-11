# 07. テストシナリオ集（やさいバス担当者向け）

📍 [目次](README.md) ▶ 07. テストシナリオ集

このページの読者：**やさいバスチームで MEGURU をテストする担当者**。

**このマニュアル全体で一番大事なページ** です。MEGURU を本番運用する前に、ここに書かれた **全パターン** を順に流して、想定通りの挙動か確認してください。

🎥 **動画候補**：各パターンごとに 1〜3 分のデモを撮影してまとめ動画にする想定

---

## 0. 事前準備（テスト共通）

### 0.1 環境を立ち上げる

[02_quickstart.md](02_quickstart.md) Step 1〜4 まで実行し、API サーバが `http://localhost:3000` で動いている状態を作る。

### 0.2 テスト用テナントと変数を作る

```bash
# テナント作成
TENANT=$(curl -s -X POST http://localhost:3000/admin/tenants \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"test-tenant","plan":"starter"}' | jq -r .id)

KEY='dev-noauth'   # 開発用ダミーキー
echo "TENANT=$TENANT"
```

### 0.3 検証用シェル関数（コピペ推奨）

`~/.meguru-test.sh` に置いて `source` すると便利：

```bash
api() { curl -s "$@" -H "X-API-Key: $KEY"; }
api_json() { curl -s "$@" -H "X-API-Key: $KEY" -H "Content-Type: application/json"; }
http_code() { curl -s -o /dev/null -w "%{http_code}" "$@" -H "X-API-Key: $KEY"; }

mk_stop() {
  api_json -X POST "http://localhost:3000/stops?tenant_id=$TENANT" \
    -d "{\"name\":\"$1\",\"address\":\"$1の住所\",\"latitude\":$2,\"longitude\":$3,\"stop_type\":\"$4\",\"capacity_cases\":100}" \
    | jq -r .id
}

mk_conn() {
  api_json -X POST "http://localhost:3000/connections?tenant_id=$TENANT" \
    -d "{\"from_stop_id\":\"$1\",\"to_stop_id\":\"$2\",\"days_of_week\":$3,\"transit_days\":$4,\"active_from\":\"2026-01-01\"}"
}

mk_shipment() {
  api_json -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
    -d "{\"origin_stop_id\":\"$1\",\"destination_stop_id\":\"$2\",\"scheduled_date\":\"$3\",\"cases\":$4,\"container_size\":\"$5\",\"external_order_id\":\"$6\"}"
}
```

これを source した前提で以降のテストを書きます。

---

## 1. ハッピーパス（基本動作）

### Test 1.1 — シンプルな直行配送 ✅

**目的**：A → B のシンプルな配送が 1 レグで割り当てられること。

```bash
A=$(mk_stop "A-farm" 35.61 140.10 "collection")
B=$(mk_stop "B-shop" 35.66 139.80 "delivery")
mk_conn "$A" "$B" '["mon","tue","wed","thu","fri"]' 0

mk_shipment "$A" "$B" "2026-05-13" 3 "medium" "T1.1" | jq
```

**期待**：

- `status: "pending"`
- `legs` の長さが **1**
- `legs[0].from_stop_id == A`, `legs[0].to_stop_id == B`
- HTTP 201

✅ 合格基準：レスポンスの `legs` が 1 件、from/to が想定通り。

---

### Test 1.2 — 中継ありの 2 レグ配送 ✅

**目的**：直行ルートがなく中継経由になるケースが 2 レグに分割されること。

```bash
A=$(mk_stop "A-farm" 35.61 140.10 "collection")
X=$(mk_stop "X-hub" 35.64 140.04 "transit")
B=$(mk_stop "B-shop" 35.66 139.80 "delivery")
mk_conn "$A" "$X" '["mon","tue","wed","thu","fri"]' 0
mk_conn "$X" "$B" '["mon","tue","wed","thu","fri"]' 0

mk_shipment "$A" "$B" "2026-05-13" 3 "medium" "T1.2" | jq
```

**期待**：

- `legs` の長さが **2**
- `legs[0]`: A → X
- `legs[1]`: X → B

✅ 合格基準：中継 X を経由する 2 レグが返る。

---

### Test 1.3 — 翌日配送（transit_days=1）✅

**目的**：日またぎ配送で `scheduled_date` がレグ間で 1 日ずれること。

```bash
A=$(mk_stop "A-farm" 35.61 140.10 "collection")
X=$(mk_stop "X-hub" 35.64 140.04 "transit")
B=$(mk_stop "B-shop" 35.66 139.80 "delivery")
mk_conn "$A" "$X" '["mon","tue","wed","thu","fri"]' 0
mk_conn "$X" "$B" '["tue","wed","thu","fri","sat"]' 1   # 翌日着

mk_shipment "$A" "$B" "2026-05-13" 3 "medium" "T1.3" | jq
```

**期待**：

- レグ1の `scheduled_date == "2026-05-13"`
- レグ2の `scheduled_date == "2026-05-14"`（翌日）

✅ 合格基準：日付が 1 日ずれる。

---

### Test 1.4 — 3 ホップ以上の経路 ✅

**目的**：A → X → Y → B のような 3 レグ以上も組めること。

```bash
A=$(mk_stop "A" 35.61 140.10 "collection")
X=$(mk_stop "X" 35.64 140.04 "transit")
Y=$(mk_stop "Y" 35.65 139.95 "transit")
B=$(mk_stop "B" 35.66 139.80 "delivery")
mk_conn "$A" "$X" '["mon"]' 0
mk_conn "$X" "$Y" '["mon"]' 0
mk_conn "$Y" "$B" '["mon"]' 0

mk_shipment "$A" "$B" "2026-05-11" 1 "small" "T1.4" | jq '.legs | length'
# 期待: 3
```

✅ 合格基準：3 レグで返る。

---

## 2. 経路選択ロジック

### Test 2.1 — 複数経路がある場合の最短選択 ✅

**目的**：A → B の経路が 2 通り（直行 / 中継）あるとき、`transit_days` が小さい方が選ばれる。

```bash
A=$(mk_stop "A" 35.61 140.10 "collection")
X=$(mk_stop "X" 35.64 140.04 "transit")
B=$(mk_stop "B" 35.66 139.80 "delivery")
# 経路1: A→B（直行、1日かかる）
mk_conn "$A" "$B" '["mon"]' 1
# 経路2: A→X→B（中継、0日+0日=0日）
mk_conn "$A" "$X" '["mon"]' 0
mk_conn "$X" "$B" '["mon"]' 0

mk_shipment "$A" "$B" "2026-05-11" 1 "small" "T2.1" | jq '.legs | length'
# 期待: 2（中継ルートが選ばれる）
```

✅ 合格基準：日数が短い中継経路が選ばれる（2 レグ）。

---

### Test 2.2 — 接続がその曜日に運行していない ❌

**目的**：火曜日にしか走らない接続を、月曜日の配送に使えないことを確認。

```bash
A=$(mk_stop "A" 35.61 140.10 "collection")
B=$(mk_stop "B" 35.66 139.80 "delivery")
mk_conn "$A" "$B" '["tue"]' 0   # 火曜のみ

# 2026-05-11 は月曜 → 経路なし期待
code=$(http_code -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-11\",\"cases\":1,\"container_size\":\"small\",\"external_order_id\":\"T2.2\"}")
echo "HTTP $code"   # 期待: 422
```

✅ 合格基準：HTTP 422 が返る。

---

### Test 2.3 — `active_until` を過ぎている接続 ❌

```bash
A=$(mk_stop "A" 35.61 140.10 "collection")
B=$(mk_stop "B" 35.66 139.80 "delivery")
# 2026-04-30 までしか有効でない接続
api_json -X POST "http://localhost:3000/connections?tenant_id=$TENANT" \
  -d "{\"from_stop_id\":\"$A\",\"to_stop_id\":\"$B\",\"days_of_week\":[\"mon\"],\"transit_days\":0,\"active_from\":\"2026-01-01\",\"active_until\":\"2026-04-30\"}"

# 5/12 配送は使えないはず
code=$(http_code -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-12\",\"cases\":1,\"container_size\":\"small\",\"external_order_id\":\"T2.3\"}")
echo "HTTP $code"   # 期待: 422
```

✅ 合格基準：HTTP 422。

---

### Test 2.4 — 同じ from/to の接続が複数あるとき

接続を二重登録した場合の挙動。

```bash
A=$(mk_stop "A" 35.61 140.10 "collection")
B=$(mk_stop "B" 35.66 139.80 "delivery")
mk_conn "$A" "$B" '["mon"]' 0    # 1本目
mk_conn "$A" "$B" '["mon"]' 1    # 2本目（同条件・違う日数）
# 最短 = 1本目 が選ばれることを確認
mk_shipment "$A" "$B" "2026-05-11" 1 "small" "T2.4" | jq
```

✅ 合格基準：`transit_days=0` の方が選ばれる。

---

## 3. ステータス遷移

### Test 3.1 — pending → confirmed → ... → delivered

> 🟡 各 PATCH エンドポイントは Phase 2。現状は DB 直接操作でも可。

```sql
-- DB から直接遷移させて反映するか確認
UPDATE shipments SET status = 'confirmed' WHERE id = '<shipment_id>';
UPDATE shipments SET status = 'picked_up' WHERE id = '<shipment_id>';
UPDATE shipments SET status = 'in_transit' WHERE id = '<shipment_id>';
UPDATE shipments SET status = 'delivered' WHERE id = '<shipment_id>';
```

各遷移後に `GET /shipments/:id` で確認。

✅ 合格基準：各 status が API レスポンスに正しく反映される。

---

### Test 3.2 — picked_up 以降はキャンセル不可

```bash
# Test 1.1 の shipment を picked_up 状態に
SID=$(mk_shipment "$A" "$B" "2026-05-13" 3 "medium" "T3.2" | jq -r .id)
docker exec -i meguru_db_1 psql -U meguru -d meguru \
  -c "UPDATE shipments SET status='picked_up' WHERE id='$SID'"

# キャンセル試行 → 422 期待
code=$(http_code -X PATCH "http://localhost:3000/shipments/$SID/cancel")
echo "HTTP $code"   # 期待: 422
```

✅ 合格基準：HTTP 422 + エラーメッセージ。

---

### Test 3.3 — pending はキャンセル可能

```bash
SID=$(mk_shipment "$A" "$B" "2026-05-13" 1 "small" "T3.3" | jq -r .id)
code=$(http_code -X PATCH "http://localhost:3000/shipments/$SID/cancel")
echo "HTTP $code"   # 期待: 200
api "http://localhost:3000/shipments/$SID" | jq .status   # 期待: "cancelled"
```

✅ 合格基準：HTTP 200、status が `cancelled`。

---

### Test 3.4 — 二重キャンセル

```bash
# 既にキャンセル済の荷物を再キャンセル
code=$(http_code -X PATCH "http://localhost:3000/shipments/$SID/cancel")
echo "HTTP $code"   # 期待: 422
```

✅ 合格基準：HTTP 422。

---

## 4. コンテナサイズ・容量

### Test 4.1 — 各コンテナサイズで作成

```bash
for size in small medium large xlarge xxlarge; do
  mk_shipment "$A" "$B" "2026-05-13" 1 "$size" "T4.1-$size" | jq -r '"\(.container_size): \(.id)"'
done
```

✅ 合格基準：5 件すべて作成成功、`container_size` が要求どおり。

---

### Test 4.2 — 不正なサイズ ❌

```bash
code=$(http_code -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":1,\"container_size\":\"huge\",\"external_order_id\":\"T4.2\"}")
echo "HTTP $code"   # 期待: 422 もしくは 400
```

✅ 合格基準：4xx 系で拒否。

---

### Test 4.3 — ルート容量超過

```bash
# capacity_cases=50 のルートに 100 ケースを乗せようとする
# （現状は警告のみ／予約超過判定は Phase 2）
mk_shipment "$A" "$B" "2026-05-13" 100 "large" "T4.3" | jq
```

✅ 合格基準（現状）：作成は通るが、ログにキャパ警告が出ること。

---

## 5. マルチテナント分離

### Test 5.1 — テナント A のデータが テナント B から見えない

```bash
T1=$(curl -s -X POST http://localhost:3000/admin/tenants \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"T1","plan":"starter"}' | jq -r .id)
T2=$(curl -s -X POST http://localhost:3000/admin/tenants \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"T2","plan":"starter"}' | jq -r .id)

# T1 にバス停 1 件
curl -s -X POST "http://localhost:3000/stops?tenant_id=$T1" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"T1の店","address":"a","latitude":35.6,"longitude":140.0,"stop_type":"both","capacity_cases":10}'

# T2 で stops を取得 → 0 件期待
curl -s "http://localhost:3000/stops?tenant_id=$T2" -H 'X-API-Key: dev-noauth' | jq 'length'
# 期待: 0
```

✅ 合格基準：T2 からは T1 のバス停が見えない。

---

### Test 5.2 — テナント無効化

```bash
curl -X PATCH "http://localhost:3000/admin/tenants/$T1" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"active":false}'

# 無効テナントに対するリクエストの挙動
code=$(http_code -X POST "http://localhost:3000/shipments?tenant_id=$T1" \
  -H 'Content-Type: application/json' \
  -d '{...}')
echo "HTTP $code"
```

✅ 合格基準：無効テナントへの書き込みが拒否される（仕様確認）。

---

## 6. 認証

### Test 6.1 — API キーなし ❌

```bash
code=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/stops?tenant_id=$TENANT")
echo "HTTP $code"   # 期待: 401
```

✅ 合格基準：HTTP 401。

---

### Test 6.2 — 間違った API キー ❌

```bash
code=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H 'X-API-Key: wrong-key')
echo "HTTP $code"   # 期待: 401
```

✅ 合格基準：HTTP 401。

> 🟡 現状は dev-noauth しか実装されていないため、未認証なら 401 になるが、何でも通る状態。本番前に必ず実 API キー検証を入れる。

---

### Test 6.3 — JWT 取得

```bash
JWT=$(curl -s -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d "{\"tenant_id\":\"$TENANT\",\"role\":\"admin\"}" | jq -r .token)
echo "$JWT"
```

✅ 合格基準：3 セグメントの JWT 文字列が返る。

---

### Test 6.4 — JWT で認証された API 呼び出し

```bash
curl -s "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H "Authorization: Bearer $JWT" | jq
```

✅ 合格基準：認証成功、stops が返る。

---

## 7. 冪等性

### Test 7.1 — 同じ external_order_id で二重 POST

```bash
mk_shipment "$A" "$B" "2026-05-13" 1 "small" "DUP-001" | jq .id
mk_shipment "$A" "$B" "2026-05-13" 1 "small" "DUP-001" | jq .id
# 期待: 同じ id が返る（または 2 回目が 409）
```

✅ 合格基準（仕様）：重複作成されない（同 id 返却 or 409）。

---

## 8. やさいバスデータでの実機検証

### Test 8.1 — シャドウ probe 実行

[06_shadow_testing.md#step-4-probeスモークテスト](06_shadow_testing.md#step-4-probeスモークテスト) を実行し、

- `count=158` バス停取得
- `posted=10 failed=0`
- `unmappable=0`

すべて満たすこと。

✅ 合格基準：すべて満たす。

---

### Test 8.2 — order_status_id マッピング全 13 値

| やさいバス | MEGURU |
|---|---|
| 1 仮注文 | `pending` |
| 2 注文受付 | `confirmed` |
| 3 キャンセル | `cancelled` |
| 4 出荷準備中 | `confirmed` |
| 5 出荷済 | `picked_up` |
| 6 配送中 | `in_transit` |
| 7 配達完了 | `delivered` |
| 8 取消依頼中 | `pending` |
| 9 取消承認 | `cancelled` |
| 10 取消拒否 | `confirmed` |
| 11 配送異常 | `failed` |
| 12 取引終了 | `delivered` |
| 13 取消再受付 | `confirmed` |

> 確認：[crates/meguru-bridge/src/mapping.rs](../../crates/meguru-bridge/src/mapping.rs) と一致するか
> 上記表は推定。実装と差異があれば実装を正とする。

```sql
SELECT DISTINCT order_status_id FROM dtb_order WHERE bus_area_id=32 ORDER BY 1;
```

→ probe を流して各 status の `mapped_status` が想定通りか確認。

---

### Test 8.3 — container_size_id マッピング全 5 値

| やさいバス | MEGURU |
|---|---|
| 1 | `small` |
| 2 | `medium` |
| 3 | `large` |
| 4 | `xlarge` |
| 5 | `xxlarge` |

```sql
SELECT DISTINCT container_size_id, COUNT(*)
FROM dtb_order_container WHERE order_id IN (
  SELECT id FROM dtb_order WHERE bus_area_id=32
)
GROUP BY container_size_id;
```

✅ 合格基準：1〜5 すべてが正しく変換される。

---

### Test 8.4 — StopType フラグ全 16 通り

`mtb_bus_stop` は 4 フラグ（farmer / buyer / transit / garage）の組合せ＝最大 16 通り。

```sql
SELECT
  flag_farmer, flag_buyer, flag_transit, flag_garage, COUNT(*)
FROM mtb_bus_stop WHERE delete_flag=0 AND bus_area_id=32
GROUP BY flag_farmer, flag_buyer, flag_transit, flag_garage
ORDER BY 1,2,3,4;
```

→ 各組合せが `stop_type` enum（collection/delivery/transit/both/garage）に正しく変換されるか確認。詳細マッピング表：[docs/bridge_mapping.md](../bridge_mapping.md)。

✅ 合格基準：probe ログの `by_type` 集計と上の SQL の集計が **完全一致** する（`unmappable=0`）。

---

## 9. 異常系・境界値

| # | テスト | 期待 |
|---|---|---|
| 9.1 | 存在しない `tenant_id` で POST | 401 or 404 |
| 9.2 | 存在しない `origin_stop_id` | 422 |
| 9.3 | `cases: 0` | 422（正の整数のみ） |
| 9.4 | `cases: -5` | 422 |
| 9.5 | `scheduled_date` が過去 | 仕様確認（許容するか） |
| 9.6 | `scheduled_date` が極端な未来（例 2099-01-01）| 200 だが要警告 |
| 9.7 | `origin == destination` | 422 |
| 9.8 | 巨大ペイロード（1MB の JSON） | 413 |
| 9.9 | 不正 JSON | 400 |
| 9.10 | 同一ストップを `origin` `destination` 両方に | 422 |

---

## 10. パフォーマンス・負荷

| # | テスト | 目標 |
|---|---|---|
| 10.1 | バス停 1000 件 + 接続 5000 件で shipment 作成 | 200ms 以内 |
| 10.2 | 1 分間に 100 shipment 並列作成 | エラー 0 |
| 10.3 | 大規模グラフでの最短経路（10 ホップ）| 1 秒以内 |

```bash
# 雑な並列テスト（要 ab）
ab -n 1000 -c 10 -p shipment.json -T application/json \
   -H "X-API-Key: dev-noauth" \
   "http://localhost:3000/shipments?tenant_id=$TENANT"
```

---

## 11. データ整合性

### Test 11.1 — Shipment キャンセルでレグも cancelled に

```bash
SID=$(mk_shipment "$A" "$B" "2026-05-13" 1 "small" "T11.1" | jq -r .id)
curl -X PATCH "http://localhost:3000/shipments/$SID/cancel" -H "X-API-Key: $KEY"

docker exec -i meguru_db_1 psql -U meguru -d meguru \
  -c "SELECT status FROM shipment_legs WHERE shipment_id='$SID'"
# 期待: 全レグ cancelled
```

✅ 合格基準：レグも cancelled。

---

### Test 11.2 — ルート削除時、未配送荷物の再ルーティング

> 🟡 reroute ワーカー実装次第。

```bash
# ルートを削除
curl -X DELETE "http://localhost:3000/routes/$ROUTE_ID" -H "X-API-Key: $KEY"

# 該当ルートに乗っていた pending 荷物が…
# (a) 別ルートに振り替えられる、または
# (b) failed に遷移する
```

✅ 合格基準：仕様どおりの挙動（事前に決めて記載）。

---

## 12. テスト結果記録テンプレ

| Test# | 内容 | 期待 | 実測 | 判定 | 備考 |
|---|---|---|---|---|---|
| 1.1 | 直行配送 | legs.length=1 | | ⬜ | |
| 1.2 | 中継 2 レグ | legs.length=2 | | ⬜ | |
| 1.3 | 翌日配送 | date+1 | | ⬜ | |
| 1.4 | 3 ホップ | legs.length=3 | | ⬜ | |
| 2.1 | 最短選択 | 中継ルート選択 | | ⬜ | |
| 2.2 | 運行外曜日 | 422 | | ⬜ | |
| 2.3 | 期限切れ接続 | 422 | | ⬜ | |
| 2.4 | 接続重複 | transit_days=0が選択 | | ⬜ | |
| 3.1 | status 全遷移 | 各 status 反映 | | ⬜ | |
| 3.2 | picked_up後 cancel | 422 | | ⬜ | |
| 3.3 | pending cancel | 200 | | ⬜ | |
| 3.4 | 二重 cancel | 422 | | ⬜ | |
| 4.1 | 全サイズ | 5件作成 | | ⬜ | |
| 4.2 | 不正サイズ | 422 | | ⬜ | |
| 4.3 | 容量超過 | 警告のみ | | ⬜ | |
| 5.1 | テナント分離 | 0件 | | ⬜ | |
| 5.2 | テナント無効化 | 拒否 | | ⬜ | |
| 6.1 | キーなし | 401 | | ⬜ | |
| 6.2 | 不正キー | 401 | | ⬜ | |
| 6.3 | JWT取得 | 3セグメント | | ⬜ | |
| 6.4 | JWT認証 | 200 | | ⬜ | |
| 7.1 | 冪等 | 重複なし | | ⬜ | |
| 8.1 | probe実行 | post=10 fail=0 | | ⬜ | |
| 8.2 | status全13値 | 全変換成功 | | ⬜ | |
| 8.3 | size全5値 | 全変換成功 | | ⬜ | |
| 8.4 | StopType全16通り | unmappable=0 | | ⬜ | |
| 9.1〜9.10 | 異常系 | 各 4xx | | ⬜ | |
| 10.1〜10.3 | 性能 | 目標達成 | | ⬜ | |
| 11.1 | レグ連鎖 | 全cancelled | | ⬜ | |
| 11.2 | 再ルーティング | 仕様通り | | ⬜ | |

---

## 13. テスト結果の連絡

不具合を見つけたら **GitHub Issue** へ：

```
リポジトリ：https://github.com/VegibusIT/meguru-manual
（または MEGURU 本体リポジトリ）

タイトル：[bug] Test X.Y - 簡潔な症状
本文：
- Test 番号:
- 環境: ローカル / Fly.io shadow
- 期待:
- 実測:
- ログ抜粋:
- 再現コマンド:
```

連絡先：

- M2Labo 安河内（company.yasukochi@gmail.com）
- 緊急時：[08_operator_guide.md#緊急停止手順](08_operator_guide.md#緊急停止手順)

---

次：トラブルシュート・運用は [08_operator_guide.md](08_operator_guide.md)。
