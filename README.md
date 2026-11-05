# City Builder

JVM 系言語を組み合わせて開発する、都市規模の拡大に強い街づくりシミュレーションゲーム（仮称）。

## 目標

Cities: Skylines に近い街づくり体験を目指しつつ、人口や都市面積が増えたときにシミュレーション負荷が急増しにくい構造を、最初から設計する。

核心は、**個体の同一性は ID で保持し、同じ条件の計算を集団で共有する**こと。個体を単なる人数に置き換えず、誰がいつどこに到達し、どの進路を選ぶかを追えるようにする。

- **ID と共有計算**: 各住民・車両は ID を持つ。状態・区間・移動条件が一致する計算は共有する。
- **State 遷移と移動リクエスト**: 曲がり角、横断歩道、十字路、歩道橋、駅など、判定が必要な境界で状態を切り替える。
- **移動計算と判定の分離**: 区間上の移動をまとめて予測し、進入許可や通行条件は境界で判断する。
- **イベント駆動**: 信号が青になる、通行可能になる、列車が到着するといった変化で待機集団を再開する。
- **距離に応じた時間的 LOD**: 遠方では次の判定地点まで数秒以上をまとめて計算する。途中の重要イベントは飛ばさない。
- **描画範囲内だけ詳細計算**: 可視範囲では細かな位置・アニメーション・必要な局所相互作用を扱い、画面外は共有計算とイベントで進める。

これは設計方針であり、現時点で性能優位を実測・保証したものではない。

## 技術方針

- JVM 系言語を併用する。初期の採用案は Java 21 + Kotlin、Gradle Kotlin DSL、libGDX / LWJGL3 のデスクトップ3D構成。具体的なバージョンと描画性能はプロトタイプで検証して固定する。
- オブジェクト指向で責務・契約・移譲を明確化する。
- 多数の ID と最低限の状態は、必要に応じてプリミティブ配列等のコンパクトな表現で保持する。
- 描画とシミュレーションの周期を分離する。
- UI・Process・Data の責務を [UPD Design](https://github.com/tomiya7688/upd-design) に従って分離する。

## ドキュメント

### 正式な詳細設計書（実装者向け）

- **[Software Design Description — 詳細設計書（全8章）](docs/design/README.md)** — 要求・システム構成・データ・計算アルゴリズム・都市機能統合・API/保存・試験・設計判断。

### ゲーム内容の完成形

- [完成版ゲーム設計 v1.0](docs/GAME_DESIGN.md) — 遊び方・完成範囲・ゲームモード・進行方針
- [設計トレーサビリティ](docs/DESIGN_TRACEABILITY.md) — 全都市機能・エンジン・品質タスクと正本の対応
- [地形・建設・ゾーニング](docs/WORLD_CONSTRUCTION.md) — マップ・道路/歩道・用途区画・建物成長
- [住民・経済・財政](docs/POPULATION_ECONOMY.md) — 住民/世帯・通勤・産業・消費・税収
- [定量モデル](docs/QUANTITATIVE_MODEL.md) — 人口・需要・建物成長・供給・会計の初期計算式
- [交通と公共交通](docs/TRANSPORT_GAMEPLAY.md) — 自動車・徒歩・バス・鉄道・地下鉄・物流
- [都市インフラ・公共サービス・環境](docs/CITY_SERVICES_ENVIRONMENT.md) — 電力・上下水・医療・災害
- [UI・操作・保存](docs/PLAYER_EXPERIENCE.md) — 建設UI・情報レイヤー・日英対応・セーブ/ロード
- [コンテンツ・アンロック・シナリオ](docs/CONTENT_PROGRESSION.md) — 施設カタログと5本のシナリオ
- [バランス・性能・完成判定](docs/GAME_BALANCE_RELEASE.md) — 数値・品質ゲート・E2Eシナリオ

### エンジン・実装規約

- [シミュレーション設計](docs/SIMULATION_DESIGN.md) — ID 集団・State・移動要求・横断歩道・時間的 LOD・整合性
- [共通実装仕様](docs/IMPLEMENTATION_SPEC.md) — 共通モデル・Stateとイベント・UPD境界・正しさの契約
- [State・イベント一覧](docs/STATE_EVENT_CATALOG.md) — 交通・住民・施設・財政・災害の遷移と待機解除
- [ランタイム実行契約](docs/RUNTIME_PROTOCOL.md) — ID正規状態・最早到着バッチ・同時刻イベント・予約失効
- [JVM技術構成](docs/TECH_STACK.md) — Java 21・Kotlin・Gradle・libGDX/LWJGL3の検証方針
- [JVMコンポーネント設計案](docs/JVM_COMPONENT_DESIGN.md) — 小さな責務・final実装・API契約・UPDモジュール構成
- [交通・横断歩道の縦断仕様](docs/TRAFFIC_VERTICAL_SLICE.md) — 歩行者/車の待機と進入・交差点の競合処理
- [都市システム統合仕様](docs/DOMAIN_INTEGRATION.md) — 電力/水/ごみ/税/住民などの共有率計算と状態変化
- [検証計画](docs/VALIDATION_PLAN.md) — 正しさと大規模性能の比較条件
- [実装ロードマップ](docs/IMPLEMENTATION_ROADMAP.md) — Phase 0〜3、依存関係と実証シナリオ
- [機能バックログ（Issues）](https://github.com/tomiya7688/city-builder/issues) — 都市機能48件とコア実装・検証Issue
- [開発・オブジェクト指向ルール](docs/CODING_RULES.md) — 継承・オーバーロード・final・責務・ネスト・カプセル化・UPD
- [AGENTS.md](AGENTS.md) — コード生成やレビューでも参照する開発上の入口

## 現状

**完成品のゲームデザインをv1.0として具体化した設計段階**。ゲーム内容・都市機能・画面・シナリオ・数値の初期値・完成判定は文書化済み。都市機能48件はIssueごとに状態・共有計算・イベント・依存・受入条件を記載済み。完成品に必要な補足タスクもIssue #68〜#76として整理済み。実装・性能測定・QAは未完了であり、まずPhase0の基盤Issue #61〜#67から最小のState遷移エンジンを実証する。

ライセンスその他の公開条件は未定。
