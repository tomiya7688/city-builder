# 設計トレーサビリティ — 完成版v1.0

> この表は「**設計された完成品の機能**」と「**実装する作業**」を結び付ける入口。**全Issueは実装未完了であり、この表に載ることは完成を意味しない。** 機能Issue #13〜#60の48件には実装仕様v0.1を記載済みであり、必要な共通ランタイム規則はv0.2以降の正本を優先する。

**実装者向けの正式な詳細設計書:** [CB-SDD（全8章）](design/README.md)。要求IDと試験IDはCB-SDD第1・7章を参照。

## 正本の階層

1. **どんなゲームを作るか:** [GAME_DESIGN](GAME_DESIGN.md)。
2. **遊びの詳細ルール:** [WORLD_CONSTRUCTION](WORLD_CONSTRUCTION.md)、[POPULATION_ECONOMY](POPULATION_ECONOMY.md)、[TRANSPORT_GAMEPLAY](TRANSPORT_GAMEPLAY.md)、[CITY_SERVICES_ENVIRONMENT](CITY_SERVICES_ENVIRONMENT.md)、[PLAYER_EXPERIENCE](PLAYER_EXPERIENCE.md)、[CONTENT_PROGRESSION](CONTENT_PROGRESSION.md)。
3. **調整値と完成ゲート:** [GAME_BALANCE_RELEASE](GAME_BALANCE_RELEASE.md)、[VALIDATION_PLAN](VALIDATION_PLAN.md)。 計算式は[QUANTITATIVE_MODEL](QUANTITATIVE_MODEL.md)。
4. **技術上の動作意味:** [IMPLEMENTATION_SPEC](IMPLEMENTATION_SPEC.md)、[RUNTIME_PROTOCOL](RUNTIME_PROTOCOL.md)、[TRAFFIC_VERTICAL_SLICE](TRAFFIC_VERTICAL_SLICE.md)、[DOMAIN_INTEGRATION](DOMAIN_INTEGRATION.md)。 State遷移の全体一覧は[STATE_EVENT_CATALOG](STATE_EVENT_CATALOG.md)。
5. **JVM技術・必須ルール:** [TECH_STACK](TECH_STACK.md)、[JVM_COMPONENT_DESIGN](JVM_COMPONENT_DESIGN.md)、[CODING_RULES](CODING_RULES.md)、[upd-design](https://github.com/tomiya7688/upd-design)。
6. **作業と合格条件:** 各Issue本文。共通契約と矛盾する場合はIssueだけ独自解釈せず、正本と一緒に改訂する。

## ゲーム完成版の受入チェック

| 必須テーマ | 文書 | エンドツーエンド判定 |
| --- | --- | --- |
| ゲームの開始・育成・運営 | GAME_DESIGN + WORLD_CONSTRUCTION | E01 街が生まれる |
| 道路・歩道・公共交通 | TRANSPORT_GAMEPLAY | E02 交通問題を解く |
| 住民・雇用・企業・税収 | POPULATION_ECONOMY | E01 / E04 財政難から復活 |
| 電力/水/ごみ/医療/教育/治安/消防 | CITY_SERVICES_ENVIRONMENT | E03 停電から復旧 |
| 気象・災害・環境/復興 | CITY_SERVICES_ENVIRONMENT | E05 災害と復興 |
| 建設UI・レイヤー・セーブ・アクセシビリティ | PLAYER_EXPERIENCE | E01〜E05の操作/セーブ/再現 |
| シナリオ・解放・データ拡張 | CONTENT_PROGRESSION | 5シナリオ全実行 |
| 大都市性能と論理整合 | IMPLEMENTATION_SPEC + VALIDATION_PLAN | E06 大都市スケーラビリティ |
| 品質・配布真正性 | GAME_BALANCE_RELEASE + #1 | 全OS試験・署名・attestation |

## 対応Issue一覧

## 地形・マップ

設計文書: [WORLD_CONSTRUCTION.md](WORLD_CONSTRUCTION.md) ・親Issue: [#4](https://github.com/tomiya7688/city-builder/issues/4)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#13](https://github.com/tomiya7688/city-builder/issues/13) | 地形の標高・傾斜と造成 | P1 | Phase 1 |
| [#14](https://github.com/tomiya7688/city-builder/issues/14) | 河川・湖・海と水面の管理 | P2 | Phase 3 |
| [#15](https://github.com/tomiya7688/city-builder/issues/15) | 敷地・土地セル・接道判定 | P1 | Phase 1 |
| [#16](https://github.com/tomiya7688/city-builder/issues/16) | マップ拡張と都市外接続 | P2 | Phase 3 |

## 区画・建物

設計文書: [WORLD_CONSTRUCTION.md](WORLD_CONSTRUCTION.md) ・親Issue: [#5](https://github.com/tomiya7688/city-builder/issues/5)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#17](https://github.com/tomiya7688/city-builder/issues/17) | 住宅・商業・工業・オフィスの用途区画 | P1 | Phase 1 |
| [#18](https://github.com/tomiya7688/city-builder/issues/18) | 用途別の需要モデルと供給不足 | P1 | Phase 1 |
| [#19](https://github.com/tomiya7688/city-builder/issues/19) | 建物の建設・密度・段階的成長 | P1 | Phase 1 |
| [#20](https://github.com/tomiya7688/city-builder/issues/20) | 建物の用途・居住人数・雇用容量 | P1 | Phase 1 |
| [#21](https://github.com/tomiya7688/city-builder/issues/21) | 撤去・建て替え・再開発 | P2 | Phase 2 |

## 道路・歩行・車両

設計文書: [TRANSPORT_GAMEPLAY.md](TRANSPORT_GAMEPLAY.md) ・親Issue: [#6](https://github.com/tomiya7688/city-builder/issues/6)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#22](https://github.com/tomiya7688/city-builder/issues/22) | 道路・車線・高速道路の敷設と更新 | P1 | Phase 1 |
| [#23](https://github.com/tomiya7688/city-builder/issues/23) | 信号・優先順位・右左折の進入制御 | P1 | Phase 1 |
| [#24](https://github.com/tomiya7688/city-builder/issues/24) | 歩道・横断歩道・歩道橋のネットワーク | P1 | Phase 1 |
| [#25](https://github.com/tomiya7688/city-builder/issues/25) | 車両の経路・道路容量・渋滞 | P1 | Phase 1 |
| [#26](https://github.com/tomiya7688/city-builder/issues/26) | 歩行者の経路探索・境界遷移 | P1 | Phase 1 |
| [#27](https://github.com/tomiya7688/city-builder/issues/27) | 駐車場・路上駐車・道路維持 | P2 | Phase 2 |

## 公共交通・貨物

設計文書: [TRANSPORT_GAMEPLAY.md](TRANSPORT_GAMEPLAY.md) ・親Issue: [#7](https://github.com/tomiya7688/city-builder/issues/7)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#28](https://github.com/tomiya7688/city-builder/issues/28) | バス路線・停留所・営業所 | P1 | Phase 2 |
| [#29](https://github.com/tomiya7688/city-builder/issues/29) | 交通手段選択・乗換・乗降の状態遷移 | P1 | Phase 2 |
| [#30](https://github.com/tomiya7688/city-builder/issues/30) | 鉄道・地下鉄・路面電車の路線と駅 | P2 | Phase 2 |
| [#31](https://github.com/tomiya7688/city-builder/issues/31) | タクシー・オンデマンド交通 | P2 | Phase 3 |
| [#32](https://github.com/tomiya7688/city-builder/issues/32) | 空港・港・都市間鉄道の域外旅客交通 | P2 | Phase 3 |

## 都市インフラ

設計文書: [CITY_SERVICES_ENVIRONMENT.md](CITY_SERVICES_ENVIRONMENT.md) ・親Issue: [#8](https://github.com/tomiya7688/city-builder/issues/8)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#33](https://github.com/tomiya7688/city-builder/issues/33) | 発電・送配電・停電と電力需要 | P1 | Phase 1 |
| [#34](https://github.com/tomiya7688/city-builder/issues/34) | 取水・浄水・配水管と上水需要 | P1 | Phase 1 |
| [#35](https://github.com/tomiya7688/city-builder/issues/35) | 下水管・排水・汚水処理 | P1 | Phase 1 |
| [#36](https://github.com/tomiya7688/city-builder/issues/36) | ごみ収集・処分・リサイクル | P1 | Phase 2 |
| [#37](https://github.com/tomiya7688/city-builder/issues/37) | 通信網・携帯基地局・インターネット | P2 | Phase 3 |

## 公共サービス

設計文書: [CITY_SERVICES_ENVIRONMENT.md](CITY_SERVICES_ENVIRONMENT.md) ・親Issue: [#9](https://github.com/tomiya7688/city-builder/issues/9)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#38](https://github.com/tomiya7688/city-builder/issues/38) | 病院・診療所と救急搬送 | P1 | Phase 2 |
| [#39](https://github.com/tomiya7688/city-builder/issues/39) | 小中高校・大学と教育需要 | P1 | Phase 2 |
| [#40](https://github.com/tomiya7688/city-builder/issues/40) | 警察署・犯罪・治安の管理 | P1 | Phase 2 |
| [#41](https://github.com/tomiya7688/city-builder/issues/41) | 火災・消防署・出動 | P1 | Phase 2 |
| [#42](https://github.com/tomiya7688/city-builder/issues/42) | 公園・広場・娯楽と観光スポット | P2 | Phase 2 |
| [#43](https://github.com/tomiya7688/city-builder/issues/43) | 高齢化・死亡・墓地・火葬場 | P2 | Phase 3 |

## 人口・企業・経済・政策

設計文書: [POPULATION_ECONOMY.md](POPULATION_ECONOMY.md) ・親Issue: [#10](https://github.com/tomiya7688/city-builder/issues/10)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#44](https://github.com/tomiya7688/city-builder/issues/44) | 住民・世帯・転入転出・年齢構成 | P1 | Phase 1 |
| [#45](https://github.com/tomiya7688/city-builder/issues/45) | 就労・就学・通勤・生活スケジュール | P1 | Phase 1 |
| [#46](https://github.com/tomiya7688/city-builder/issues/46) | 商店・消費行動・サービス需要 | P1 | Phase 2 |
| [#47](https://github.com/tomiya7688/city-builder/issues/47) | 工業・農業・物流と域外貿易 | P2 | Phase 2 |
| [#48](https://github.com/tomiya7688/city-builder/issues/48) | 市の予算・税率・維持費・赤字 | P1 | Phase 1 |
| [#49](https://github.com/tomiya7688/city-builder/issues/49) | 地価・住民満足度・都市の魅力 | P1 | Phase 2 |
| [#50](https://github.com/tomiya7688/city-builder/issues/50) | 地区の指定と都市政策 | P2 | Phase 2 |

## 環境・気象・災害

設計文書: [CITY_SERVICES_ENVIRONMENT.md](CITY_SERVICES_ENVIRONMENT.md) ・親Issue: [#11](https://github.com/tomiya7688/city-builder/issues/11)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#51](https://github.com/tomiya7688/city-builder/issues/51) | 大気・騒音・土壌・水質汚染 | P1 | Phase 2 |
| [#52](https://github.com/tomiya7688/city-builder/issues/52) | 昼夜・天候・季節・気温 | P2 | Phase 3 |
| [#53](https://github.com/tomiya7688/city-builder/issues/53) | 洪水・地震・暴風などの災害 | P2 | Phase 3 |
| [#54](https://github.com/tomiya7688/city-builder/issues/54) | 地下資源・森林・農地と特化産業 | P2 | Phase 3 |

## 都市経営UI・保存・進行

設計文書: [PLAYER_EXPERIENCE.md](PLAYER_EXPERIENCE.md) ・親Issue: [#12](https://github.com/tomiya7688/city-builder/issues/12)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#55](https://github.com/tomiya7688/city-builder/issues/55) | 道路・区画・施設の建設操作 | P1 | Phase 1 |
| [#56](https://github.com/tomiya7688/city-builder/issues/56) | 都市の情報レイヤー・ヒートマップ・統計 | P1 | Phase 1 |
| [#57](https://github.com/tomiya7688/city-builder/issues/57) | 住民要望・都市警告・問題箇所への誘導 | P1 | Phase 1 |
| [#58](https://github.com/tomiya7688/city-builder/issues/58) | 時間進行・停止・倍速と描画同期 | P1 | Phase 0 |
| [#59](https://github.com/tomiya7688/city-builder/issues/59) | 都市データのセーブ・ロードと互換性 | P1 | Phase 0 |
| [#60](https://github.com/tomiya7688/city-builder/issues/60) | 都市の成長段階・アンロック・実績 | P2 | Phase 3 |

## エンジン基盤・品質

設計文書: [IMPLEMENTATION_SPEC.md](IMPLEMENTATION_SPEC.md) ・親Issue: [#3](https://github.com/tomiya7688/city-builder/issues/3)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#61](https://github.com/tomiya7688/city-builder/issues/61) | 安定EntityIdとGroupMembershipの分割・統合を実装する | P1 | — |
| [#62](https://github.com/tomiya7688/city-builder/issues/62) | SimClockと決定的な離散イベントキューを実装する | P1 | — |
| [#63](https://github.com/tomiya7688/city-builder/issues/63) | 最小NetworkSegmentと境界グラフ・経路探索を実装する | P1 | — |
| [#64](https://github.com/tomiya7688/city-builder/issues/64) | MoveRequest・BoundaryResolver・State遷移と容量を実装する | P1 | — |
| [#65](https://github.com/tomiya7688/city-builder/issues/65) | 可視範囲の詳細描画・物理補正と画面外の論理移動を分離する | P1 | — |
| [#66](https://github.com/tomiya7688/city-builder/issues/66) | 横断歩道の縦断シナリオと1万/10万/100万IDベンチマーク | P1 | — |
| [#67](https://github.com/tomiya7688/city-builder/issues/67) | JVM複数言語のビルド構成とUPD/OOP規約の自動検査 | P1 | — |

## 完成品の追加タスク

設計文書: [GAME_DESIGN.md](GAME_DESIGN.md) ・親Issue: [#12](https://github.com/tomiya7688/city-builder/issues/12)。

| Issue | 作業/機能 | Priority | 既存ロードマップ |
| --- | --- | --- | --- |
| [#68](https://github.com/tomiya7688/city-builder/issues/68) | 3D都市描画・カメラ・インスタンシングとLOD | P1 | — |
| [#69](https://github.com/tomiya7688/city-builder/issues/69) | サンドボックス・シナリオ目標・達成/失敗判定 | P1 | — |
| [#70](https://github.com/tomiya7688/city-builder/issues/70) | 4地形マップ・5シナリオの初期データを制作 | P1 | — |
| [#71](https://github.com/tomiya7688/city-builder/issues/71) | 施設・道路・区画のデータ駆動カタログと安全な追加パック | P1 | — |
| [#72](https://github.com/tomiya7688/city-builder/issues/72) | チュートリアル・建設導線・説明付きヘルプ | P1 | — |
| [#73](https://github.com/tomiya7688/city-builder/issues/73) | 日本語・英語・キー再割当・判読性設定 | P1 | — |
| [#74](https://github.com/tomiya7688/city-builder/issues/74) | 人口・地価・需要・税収・維持費の経済調整を確定する | P1 | — |
| [#75](https://github.com/tomiya7688/city-builder/issues/75) | 都市の環境音・通知音・BGM・音量設定 | P2 | — |
| [#76](https://github.com/tomiya7688/city-builder/issues/76) | 完成版E2E・長時間試験・OS別リリース判定を実施する | P1 | — |

## 設計上の決定・調整・検証の扱い

- **設計で固定すること:** ゲームモード、都市機能、UIからできる操作、ID/Stateの同一性、時間/eventの確定順序、UPD/OOPの必須規約、必要な受入シナリオ。
- **データで調整すること:** 建物/道路の費用・維持費、住戸容量、税率、幸福度の重み、施設処理量、ゲームの暦倍率、難易度。
- **実測して最終確定すること:** 集団到着索引・イベントキューの最適化、JVMと描画ライブラリの版、100万都市での表示密度、実際の必要スペック、FPSとセーブ時間。
- **v1の対象外:** オンラインマルチプレイ、全個体の高頻度な生活物理、任意実行スクリプトMod、広告/課金での建設短縮。

この表と個別仕様が衝突した場合、正本を確認して差分を更新し、Issueに決定理由を残す。設計だけでIssueをCloseしない。
