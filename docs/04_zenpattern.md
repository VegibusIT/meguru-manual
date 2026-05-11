# 04. 全パターン操作練習

📖 [目次](README.md) ｜ 前：[03](03_dekiru_koto.md) ｜ 次：[05](05_komatta.md)

このページは **自分の手で MEGURU の全パターンを 1 個ずつ確認** するページです。
営業として「お客様にこんなケース聞かれた → こう動きます！」と即答できるようになるのが目的。

🎥 **動画候補**：各パターンを 30 秒〜1 分のショート動画にして、index 化（撮影後 `assets/videos/V7-*.mp4`）

---

## ⓪ 練習の準備（毎回これだけ）

ターミナルで以下を 1 回だけ実行（変数を設定）：

```bash
# 会社（テナント）を作る
TENANT=$(curl -s -X POST http://localhost:3000/admin/tenants \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"練習用","plan":"starter"}' | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

echo "会社ID = $TENANT"
```

> 💡 失敗したら：[05. 困ったとき](05_komatta.md#サーバが動かない) を参照。

---

## ① 直行配送（一番カンタン）

**シナリオ**：農家A から お店B に直接配送。

```bash
# バス停を 2 つ作る
A=$(curl -s -X POST "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"A","address":"千葉","latitude":35.6,"longitude":140.1,"stop_type":"collection","capacity_cases":100}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

B=$(curl -s -X POST "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"B","address":"東京","latitude":35.7,"longitude":139.8,"stop_type":"delivery","capacity_cases":50}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

# つながりを登録
curl -X POST "http://localhost:3000/connections?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"from_stop_id\":\"$A\",\"to_stop_id\":\"$B\",\"days_of_week\":[\"mon\",\"tue\",\"wed\",\"thu\",\"fri\"],\"transit_days\":0,\"active_from\":\"2026-01-01\"}"

# 注文
curl -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":3,\"container_size\":\"medium\",\"external_order_id\":\"P1\"}"
```

**確認**：返ってきた JSON の中の `"legs":[ ... ]` が **1 個** であること（直行なので 1 区間）。

✅ 確認できた

---

## ② 中継経由（積み替え配送）

**シナリオ**：農家A → ハブ → お店B（直行便なし）

```bash
# 中継ハブを追加
X=$(curl -s -X POST "http://localhost:3000/stops?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"X","address":"千葉","latitude":35.65,"longitude":140.04,"stop_type":"transit","capacity_cases":500}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

# A→X、X→B のつながりを追加
curl -X POST "http://localhost:3000/connections/bulk?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"connections\":[
    {\"from_stop_id\":\"$A\",\"to_stop_id\":\"$X\",\"days_of_week\":[\"mon\",\"tue\",\"wed\",\"thu\",\"fri\"],\"transit_days\":0,\"active_from\":\"2026-01-01\"},
    {\"from_stop_id\":\"$X\",\"to_stop_id\":\"$B\",\"days_of_week\":[\"mon\",\"tue\",\"wed\",\"thu\",\"fri\"],\"transit_days\":0,\"active_from\":\"2026-01-01\"}
  ]}"

# 新しいテナントを作って試すか、A→B 直行のつながりを消して再注文
```

**確認**：直行のつながりを消したテナントで実行すると `legs` が **2 個** になる。

✅ 確認できた

---

## ③ 翌日配送

**シナリオ**：夕方集荷、翌朝配達。

「ハブで一泊して翌日配達」のつながりは `transit_days: 1` で表現：

```bash
curl -X POST "http://localhost:3000/connections?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"from_stop_id\":\"$X\",\"to_stop_id\":\"$B\",\"days_of_week\":[\"tue\",\"wed\",\"thu\",\"fri\",\"sat\"],\"transit_days\":1,\"active_from\":\"2026-01-01\"}"
```

**確認**：注文を入れると、2 区間目の `scheduled_date` が翌日になる。

✅ 確認できた

---

## ④ 経路が複数あるとき「短い方」を選ぶ

**シナリオ**：A→B直行（1日）と A→X→B中継（0日）を両方登録。

→ MEGURU は **中継経由（0日）の方** を選ぶ。`legs` が 2 個になる。

✅ 確認できた

---

## ⑤ その曜日に運行していない便

**シナリオ**：火曜にしか走らない便を、月曜の注文に使おうとする。

→ エラー（HTTP 422 / "no path found"）が返る。

```bash
# わざと曜日外で注文
curl -i -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-11\",\"cases\":1,\"container_size\":\"small\",\"external_order_id\":\"P5\"}"
```

期待：`HTTP/1.1 422` と `"error":"no path found"` メッセージ

✅ 確認できた

---

## ⑥ 期限切れの便

**シナリオ**：4月末で廃止のつながりに、5月の注文を入れる。

→ エラー。

✅ 確認できた

---

## ⑦ ステータスを進める

> 注：DB を直接いじる方法。実運用ではドライバーアプリの操作で進みます。

`pending` → `confirmed` → `picked_up` → `in_transit` → `delivered`

各段階で `GET /shipments/<id>` すると最新状態が見える。

✅ 確認できた

---

## ⑧ キャンセルできる／できない

| 状態 | キャンセル？ |
|---|---|
| `pending` | ✅ できる |
| `confirmed` | ✅ できる |
| `picked_up` 以降 | ❌ できない（HTTP 422） |

**やってみる**：

```bash
# 注文 → すぐキャンセル
SID=$(curl -s -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":1,\"container_size\":\"small\",\"external_order_id\":\"P8\"}" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

curl -i -X PATCH "http://localhost:3000/shipments/$SID/cancel" -H 'X-API-Key: dev-noauth'
# 期待: HTTP 200
```

二度目のキャンセル試行：HTTP 422（既にキャンセル済）

✅ 確認できた

---

## ⑨ コンテナサイズ 5 種類

```bash
for size in small medium large xlarge xxlarge; do
  curl -s -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
    -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
    -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":1,\"container_size\":\"$size\",\"external_order_id\":\"P9-$size\"}" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print(d['container_size'], d['id'])"
done
```

期待：5 件全部できる。

✅ 確認できた

---

## ⑩ 不正なコンテナサイズ

```bash
curl -i -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":1,\"container_size\":\"巨大\",\"external_order_id\":\"P10\"}"
```

期待：HTTP 4xx で拒否

✅ 確認できた

---

## ⑪ お客様ごとの分離

会社2 を別に作って、会社1 のバス停は会社2 から見えないことを確認。

```bash
T2=$(curl -s -X POST http://localhost:3000/admin/tenants \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d '{"name":"練習用2","plan":"starter"}' | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

# 会社2 でバス停一覧 → 0 件のはず
curl "http://localhost:3000/stops?tenant_id=$T2" -H 'X-API-Key: dev-noauth'
# 期待: []
```

✅ 確認できた

---

## ⑫ 認証なしでアクセス

```bash
curl -i "http://localhost:3000/stops?tenant_id=$TENANT"
# 期待: HTTP 401
```

✅ 確認できた

---

## ⑬ 同じ注文番号で二度入れる（冪等性）

```bash
# 1回目
curl -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":1,\"container_size\":\"small\",\"external_order_id\":\"DUP-001\"}"

# 2回目（同じ注文番号）
curl -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":1,\"container_size\":\"small\",\"external_order_id\":\"DUP-001\"}"
```

期待：二重作成されない（同じ id が返る、もしくは 409）

✅ 確認できた

---

## ⑭ 出発地と到着地が同じ

```bash
curl -i -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$A\",\"scheduled_date\":\"2026-05-13\",\"cases\":1,\"container_size\":\"small\",\"external_order_id\":\"P14\"}"
# 期待: HTTP 422
```

✅ 確認できた

---

## ⑮ ケース数 0 や マイナス

```bash
curl -i -X POST "http://localhost:3000/shipments?tenant_id=$TENANT" \
  -H 'Content-Type: application/json' -H 'X-API-Key: dev-noauth' \
  -d "{\"origin_stop_id\":\"$A\",\"destination_stop_id\":\"$B\",\"scheduled_date\":\"2026-05-13\",\"cases\":0,\"container_size\":\"small\",\"external_order_id\":\"P15\"}"
# 期待: HTTP 422
```

✅ 確認できた

---

## ⑯ やさいバス本番データでテスト（特別編）

> 注：これは **安河内のみが実行できる** モードです。営業の方は読み飛ばして OK。

やさいバス Aurora MySQL から MEGURU にデータをコピーして検証する `probe` というツールがあります。

千葉エリア（バス停 158 件）の取り込みテストをワンショットで実行：

```bash
# 安河内が自分の PC で実行
cd /data/m2labo/meguru
SHADOW_PWD=$(cat /data/m2labo/.meguru_shadow.pwd) \
DATABASE_URL='postgres://meguru:meguru_dev@localhost:5432/meguru' \
YASAIBUS_DATABASE_URL="mysql://meguru_shadow:${SHADOW_PWD}@127.0.0.1:13307/vegibus_new" \
PROBE_BUS_AREA_ID=32 PROBE_TENANT_ID='8d94ed36-f5d5-4584-8dfb-b9f46d679a41' \
MEGURU_API_BASE_URL='http://127.0.0.1:3000' MEGURU_API_KEY='dev-noauth' \
RUST_LOG=info cargo run --bin meguru-bridge-probe
```

期待される結果：

📸 ![probe実行](assets/captures/02_probe_full_run.png)

- バス停 **158 件** 取得成功
- 注文ステータス変換成功（やさいバスの 13 種類 → MEGURU の 7 種類）
- コンテナサイズ変換成功（1〜5 → small〜xxlarge）
- POST 成功 10 件、失敗 0 件

✅ 確認できた

---

## 結果記録テンプレ

| # | 何を試した | 期待 | 実測 | OK? |
|---|---|---|---|---|
| ① | 直行 1 区間 | legs=1 | | ⬜ |
| ② | 中継 2 区間 | legs=2 | | ⬜ |
| ③ | 翌日配送 | 日付+1 | | ⬜ |
| ④ | 経路短い方 | 中継選択 | | ⬜ |
| ⑤ | 曜日違い | 422 | | ⬜ |
| ⑥ | 期限切れ便 | 422 | | ⬜ |
| ⑦ | ステータス進行 | 全段階 OK | | ⬜ |
| ⑧ | キャンセル可否 | 状態通り | | ⬜ |
| ⑨ | 全サイズ 5 種 | 5 件作成 | | ⬜ |
| ⑩ | 不正サイズ | 4xx | | ⬜ |
| ⑪ | 会社の分離 | 0 件 | | ⬜ |
| ⑫ | 認証なし | 401 | | ⬜ |
| ⑬ | 同番号 2 回 | 重複なし | | ⬜ |
| ⑭ | 同じ出発・到着 | 422 | | ⬜ |
| ⑮ | ケース 0 | 422 | | ⬜ |
| ⑯ | やさいバス取込 | 158 件 | | ⬜ |

全部 OK になったら、お客様に MEGURU を **自信を持って紹介** できます ✨

---

## つぎ

➡️ **[05. 困ったとき・よくある質問](05_komatta.md)** に進む
