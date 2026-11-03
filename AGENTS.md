# Agent / Contributor Instructions

City Builder にコードや設計文書を追加・編集する際は、このファイルと下記の正本を必ず読む。

- [設計の正本](docs/SIMULATION_DESIGN.md)
- [共通の実装契約](docs/IMPLEMENTATION_SPEC.md)
- ランタイム・イベント/予約・集団の実装時: [ランタイム実行契約](docs/RUNTIME_PROTOCOL.md)
- JVMの型/インターフェース/モジュールの設計時: [コンポーネント設計案](docs/JVM_COMPONENT_DESIGN.md)
- 横断歩道・道路・車両/歩行者の実装時: [交通の縦断仕様](docs/TRAFFIC_VERTICAL_SLICE.md)
- 人口・建物・インフラ・経済の実装時: [都市システム統合仕様](docs/DOMAIN_INTEGRATION.md)
- テスト・ベンチマーク時: [検証計画](docs/VALIDATION_PLAN.md)
- [実装順・依存関係](docs/IMPLEMENTATION_ROADMAP.md)
- [機能ごとの実装仕様（Issue #13〜#60）](https://github.com/tomiya7688/city-builder/issues)
- [コーディング規約の正本](docs/CODING_RULES.md)
- [UPD Design の必須境界](https://github.com/tomiya7688/upd-design/blob/main/docs/developers/boundaries.md)

## 絶対に守る規約

1. 継承可能な型は **abstract class または interface のみ**。具象クラスの継承禁止。
2. 自作メソッド／関数／コンストラクタの **オーバーロード禁止**。override と混同しない。
3. すべての具象クラスを **final** にする（言語仕様上 final を明示できない型は除く）。
4. **継承より移譲**。1 クラス 1 責務、1 関数 1 処理。
5. **関数内の制御フローネストは 2 段まで**。早期 return、処理の分割を使う。
6. **カプセル化必須**。可変状態を外部へ直接公開しない。
7. **UPD の厳守**。UI は表示、Process は判断と計算、Data は取得・保存。UI と Data の直接通信禁止。イベント・共通部品を使って迂回することも禁止。
8. 画面外の個体を毎フレーム詳細更新しない。**ID を保持しながら共通計算、State 遷移、到達イベントで処理**する。
9. **区間内の移動計算と、境界の進入判定を分離**する。横断前に信号を判定し、通常の横断途中には再判定しない。
10. **遠方の更新は粗くしてよいが、重要なイベントを飛ばさない**。可視範囲の変更で論理結果が変わらないようにする。

## 変更を提案するとき

- 変更先の UPD 層、クラスと関数の責務を説明する。
- 追加した State・イベント・リクエストの発生条件、分割する ID、再開条件を明記する。
- 変更対象Issueにある状態/ID、共有計算、イベント、依存、検証シナリオを参照し、共通実装契約と該当する詳細仕様との差異を確認する。
- ID の一意性、集団の分割・統合、通行容量、到達時刻、カメラ切り替えの整合性を確認する。
- 新しいテストを提案・実行し、実行していないテストは実施済みと書かない。
- 不明な技術選定は既成事実にせず、設計上の候補として扱う。

この文書は要約であり、詳細や衝突時の判断は上記の正本を優先する。
