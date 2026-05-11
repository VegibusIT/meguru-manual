# 99. 用語集・動画撮影リスト

📍 [目次](README.md) ▶ 99. 用語集

---

## 99.1 用語集

### MEGURU 用語

| 用語 | 意味 |
|---|---|
| **MEGURU** | M2Labo の共同配送インフラ SaaS の名前 |
| **テナント (Tenant)** | 運営事業者の単位。1 顧客 = 1 テナント |
| **バス停 (Stop)** | 集荷・配達・中継・車庫の地点 |
| **ストップ接続 (StopConnection)** | バス停間の運行可能性（グラフの辺） |
| **ルート (Route)** | 順序付きバス停の並び。1 つの便（例：朝便） |
| **荷物 (Shipment)** | 1 件の配送依頼 |
| **レグ (ShipmentLeg)** | 荷物の区間。中継があれば複数 |
| **配車 (Dispatch)** | ドライバー × ルート × 日付 の割当 |
| **配車エンジン (Dispatch Engine)** | ダイクストラで経路を選ぶエンジン |
| **RLS** | Row Level Security。テナント分離の仕組み |
| **bridge** | やさいバス→MEGURU の片方向同期コンポーネント |
| **probe** | bridge のスモークテスト用ワンショットバイナリ |

### やさいバス用語 ↔ MEGURU 用語

| やさいバス | MEGURU |
|---|---|
| 注文 (`dtb_order`) | 荷物 (`shipment`) |
| バス停 (`mtb_bus_stop`) | バス停 (`stop`) |
| バス停接続 (`mtb_bus_stop_connect`) | ストップ接続 (`stop_connection`) |
| ルート (`mmtb_route`) | ルート (`route`) |
| エリア (`bus_area_id`) | テナント (`tenant`) |
| 農家 (farmer) | 集荷ストップ (collection) |
| バイヤー (buyer) | 配達ストップ (delivery) |
| 中継 (transit) | 中継ストップ (transit) |
| 車庫 (garage) | ガレージストップ (garage) |
| 注文ステータス (1〜13) | shipment.status (7 値) |
| コンテナサイズ (1〜5) | container_size (small〜xxlarge) |
| order_code | external_order_id（補助）|

### 技術用語

| 用語 | 意味 |
|---|---|
| **Rust** | システムプログラミング言語。MEGURU のバックエンド |
| **axum** | Rust の Web フレームワーク |
| **sqlx** | Rust の非同期 SQL クライアント |
| **petgraph** | Rust のグラフライブラリ。ダイクストラ実装に使用 |
| **PostgreSQL 16** | MEGURU の DB |
| **Aurora MySQL** | やさいバスの DB |
| **Fly.io** | ホスティング先（東京リージョン）|
| **JWT** | JSON Web Token。管理者・ドライバー認証 |
| **bastion** | 踏み台サーバ。やさいバス RDS への SSH トンネル経由地 |
| **shadow** | 本番に影響を与えない検証用環境 |

---

## 99.2 略語

| 略 | 正式 |
|---|---|
| EC | Electronic Commerce |
| RLS | Row Level Security |
| VRP | Vehicle Routing Problem |
| ER | Entity-Relationship |
| API | Application Programming Interface |
| SSH | Secure Shell |
| TLS | Transport Layer Security |
| UUID | Universally Unique Identifier |
| SBPL | SATO Barcode Printer Language（関連プロジェクト用語） |

---

## 99.3 動画撮影ショットリスト

公開マニュアルとしてビデオを撮るときの候補ショット一覧。所要時間はあくまで目安。

| # | 動画 | 章 | 所要 | 撮影内容 |
|---|---|---|---|---|
| V1 | MEGURU 概要 | 01 | 3分 | スライド＋解説（プロダクトビジョン） |
| V2 | クイックスタート完走 | 02 | 7分 | 端末画面録画。テナント作成→荷物作成まで |
| V3 | 管理者：バス停・ルート登録 | 03 | 10分 | 一連の curl + DB 確認 |
| V4 | 荷主：API 連携サンプル | 04 | 7分 | Python サンプルを動かす |
| V5 | ドライバー業務フロー（モック）| 05 | 5分 | 画面モック＋音声解説 |
| V6 | シャドウ検証起動 | 06 | 5分 | SSH トンネル→probe 完走 |
| V7 | テストシナリオ抜粋 | 07 | 各2分 | パターン 1.1〜2.3 をリアル実行 |
| V8 | トラブルシュート例 | 08 | 3分 | わざと壊して直す |

撮影後のファイル配置：

```
docs/manual/assets/videos/
├── V1_overview.mp4
├── V2_quickstart.mp4
├── V3_admin.mp4
├── ...
└── thumbnails/
    ├── V1.png
    └── ...
```

Markdown への埋込：

```markdown
[![V2 クイックスタート](assets/videos/thumbnails/V2.png)](assets/videos/V2_quickstart.mp4)
```

Web 公開時には YouTube limited / Vimeo private のリンクに差し替えても可。

---

## 99.4 キャプチャ画像の取り直しリスト

既存：

| ファイル | 内容 | 取り直し要否 |
|---|---|---|
| `assets/captures/01_health_check.png` | `curl /health` の出力 | 現状で OK |
| `assets/captures/02_probe_full_run.png` | probe フル実行 | 現状で OK |
| `assets/captures/03_meguru_db_stops.png` | DB の stops 内容 | 現状で OK |
| `assets/captures/04_api_list_stops.png` | API での stops 一覧 | 現状で OK |
| `assets/captures/05_ssh_tunnel.png` | SSH トンネル＋docker | 現状で OK |

追加候補（撮影してほしい）：

| ファイル | 内容 | 用途 |
|---|---|---|
| `06_tenant_create.png` | テナント作成 | 03 |
| `07_shipment_create_2legs.png` | 中継 2 レグ荷物作成 | 02, 04 |
| `08_shipment_cancel.png` | キャンセル成功 | 04, 07 |
| `09_shipment_cancel_denied.png` | picked_up 後キャンセル拒否 | 07 |
| `10_route_register.png` | ルート登録 | 03 |
| `11_connection_bulk.png` | 接続一括登録レスポンス | 03 |
| `12_login_jwt.png` | JWT 取得 | 09 |
| `13_db_er_diagram.png` | ER 図のスクショ | 10 |
| `14_admin_dashboard_mock.png` | 管理画面モック | 03 |
| `15_driver_app_mock.png` | ドライバーアプリモック | 05 |

撮影スクリプトは `docs/captures/Makefile` 化すると再生成しやすい（任意）。

---

## 99.5 参考リンク

- MEGURU リポジトリ：（社内）
- マニュアル公開リポジトリ：https://github.com/VegibusIT/meguru-manual
- 製品ビジョン：[docs/product_vision.md](../product_vision.md)
- アーキテクチャ仕様 v4.1：[MEGURU_architecture_spec_v4.1.md](../../../MEGURU_architecture_spec_v4.1.md)
- bridge カラム対応表：[docs/bridge_mapping.md](../bridge_mapping.md)

---

## 99.6 マニュアル変更履歴

| 日付 | 版 | 変更 |
|---|---|---|
| 2026-05-12 | 1.0 | 役割別マニュアル初版（11 ファイル構成） |
| 2026-05-04 | 0.1 | 旧 MANUAL.md（シャドウ検証のみ） |

---

📍 [目次に戻る](README.md)
