# JVMコンポーネント設計案 v0.2

> **設計候補であり、実装済みAPIではない。** [コーディング規約](CODING_RULES.md)の具象継承禁止・final強制・オーバーロード禁止・移譲・1クラス/1関数1責務・ネスト制限・カプセル化を守る。時間・Stateの厳密な意味は[ランタイム実行契約](RUNTIME_PROTOCOL.md)が優先する。

## 1. 概念ごとに責務を分ける

**ゲーム世界の実体の数と計算オブジェクトの数を一対一に対応させない。** 住民100万IDが存在してもよいし、必要なら住民100万オブジェクトがあっても構わない。ただし移動の基本式や境界判定を100万回重複実行しない。

| 責務候補 | 入力 | 出力/副作用 |
| --- | --- | --- |
| `EntityStateStore` | ID、状態更新指示 | IDの唯一の論理State・所有区間・世帯などをカプセル化 |
| `GroupMembership` | GroupKey、ID集合の移管指示 | IDの派生所属索引を管理、分割・統合 |
| `MovementPredictor` | 区間の共有規則、入場時刻/個体差 | 到達時刻・共有移動モデル（判断・保存なし） |
| `GroupArrivalIndex` | 集団の到着時刻、ID差分 | 最早到着バッチだけを返す |
| `EventQueue` | 時刻・microRound・phase・安定sequence | 予約、失効、時刻順の取り出し |
| `BoundaryResolver` | MoveRequest、信号・競合・定員Snapshot | Allowed / Transition / Wait / Invalidの判断 |
| `CapacityLedger` | 区間/施設/競合領域の利用要求 | 所有者による原子的予約・確定・解放 |
| `TransitionCommitter` | BoundaryDecision、現在の版番号 | 正規State・集団・容量の一括確定 |
| `RoutePlanner` | 有向グラフ・経路条件 | 区間列と次境界、通行不能なら失敗 |
| `WorldProjector` | 世界の読み取りSnapshot・カメラ関心情報 | UI向けCityViewを生成（世界を変更しない） |

これは**実装クラスを必ず10個作るという意味ではない**。最初のサンプルでは責務を分割可能な契約として設計し、統合可能なものは不要な抽象化を作らない。パフォーマンスのために変わるのは内部表現で、責務の境界ではない。

## 2. Kotlin/JavaのAPI例（スケッチ）

以下は役割の方向性を示す**非実行のAPI断片**。ビルドできるフルコードではない。

```kotlin
@JvmInline
value class EntityId(val raw: Long)

@JvmInline
value class GroupId(val raw: Long)

data class ArrivalHead(
    val groupId: GroupId,
    val scheduledAt: Long,
    val groupRevision: Long
)

interface GroupArrivalIndex {
    fun findEarliest(groupId: GroupId): ArrivalHead?
    fun replaceArrivals(change: ArrivalIndexChange)
}

interface BoundaryResolver {
    fun decide(request: MoveRequest, view: BoundarySnapshot): MoveDecision
}

interface GroupMembership {
    fun transferMembers(command: TransferCommand): TransferReceipt
}

interface EventScheduler {
    fun reserve(event: ScheduledEvent): EventReceipt
    fun invalidate(token: EventToken)
}
```

- 実装は `final class`（または言語仕様で明示不能な最終型）にする。具象実装の継承は禁止。
- 同名関数を引数違いで定義しない。`reserve(event)`と`reserveBatch(events)`のように違う目的には別名を与える。
- `GroupMembership` などが管理する大量のIDの配列は `private` とし、可変配列をAPI経由でそのまま公開しない。
- ホットパスのID保持には `LongArray` 等の候補を検証してよいが、外部APIの型安全性を犠牲にする必要はない。
- 責務同士を**具象クラスの継承**で結び付けず、`interface` への依存と移譲を使う。
- `ArrivalIndexChange`、`MoveRequest`等は契約を示す仮名。実装で命名・入出力形を確定する際は用途を優先する。

## 3. 実行シーケンス

```text
EventQueue: dueを取り出す
   ↓
GroupArrivalIndex: 到着IDのバッチを選択
   ↓
BoundaryResolver: 信号/State/容量に対する判断
   ↓
CapacityLedger: 許可分の資源を予約
   ↓
TransitionCommitter: 正規Stateと集団所属を原子的に移管
   ↓
EventScheduler: 退出/次到着/待機再開を登録
   ↓
WorldProjector: UIへ確定Stateの投影
```

各役割は自分の担当だけを実行する。手順を束ねる`WorldStepRunner`等は許可するが、内部で移動公式・信号判断・保存・描画を直接行う万能クラスにはしない。

**単一の確定主体:** 初期PhaseではProcessの世界状態を一つの決定的スレッドで確定する。JVMの並列処理は独立した予測計算や読み取りSnapshotを対象とする。並列の実行順でIDの入場許可が変わらないよう、コミットは安定sequenceに従う。

## 4. モジュールとUPD境界の候補

```text
:shared-contracts   値/契約の共有のみ（処理層ではない）
:ui                 入力・可視化
:process            都市シミュレーション・コマンド判定
:data               マップ/セーブの取得・保存
:launcher            起動時の依存配線だけ（業務判断を持たせない）
```

依存:

```text
UI ──→ Process ──→ Data
  ↘        ↓         ↙
       Shared contracts
```

- `:ui` は `:data` を直接呼ばない。`:data` は `:ui` や `:process` のゲームルールに依存しない。
- `:process` は保存が必要なときだけDataへリクエストする。共有DTO/値は `:shared-contracts` に置き、DataがProcess実装型をimportすることを避ける。
- `:launcher` は依存配線だけを行い、UPDの第四の**処理層**にしない。
- 実際のGradleプロジェクト名・Kotlin/Javaの割当・JDKバージョンは[#67](https://github.com/tomiya7688/city-builder/issues/67)で検証して確定する。UPD checkerの静的検査範囲を過大評価しない。

## 5. 実装を開始するときの最小順序

1. ID/StateStoreとGroupMembershipの不変条件テスト（#61）。
2. 安定順序イベントキューと時刻進行のテスト（#62）。
3. 3〜5本の有向区間、1つの信号、歩行者/車両の競合領域（#63）。
4. MoveRequestの部分許可・WaitKeyと集団移管のテスト（#64）。
5. CityViewからの可視化のみの補間とカメラ不変性（#65）。
6. 同じseed・入力に対する全件逐次更新と集団処理の結果/性能比較（#66）。

**採用前に残す設計上の可変点:** グループ到着順索引の実データ構造、速度変化時の再アンカーの最適化方式、競合領域の粒度、可視個体数上限、交通モデルの容量単位、使用するJVM言語の役割分担。いずれも実測と正しさのテストで決める。
