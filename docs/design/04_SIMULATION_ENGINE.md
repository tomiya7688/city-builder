# 4. 共通計算・イベント・State遷移エンジン（CB-SDD-04）

> 本章はPhase0の**処理アルゴリズムと確定順序**の詳細設計。[RUNTIME_PROTOCOL](../RUNTIME_PROTOCOL.md) と [STATE_EVENT_CATALOG](../STATE_EVENT_CATALOG.md) を満たす。実際の索引方式と性能は #61〜#66 で比較して確定する。

## 4.1 計算を起動する条件

常時全IDに`update()`する方式は基本にしない。次の4種類を区別する。

| 計算の種類 | 例 | 起動 |
| --- | --- | --- |
| 区間移動の予測 | 歩道を歩く、道路を走る、駅間移動 | 区間進入時、移動ルール変更時 |
| 境界・State判定 | 曲がり角、横断歩道、入場、改札 | **最早到着イベント**、待機解除イベント |
| 率の積分・周期集計 | ごみ蓄積、税の基礎、産業生産、汚染 | 条件変更、予定閾値到達、日次/週次締め |
| 可視化 | 車のアニメ、住民の歩行、ヒートマップ | UIの描画フレーム（正規Stateは変更しない） |

「5秒先を予測する」は**次の必須判定を飛ばさずに実行を休止する**意味。5秒刻みに全員が同時移動する意味ではない。

## 4.2 集団と最早到着索引

### 入力

- `GroupKey`: 対象種、移動State、SegmentId、出口BoundaryId、移動ルール版、必要な条件区分。
- `GroupMembers`: Group内のEntityId集合。
- 各IDの`enteredAt`、現在進捗アンカー、個体差の速度係数、次境界/進路。
- Segmentの長さ・速度・混雑/制約revision。

### 共有式と個別到達

線形モデル（初期採用案）：

```text
t0 = enteredAt_i
p0 = anchorProgress_i           // 0..1
v  = sharedSpeed(segment, ruleVersion) * individualFactor_i
remainingDistance = length(segment) * (1 - p0)
arrivalAt_i = t0 + ceilMs(remainingDistance / v)
p_i(t) = clamp(p0 + v*(t-t0)/length, 0, 1)
```

`v <= 0`なら到達時刻を作らず、速度/通行条件の解除イベントで待機させる。丸めは「物理的に到達するより早い時刻に到着済みとしない」方向（到着時刻の切り上げ）に統一する。ミリ秒オーバーフロー/負値はエラーとする。

同じ移動計算式を共有しても、IDごとに`enteredAt`や`individualFactor`が違うため到達時刻は異なる。

### 二段階の予約

```text
Group A:
  ID=101 arrival=5s
  ID=102 arrival=6s
  ID=103 arrival=7s
  -> GroupArrivalIndexには3つの到着候補
  -> グローバルEventQueueにはGroup Aの「最早到着5s」だけ
```

グループ内部の到着候補を`(arrivalAt, EntityId)`で順序管理し、グローバルキューへは原則**先頭時刻の到着バッチ**を1つ登録する。5秒の到着で101だけを移管し、102/103は到着済みにせず、6秒の次イベントを予約する。

### GroupArrivalIndexの契約

| 操作 | 入力 | 出力/不変条件 |
| --- | --- | --- |
| `addMembers` | ID集合と算出済み到達時刻 | 重複IDなし、先頭が変わるならGroup版を更新 |
| `peekEarliest` | GroupId | 最早時刻、同時刻のID集合、版 |
| `removeArrived` | GroupId、正当化済みID集合 | 対象だけ取り除き、次の先頭を得る |
| `reanchorGroup` | 条件変更時刻・旧/新ルール | 変更時刻の進捗から新しい到達時刻を算出 |
| `split/merge` | GroupKey/ID集合 | 集合保存、Groupの先頭を再予約 |

内部の採用候補：ソート配列、min-heap、時刻バケット＋安定したID順。各方式のメモリ/再予約/挿入コストをベンチマークで比較する。**グローバルイベントが1件でもGroup内の管理がO(1)とは限らない**。

## 4.3 EventQueueと論理時間

イベントの並び順は以下の辞書式順序で固定する。

```text
(scheduledAtMs, microRound, phase, stableSequence)
```

`scheduledAtMs`は単調増加の論理時刻（Long、ミリ秒）。現在時刻より過去のイベント登録は禁止。phaseの初期割当は0入力、10退出と容量解放、20信号/設備/待機解除、30進入・移管、40統計/表示投影。

**microRound**は同一時刻の位相を逆行する場合に増加させるための数字。phase30中にphase20のイベントを生成した場合、同時刻`microRound+1`のphase20として扱う。無制限に増えないようラウンドと同一IDの同時刻遷移数に検査上限を設け、異常を診断する（上限値は計測後の設定値）。

### Processのイベントループ（擬似コード）

```text
advanceTo(targetMs):
  assert targetMs >= world.now
  while queue.hasEventAtOrBefore(targetMs):
    event = queue.popNext()
    if event.isCanceled or !versionsMatch(event): continue
    world.now = event.scheduledAt
    proposal = domainHandler.prepare(event, readOnlyWorld)
    commitProposalAtomically(proposal)
    enqueueDerivedEventsInStableOrder(proposal)
  world.now = targetMs
```

`prepare`が無効ID・版違い・競合で失敗した場合は確定しないで適切な再評価結果にする。例外時に**半分だけ移動済み**という状態を残さない。

イベントの`stableSequence`はProcessが入力順と決定済みルールに従って採番する。並列Workerの終了順を反映してはいけない。

### 1xとゲーム内暦

既存ゲーム設定では`5,400,000 SimClock ms = ゲーム内1日`。UIの1x/2x/4x/8xは**SimClockをどこまで進めるか**を変え、キューにある途中イベントは必ず順に確定させる。実装負荷が追いつかない場合は論理イベントを飛ばさず、表示する時間倍率を実際の処理可能速度へ制限する。

## 4.4 MoveRequestと結果の契約

一回の要求は以下の情報を持つ。

```text
MoveRequest:
  requestId             一意
  entityIds             重複のない対象集合
  sourceLocation        現在占有する区間/施設
  destinationLocation   次の区間/施設
  boundaryId            進入境界
  requestedAt           SimClock時刻
  expectedState         要求元のState
  expectedStateRevision 競合/古い予約の判定
  routeRevision         使用した経路の版
```

結果はID集合を**相互排他的に**分ける：

```text
MoveDecision:
  allowedIds[]      // 進入/容量が承認されたID
  transitionIds[]   // State変更が必要なIDと次State
  waitingIds[]      // WaitKeyと理由を持つID
  invalidIds[]      // 存在/所属/経路が矛盾するIDと理由
```

`union(allowed, transition, waiting, invalid) == requestedIds`、各集合の交差は空。`Transition`は再要求のための通常制御、`Wait`は信号/容量/競合などの解除条件待ち、`Invalid`は想定外の誤った要求を示す。

**State切替と通過の分離**：歩道を歩いているIDが横断歩道に入ろうとすると、現在Stateが適合しないので一度`CrosswalkWaiting`へ変更する。この時点では横断歩道の占有を増やさない。次の許可判定で青/容量/競合を通過したとき初めて`CrosswalkWalking`となり所有区間を変更する。

## 4.5 容量と競合の調停

**別物として管理する容量**：

1. **滞在容量**：区間/車両/病院/建物内で同時に占有できる数。
2. **入場容量**：単位時間または信号位相中に通過できる数。
3. **排他領域**：車/歩行者の競合など、同時許可できない区間の組み合わせ。

申請は`CapacityLedger`に対し「事前の読み取り→予約トークン確保→確定/解放」を原子的に実行する。個別の数量は施設の意味に応じて人数、車両、座席、住戸、重量など型で区別する。**共有Ledgerが建物・横断歩道のゲームルールを同じにするわけではない**。

### 複数要求の決定順序

- ルールによる優先（交差点で横断中の歩行者、明示した緊急車両優先など）。
- 要求が成立した時刻。
- 安定したsequence、同条件ではEntityId。

初期実装の確定は単一スレッド。確定時は元区間の所有・先区間の予約・排他領域の占有を**まとめて**変更し、二つの要求が同じ最後の空きを取得しない。

部分許可が出た場合、残りIDを待機群へ分割。待機群は`CapacityReleased`等を購読し、解除時刻に同じ境界で再評価する。

## 4.6 待機群の購読と取りこぼし防止

`WaitKey(targetId, reason, conditionRevision)`に対してID集団/待機順を登録する。

- 赤信号→`SignalGreen`。
- 横断歩行者の占有→`ConflictZoneCleared`。
- 道路満員→`SegmentCapacityReleased`。
- バス/列車満員→`TripCapacityReleased`または次便到着。
- 施設閉鎖→`ServiceRestored`または代替施設候補の発生。

**読み取りと購読登録を同じProcess確定境界で行う。** 状態を見てから登録するまでに解除イベントを見失う「lost wakeup」を作らない。待機登録時点で条件が既に成立していれば、同時刻の次microRoundに再評価を予約する。

解除イベントは**全員を無条件に許可する命令ではなく、再試行の契機**。起動しても定員が足りないIDは引き続き待機する。永久待機は低頻度の診断イベントで検出し、原因と改善手段をUIへ表示する。

## 4.7 再アンカーと予測失効

区間長L、進捗p0、速度v0で移動中のIDが、時刻tcでv1へ変更された例：

```text
pAtChange = clamp(p0 + v0*(tc - anchorAt) / L, 0, 1)
newArrival = tc + ceilMs(L*(1-pAtChange)/v1)
```

**旧ルールでtcまで進んだ地点を確定**し、そこで新ルールへ切り替える。次の交差点への到達が早まる場合も遅れる場合も新しい予定を予約し、旧イベントの版を失効する。

例：L=100m、t=0にv0=1m/sで開始、t=30sでv1=2m/sへ変更。変更時に30m進んでおり残り70mだから、到着時刻は`30+35=65s`。旧予定100sを放置してはいけない。

**失効の影響索引**：

- SegmentId → 移動中GroupId/ID
- BoundaryId → 待機GroupId/WaitKey
- RouteRevision → 影響TripId
- UtilityComponentId → 関係BuildingId

速度や通行条件が変更された**影響集団だけ**再アンカーする。索引更新自体が重い場合は集団/区間単位でバッチ化するが、重要な確定時刻を越えて処理を遅らせない。

キャンセル済イベントが優先度キューに溜まり続ける問題を防ぐため、失効割合や実要素数を監視し、安定順序を維持した再構築（compaction）を実装候補に含める。

## 4.8 横断歩道の完全な例

条件：歩行者101/102/103がt=5sに入口へ、歩行信号t=7s青、同時占有2人、横断所要5秒、t=13s赤。車201がt=8sに停止線、t=14sに車用青。

| 時刻 | トリガー | 処理と次State |
| --- | --- | --- |
| 5s | EntranceArrival | 101/102/103を`SidewalkWalking→CrosswalkWaiting`。赤なので信号購読 |
| 7s | SignalGreen | 容量2により101/102だけ`CrosswalkWalking`、103はWait |
| 8s | CarArrival | 競合領域が占有中なので201は`WaitingForPedestrian` |
| 12s | PedestrianExit | 101/102が歩道へ、領域容量解放。まだ青なので103が横断開始 |
| 13s | SignalRed | 新しい横断開始は止めるが、103は継続 |
| 14s | CarSignalGreen | 車201は競合領域が空くまで待機 |
| 17s | PedestrianExit | 103が歩道へ。空き通知と車信号で201の進入を再評価 |

原子確定後の状態：歩行者3IDは変化なく有効、横断者0、車201は未承認のまま交差領域へ入らない。到着時刻が5/6/7sに分かれる場合は各自の到着時刻の順に`CrosswalkWaiting`へ追加する。

## 4.9 失敗時の扱い

| 事象 | 処理 |
| --- | --- |
| 参照先ID不存在 | `Invalid`、世界変更なし。原因IDと参照を診断 |
| 旧revisionの到着イベント | 破棄/再計算、現在のStateを変更しない |
| 満員/信号赤 | `Wait`、解除条件を購読 |
| 同時刻の循環遷移 | 最大microRound/遷移数を超えたら**不変条件違反**として診断・テスト失敗。黙って成功扱いしない |
| 進路閉鎖 | 影響IDの現在進捗を保持し、待機・安全な退避・再探索 |
| グリッドロック | 区間の相互待機を検出し警告/再経路探索候補。突然テレポートしない |
| EventQueueの過負荷 | 時間倍率を処理可能な速度へ制限し、イベントをスキップしない |
| Workerの提案が失効 | version照合で破棄し、必要なら再計算。Workerが直接書込まない |

## 4.10 性能モデルとテスト

目指す計算量は「毎フレームの全ID更新」に依存させないこと。実コストは**集団数、到着イベント数、分割・統合、キャッシュ失効、可視表現数**に依存する。一人ずつ異なる進路/時刻なら最悪でID数に近づくため、理論上の一定時間を保証しない。

必須試験：到着5/6/7秒、信号赤→青→赤、1000人対容量100、車と歩行者の競合、早着/遅延の再アンカー、同時刻microRound、信号変更を見逃さない、キャンセルキューの肥大、カメラ不変、保存/再開の一致。詳細は [VALIDATION_PLAN](../VALIDATION_PLAN.md) と本書第7章を参照。
