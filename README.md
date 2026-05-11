# MEGURU 操作マニュアル

**version**: 1.0
**最終更新**: 2026-05-12
**対象読者**: MEGURU を運用・テスト・開発するすべての人
**前身ドキュメント**: [docs/MANUAL.md](../MANUAL.md)（シャドウ検証編のみ）／ [docs/MEGURU_system_overview.md](../MEGURU_system_overview.md)

---

## このマニュアルの目的

MEGURU をはじめて触る人でも、**このマニュアルだけ読めばすべてを理解・操作・テストできる** ことを目指したドキュメント集です。

役割別に分割しているので、自分に必要な章から読んでください。やさいバスチームがテストで踏むべき全パターンは [07_test_scenarios.md](07_test_scenarios.md) に集約しています。

---

## 5秒で MEGURU を理解する

```
┌─────────────┐    ┌─────────────────┐    ┌─────────────┐
│ 注文システム │───→│  MEGURU         │───→│ 配送現場     │
│ やさいバス   │    │  共同配送       │    │ ドライバー   │
│ JA / EC      │    │  インフラ SaaS  │    │ 集荷・配達   │
└─────────────┘    └─────────────────┘    └─────────────┘
```

- M2Labo は **ソフトウェアだけ** を提供（車・人は持たない）
- 運営事業者（テナント）が物流資産を持って自分の地域で運用
- Rust + PostgreSQL + Fly.io（東京リージョン）

詳細は [01_overview.md](01_overview.md) を読んでください。

---

## マニュアル構成

| # | ファイル | 対象 | 内容 |
|---|---|---|---|
| 00 | **README.md**（このファイル）| 全員 | 目次・読む順序 |
| 01 | [01_overview.md](01_overview.md) | 全員 | MEGURU とは何か。30分でつかむ全体像 |
| 02 | [02_quickstart.md](02_quickstart.md) | 全員 | ローカルで起動して荷物を1件流すまで |
| 03 | [03_admin_guide.md](03_admin_guide.md) | テナント管理者 | バス停・ルート登録、ドライバー管理、料金設定 |
| 04 | [04_shipper_guide.md](04_shipper_guide.md) | 荷主・外部システム | API キー取得から荷物作成・追跡まで |
| 05 | [05_driver_guide.md](05_driver_guide.md) | ドライバー | 現場業務フロー（β：開発中含む）|
| 06 | [06_shadow_testing.md](06_shadow_testing.md) | M2Labo 内部 | やさいバス本番→MEGURU シャドウ検証の運用 |
| 07 | [07_test_scenarios.md](07_test_scenarios.md) | **やさいバス担当者** | **全テストパターン（このマニュアルの肝）** |
| 08 | [08_operator_guide.md](08_operator_guide.md) | DevOps | 起動・監視・トラブルシュート・緊急停止 |
| 09 | [09_api_reference.md](09_api_reference.md) | 開発者・SIer | 全エンドポイント仕様 |
| 10 | [10_data_model.md](10_data_model.md) | 開発者 | DB スキーマ・テーブル関連図 |
| 99 | [99_glossary.md](99_glossary.md) | 全員 | 用語集（やさいバス語 ↔ MEGURU 語） |

---

## 読む順序（役割別）

### やさいバスでテストする人 — まずここから

1. [01_overview.md](01_overview.md) で何を作っているか把握（15分）
2. [02_quickstart.md](02_quickstart.md) でローカル起動を体験（30分）
3. [07_test_scenarios.md](07_test_scenarios.md) で全パターンをテスト（半日〜1日）
4. 詰まったら [08_operator_guide.md](08_operator_guide.md) のトラブルシュート

### テナント運営者（将来）

1. [01_overview.md](01_overview.md)
2. [03_admin_guide.md](03_admin_guide.md)
3. [99_glossary.md](99_glossary.md)

### 荷主・SIer 連携担当

1. [01_overview.md](01_overview.md)
2. [04_shipper_guide.md](04_shipper_guide.md)
3. [09_api_reference.md](09_api_reference.md)

### M2Labo 内部開発者

1. [01_overview.md](01_overview.md)
2. [10_data_model.md](10_data_model.md)
3. [09_api_reference.md](09_api_reference.md)
4. [06_shadow_testing.md](06_shadow_testing.md)
5. [08_operator_guide.md](08_operator_guide.md)

---

## キャプチャ・図版・動画について

- **画像** は [assets/captures/](assets/captures/) （実体は `docs/captures/`）に集約。 各章から相対パスで参照しています。
- **動画** の撮影候補ショットは [99_glossary.md#動画撮影ショットリスト](99_glossary.md#動画撮影ショットリスト) に列挙。撮影後 `assets/videos/` に置いて差し替えてください。
- **Web 公開** する場合は [meguru-site/](../../../meguru-site/) に `manual/` パスとしてマウントするか、VitePress / MkDocs で静的サイト化できる構成です。詳細は [08_operator_guide.md#マニュアルのweb公開手順](08_operator_guide.md#マニュアルのweb公開手順) を参照。

---

## ステータス凡例

このマニュアル内で各機能の実装状況を以下の記号で表します：

| 記号 | 意味 |
|---|---|
| ✅ | 実装済・本番品質 |
| 🟡 | 設計済・部分実装 |
| ⚠️ | スタブ／未実装 |
| 🔴 | 注意・破壊的操作 |
| 🎥 | 動画撮影候補 |
| 📸 | キャプチャ既存／取得候補 |

---

## 問い合わせ

- M2Labo 安河内（company.yasukochi@gmail.com）
- 緊急時はやさいバス bastion を切るのが最速：[08_operator_guide.md#緊急停止手順](08_operator_guide.md#緊急停止手順)

---

## 履歴

| 日付 | 版 | 変更 |
|---|---|---|
| 2026-05-12 | 1.0 | 役割別マニュアル初版。やさいバス担当者向け全パターンテスト章を追加 |
| 2026-05-04 | 0.1 | 旧 MANUAL.md（シャドウ検証のみ） |
