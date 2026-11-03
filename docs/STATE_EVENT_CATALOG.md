# State・イベント・境界リクエスト一覧 v1.0

> **設計上の状態遷移契約（未実装）**。共通のID・時刻・イベント順・容量予約の正本は [RUNTIME_PROTOCOL](RUNTIME_PROTOCOL.md)。この文書は各都市機能のState名、トリガー、分割・合流先、待機解除、異常処理をまとめ、実装ごとに独立した挙動へ分岐するのを防ぐ。

## 1. Stateの種類を1つに混ぜない

一つの住民に「通勤中」「社会人」「自動車所有」「病気」など複数の性質があっても、それらを一つの巨大なState列挙へ詰めない。

- **LifecycleState:** 生存、転出予定、死亡、転出済。
- **ActivityState:** 在宅、通勤、勤務、通学、買物、休暇など、その時刻の主要行動。
- **MobilityState:** 道路/歩道/横断/乗車/待機など、移動の仕方と現在の所有区間。
- **ServiceState:** 治療・在学・居住のようなサービス所属。
- **WorldResourceState:** 建物/車両/設備の運転、利用可、故障、撤去など。

**各State体系ごとに唯一の正規所有者と登録先を決める。** 同じCitizenIdが移動に関して同時に徒歩区間と車両の乗客席へ存在することは許さない。一方、その市民が「仕事あり」「学生」など別領域の状態を同時に持つことは正常。

Stateの実体表現は状態コード＋必要なコンテキスト（現在区間、次境界、待機理由、目的地など）を基本にし、全交差点・全車線を固有クラスへしない。

## 2. 共通のイベントエンベロープ

すべての確定イベントは、可能な範囲で次の情報を持つ：

| 項目 | 目的 |
| --- | --- |
| EventId | 確定/取消を識別 |
| scheduledAt | SimClockのLongミリ秒 |
| microRound / phase / stableSequence | 同時刻の決定的処理順 |
| eventKind | 到達/条件変更/予約解除/施設完了などの種別 |
| originDomain / CauseId | どの確定変更が発生源か追跡 |
| target | EntityId・GroupId・SegmentId・BuildingId・RegionId等 |
| expectedRevision | 古い予測/旧Stateによる二重更新を拒否するため |
| payload | 型が明確な最小限の差分・理由・数量 |

共通結果:

- **Allowed:** 要求した区間/施設の容量予約・所有変更が確定可能。
- **Transition:** 対象Stateが違うので適切な次Stateへ切り替える（正常な制御フロー）。
- **Wait:** 容量・信号・目的地・稼働条件が揃わず、ConditionKeyに購読して休止。
- **Invalid:** 元のID/所有区間・経路等が無効。世界を変更せず、必要ならエラーを表す。

同じリクエストの全IDはこれらの結果集合のどれか一つにだけ入り、合計集合が元の要求集合に一致する。

## 3. 交通State一覧

### 歩行者・横断

| 現在State | 起点 | 次State | 重要な判定と待機解除 |
| --- | --- | --- | --- |
| SidewalkWalking | 予定区間の入口に到着 | SidewalkWalking（次区間へ） | 通過可能な歩道接続と有効Route |
| SidewalkWalking | 横断歩道へMoveRequest | CrosswalkWaiting | 現State不一致による正常なTransition |
| CrosswalkWaiting | 信号青・進入容量・競合許可 | CrosswalkWalking | SignalGreen / CapacityReleased / ConflictCleared |
| CrosswalkWaiting | 信号赤、満員、優先交通あり | CrosswalkWaiting | WaitKeyを登録して休止、毎フレーム判定しない |
| CrosswalkWalking | 横断終了境界へ到達 | SidewalkWalking | 信号の通常赤は途中判定せず、出口の状態切替 |
| SidewalkWalking / CrosswalkWaiting | 経路閉鎖 | RouteWaiting | RouteInvalidatedを受けて再探索する |
| RouteWaiting | 代替Route確定 | SidewalkWalking / 別の移動State | 経路/現在区間の所有を一度だけ更新 |

### 車両・道路

| 現在State | 起点 | 次State | 判定・解除条件 |
| --- | --- | --- | --- |
| Driving | SegmentArrivalDue | Driving（次区間） | SignalAllowed / BoundaryCapacity / ConflictZone |
| Driving | 境界で赤/競合 | WaitingForSignal / WaitingForPedestrian | SignalPhaseChanged / ConflictCleared |
| Driving | 下流区間満員 | QueuedForExit | CapacityReleased（現在区間の占有を維持） |
| QueuedForExit | 下流空き | Driving | 原子的な入場・退出容量移管 |
| Driving / Waiting | 通行止め・ネットワーク改訂 | Rerouting | RouteInvalidated・再アンカー |
| Rerouting | 新しい経路あり | Driving / 適切なWaitState | 状態移管と古い予約の失効 |
| Driving | 目的地到着・駐車可能 | Parked | ParkingCapacityReserved |
| Driving | 駐車できない | SearchingParking | ParkingSpaceReleased / AlternativeModeSelected |

車の物理的な加速・車線変更の一部は近距離表現でもよいが、**境界進入・容量・信号・到着時刻**を描画側で確定しない。

### 公共交通

| 移動State | 起点 | 次State | 解除条件・保存するID |
| --- | --- | --- | --- |
| WalkingToStop | 駅/停留所境界到達 | WaitingForService | CitizenId、StopId、RouteTripId |
| WaitingForService | 指定便到着、定員あり | Boarding | BoardingRequested / SeatReserved |
| WaitingForService | 満員/欠便 | WaitingForService / Rerouting | CapacityReleased / NextTripDue / ServiceCancelled |
| Boarding | 乗車処理完了 | Riding | CitizenIdがVehicleIdの定員を一つ占有 |
| Riding | 降車駅・停留所へ到着 | Alighting | 乗客本人の降車予定・StationArrivalDue |
| Alighting | 降車処理完了 | WalkingToDestination / Transferring | 乗車枠を解放し、駅/歩道へ移動 |
| Transferring | 別路線のホーム到着 | WaitingForService | 次のTripId・待機容量 |
| AnyTransitState | 便の運行中止 | Rerouting / WaitingForService | 旧便予約を解除して代替候補を探す |

一人のCitizenIdは同時に複数の車両席を保持しない。バス/列車が見えなくても到着イベントで処理する。

## 4. 建物と住宅・企業

| 現在State | 起点 | 次State | 連携 |
| --- | --- | --- | --- |
| ZonedParcel | 需要と建築適性が成立 | ConstructionPlanned | ParcelIdの建設予約 |
| ConstructionPlanned | プレイヤー/事業者が工事を確定 | UnderConstruction | 費用と工期の予約 |
| UnderConstruction | ConstructionDue | Operating | 建物の入居/雇用容量を有効化 |
| Operating | 世帯/企業が入居 | Operating | 収容容量と所属IDの原子的予約 |
| Operating | 入居・売上不足 | Vacant / Degraded | DemandChanged / ServiceChanged |
| Operating | 条件継続・アップグレード可 | Upgrading | 新しい容量は完工後に有効 |
| Upgrading | UpgradeDue | Operating | BuildingCapacityChanged |
| Vacant / Degraded | 改善または長期放置 | Operating / Abandoned | 条件継続日数・維持費 |
| AnyUsable | DemolitionRequested | RelocationRequired / Demolishing | 住民と企業の移転先予約、経路/需要失効 |
| Demolishing | DemolitionDue | Removed | ID参照・敷地占有・容量解放 |

**状態と適性の区別:** Operatingでも停電・断水で実稼働率は0〜低下し得る。全状態を1つの列挙へ合成せず、設備サービス状態と建物ライフサイクルを組み合わせる。

## 5. 住民・世帯・学業・医療

| Domain | Stateの進行 | 主なEvent |
| --- | --- | --- |
| 世帯入居 | SearchingHome → ReservationPending → Resident → MovingOut | HousingRequested、HomeReserved、MoveInDue、RelocationDue |
| 雇用 | Unemployed → Applying → Employed → Unemployed/Retired | JobVacancyChanged、JobAssigned、JobClosed、RetirementDue |
| 教育 | Unenrolled → EnrollmentWaiting → Enrolled → Graduated / Transferred | EnrollmentDue、SchoolSpaceAvailable、TermEndDue |
| 医療 | Healthy → CareRequested → WaitingForCare / TravelingToCare → UnderTreatment → Healthy / Transferred | IllnessOccurred、BedReserved、AmbulanceArrivalDue、TreatmentDue |
| ライフサイクル | Alive → DeathPending / Outbound → Dead / Outside | BirthdayDue、DeathOccurred、MigrationDeparture |
| 日常活動 | AtHome → Traveling → AtWork/AtSchool/Shopping → ReturningHome | ScheduleDue、TripRequested、TripArrived |

各領域のStateはIDをまたいで集団化可能だが、世帯・仕事・学校・病床・旅行の同時二重所属を禁止する。

## 6. 供給・施設・生産・物流・財政・災害

| Domain | Stateまたはイベント列 | 条件・数量整合 |
| --- | --- | --- |
| 発電/送電 | Operating ↔ Limited/Unavailable → Restoring | GeneratorStopped、PowerRebalanced、PowerLost/Restored。接続成分と供給量の差分 |
| 上水/下水 | Available ↔ Insufficient/Disconnected → Restoring | PipeChanged、PumpStopped、SupplyRebalanced、SewageOverflow。処理量を超えない |
| ごみ | Accumulating → PickupRequested → TruckAssigned → Collected → Disposed | AccumulationThresholdDue、TruckArrivalDue、CapacityReleased。ごみ重量の収支 |
| 通信 | Available ↔ Limited/Offline | TowerStopped、BandwidthChanged、CoverageRecomputed |
| 工場 | StockReady → ProductionReserved → Producing → BatchCompleted → WaitingForInputs | MaterialReceived、ProductionDue、InputShortage。原料/生産/在庫の保存 |
| 物流 | Packed → ShipmentReserved → InTransit → Delivered / Returned / LostByExplicitEvent | TruckAssigned、ShipmentArrived、RoadClosed。貨物を二重在庫へ入れない |
| 財政 | PeriodOpen → Closing → Closed → NextPeriodOpen | DailyCloseDue、TaxPosted、ExpensePosted。TransactionIdとPeriodIdで冪等 |
| 火災 | FireStarted → Reported → ResponseDispatched → Burning / Extinguishing → Extinguished / Destroyed | FireSpreadDue、UnitArrivalDue、ExtinguishDue。影響建物と延焼率を保存 |
| 氾濫/地震等 | Started → Damaging → Evacuating / Recovering → Resolved | DisasterStarted、DamageEvaluated、RecoveryDue。道路/電力等のrevisionを更新 |
| シナリオ | InProgress → Succeeded / Failed → Continued / ReturnedToMenu | GoalProgressChanged、GoalCompleted、FailureReached。達成通知は一度だけ |

数量の確定・消費には各DomainのCapacityLedger/InventoryLedger/FinanceLedgerを使う。共通するのは予約・確定・解放の**仕組み**であり、施設ごとのゲームルールまで同一ではない。

## 7. 同時刻の処理順とDomain通知

初期案の位相:

1. phase 0: 外部コマンド・建設や政策などの変更の入力確定。
2. phase 10: 到着・退出・利用完了・使用済容量の解放。
3. phase 20: 信号・設備条件の変更、供給再計算、待機群への解除通知。
4. phase 30: 進入要求・新予約・容量競合の許可とState移管。
5. phase 40: 確定差分から集約統計・警告・UI用CityViewの更新。

後の位相から先の位相に戻る必要があれば、同じSimClock時刻の次の`microRound`に予約する。無限ループするイベント列は理由付きでテスト失敗とし、黙って一部IDだけ処理を飛ばさない。

## 8. 待機の解除を共通化する

`WaitKey = (targetId, reason, conditionVersion)` をキーに購読集合を管理する。

- 赤信号→`SignalGreen`、交差点の競合→`ConflictCleared`。
- 満員道路/駐車/バス/学校/病院→`CapacityReleased`。
- 電力/上水/通信の停止→`ServiceRestored`。
- 商品・材料不足→`StockReceived`。
- 出勤先/目的地閉鎖→`FacilityReopened` または `RouteReplanned`。

**同じ解除条件を待つIDはまとめて起動できる**。起動した時点で全員許可ではなく、改めて定員や優先度で部分承認する。解除条件が一度も来ない異常な待機は低頻度の診断イベント/警告で発見する。

## 9. 正しさの受入基準

- すべてのIDの所有区間とStateは一意、各予約結果と分割集合は一致。
- 同時刻の信号・退出・進入が再現可能で、二重許可なし。
- 画面外でもState遷移は起き、可視範囲に入ったときに後追いで発生させない。
- 住民の仕事・学校・住所は、徒歩/車/公共交通の移動Stateに引きずられて失われない。
- 設備停止から復旧した際の通知で対象建物だけが更新され、無限な再通知が起きない。
- セーブ/ロード後にイベント予約・待機購読・容量・領域所属を正しく再現できる。
- 各Domainの状態を実装する際に、**この一覧を巨大な継承階層や万能Stateクラスへ翻訳しない**。責務別の小さなクラスとデータに分ける。
