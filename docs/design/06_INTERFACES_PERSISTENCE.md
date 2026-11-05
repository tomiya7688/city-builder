# 6. API・保存形式・障害回復（CB-SDD-06）

> 本書は「他のモジュールが使う契約」「セーブできる正規状態」「異常時にどこまで確定するか」を定義する。UPDの責務境界は [CODING_RULES](../CODING_RULES.md)、イベント時刻とStateは [RUNTIME_PROTOCOL](../RUNTIME_PROTOCOL.md) に従う。

## 6.1 外部に公開するインターフェース

### UI→Process：コマンド契約

すべてのコマンドには、次の共通情報を持たせる。

| フィールド | 型/意味 | 保証 |
| --- | --- | --- |
| `commandId` | 入力要求を一意に識別する値 | 再送時の重複適用を避ける |
| `kind` | コマンド種別 | 未対応種別は明示拒否 |
| `expectedWorldRevision` | UIが確認した世界の版 | 古いプレビューが競合した場合は再評価を要求 |
| `payload` | 操作ごとの不変値 | UI部品、GPU/DBハンドルを含めない |
| `requestedGameTime` | 必要なときだけ希望する論理時刻 | 実際の確定時刻はProcessが指定 |
| `source` | Human / Tutorial / Replay等の識別 | 再現テストで入力列を検証 |

### コマンドの種類（v1）

| 種別 | 主なパラメータ | Processの判断 |
| --- | --- | --- |
| `CreateWorld` | マップID、seed、難度、左右通行、都市名 | 設定値と地形データの検証 |
| `PlaceRoad` | 道路テンプレートID、始終点、曲線、標高オプション | 接続・費用・既存物との衝突 |
| `ZoneArea` | 対象セル集合、用途、密度 | 土地適性、既存構造への影響 |
| `PlaceFacility` | テンプレートID、位置、向き | 資金・接道・地形・供給制約 |
| `Demolish` | 対象EntityId、撤去条件 | 現在の占有・移転計画・費用 |
| `ChangeTaxPolicy` | 用途/地区、税率、有効時刻 | 税率範囲・施行時刻・既存期間の積算 |
| `ChangeServiceBudget` | 部門、予算率 | 予算制約・施設稼働への反映 |
| `ConfigureTransitLine` | 停留所列、車両数、運行間隔、料金 | 接続・定員・車庫の利用可否 |
| `SetSimulationSpeed` | Pause/1x/2x/4x/8x | 時刻の進行設定。予約を飛ばさない |
| `SaveWorld` | セーブスロット識別、表示名 | スナップショットを作る時点と保存依頼 |
| `LoadWorld` | セーブスロット識別 | Data読込、検証、世界の置換可否 |

コマンドは**一つのルール上の確定単位**として扱う。道路敷設の一括操作を部分確定するなら、個々の部分コマンドと料金・依存関係を事前にUIへ明示する。v1初期は一コマンドの全成功/全拒否を基本とする。

### コマンドの結果

- `Applied(commandId, committedAt, worldRevision, affectedIds/regions)`：確定済。
- `Rejected(commandId, reasonCode, details)`：不成立。世界変更なし。
- `Conflict(commandId, currentRevision, changedRegions)`：古い状態を根拠に要求。プレビューを再取得。
- `Pending(commandId, operationId)`：セーブ/ロード等の非同期作業を受理。最終結果を後でProcessから通知。
- `Completed/Failed`：非同期要求に対する最終結果。**Pendingだけで成功表示しない**。

異常は機械可読な理由コードで扱う（例：`INSUFFICIENT_FUNDS`, `NOT_CONNECTED`, `INVALID_TERRAIN`, `CAPACITY_FULL`, `STALE_REVISION`, `CONTENT_MISSING`, `SAVE_IO_FAILURE`）。UIはこれをローカライズして表示し、判定条件の複製を持たない。

### コマンドJSON（契約の概念例。実際の外部通信形式とは限らない）

```json
{
  "commandId": "cmd-000042",
  "kind": "PlaceRoad",
  "expectedWorldRevision": 881,
  "payload": {
    "templateId": "road.local.two_lane",
    "pointsMeters": [[100, 0, 120], [180, 0, 120]],
    "elevationMode": "GROUND"
  }
}
```

成功なら`Applied`が道路区間IDと新revisionを返す。接続・地形・料金のチェックに失敗した場合は`Rejected`で、道路も支出も変えない。

## 6.2 Process→UI：読み取り契約

UIはProcessへ次の**問い合わせ**を出し、時刻・版付きの読み取り専用値を受け取る。

| Query | 返る値 | 応答に含める版 |
| --- | --- | --- |
| `GetCitySummary` | 人口、残高、収支、需要、満足度、シミュレーション日時 | `worldRevision`, `simTimeMs` |
| `GetRegionLayer` | 区域別交通量、電力/水、地価、汚染等の集約値 | `layerRevision`, `regionRevision` |
| `GetVisibleEntities` | 表示用ID、現在区間、位置アンカー、State、外観テンプレート | `worldRevision`, `simTimeMs` |
| `GetEntityDetails` | 住民/車両/建物のIDに基づく状態、関係先、問題理由 | `entityRevision` |
| `GetBuildPreview` | 設置可否候補、見積費用、干渉、失敗理由 | `worldRevision`, `mapRevision` |
| `GetAlerts` | CauseId、重大度、影響範囲、解決済フラグ | `alertRevision` |

`GetVisibleEntities`の結果を得るためにUIが全IDの実体を走査することを要求しない。Process側の空間索引/区間別可視抽出やSnapshotで返す。UIで見た目の補間は可能だが、都市の税・住民数・移動所有・交差点の許可を変更してはならない。

**スナップショットに混ざる時刻は禁止。** あるCityViewの交通量・人口・残高には同じ論理的な確定境界を示す版と時刻を付ける。非同期の別の差分を後から合成する場合は、対応する版を検証する。

## 6.3 Process→Data：保存/取得のポート

Processは`WorldStoragePort`のような**値の型契約**でDataを呼ぶ。DataへProcess内部の可変WorldStateや画面部品を渡さない。

| 要求 | 入力 | 結果 |
| --- | --- | --- |
| `ReadMapData` | mapId, contentVersion | 不変の地形/水域/外部接続の入力またはFormatError |
| `ReadContentCatalog` | packIds/versions | テンプレート・スキーマ・アセット参照またはMissingContent |
| `WriteWorldSnapshot` | slotId, immutable Snapshot, metadata | SaveCompleted/IOFailed/ValidationFailed |
| `ReadWorldSnapshot` | slotId | Snapshot/VersionMismatch/Corrupted/IOFailed |
| `ListSaveSlots` | フィルタ | スロット名、日時、サイズ、互換性の事実 |

Dataはスキーマ・CRC・ファイルアクセスの事実を検査する。**世帯を二重収容しているか、車が赤信号で進んだか等のゲームルールの検証はProcess**で行う。

## 6.4 セーブのスナップショット境界

**同一SimClock時刻のイベント処理が矛盾なく完了した境界**でProcessが不変Snapshotを切り出す。phase/microRoundの途中で一部のみ保存しない。

- 新規コマンドの取込みはスナップショット確定の前後どちらか一方に位置づけ、境界を越えて半分だけ適用しない。
- Snapshotはその時点のEntityId/State、建物/グラフ、容量予約、待機購読、未完了イベント、乱数・会計等を含む。
- Dataへの受渡し後にProcessがSnapshotの内容を書き換えない。コピー・copy-on-write・不変構造の方式はメモリ/性能を計測して選定。
- UIに対しては`Pending`→`Completed/Failed`の二段階で通知。I/O作業中の都市進行を必要以上に止めない。

## 6.5 保存ファイルの構造（v1初期採用案）

`*.citysave`は**バージョン付きチャンクコンテナ**とする。Java標準のObject SerializationでJVMインスタンスグラフをそのまま保存する方式は使わない。圧縮方式とバイナリ実レイアウトは実装時に固定するが、以下の**論理スキーマ**は必須。

```text
Header
  magic = "CTBD"
  formatMajor / formatMinor
  engineRulesVersion
  saveId / worldId / createdAtUtc / simTimeMs
  worldSeed / nextEntityId / nextEventSequence
  cityCalendarScale / difficulty / trafficSide
  contentPackIds + versions + hashes
  chunkDirectory(offset, length, type, checksum)

Chunks
  TERRAIN_AND_ZONES        地形・水域・資源・敷地と用途
  NETWORK_GRAPH           道路/歩道/鉄道/電気/水道の接続・revision
  BUILDINGS_AND_FACILITIES BuildingId・テンプレート・稼働・定員
  CITIZENS_AND_HOUSEHOLDS CitizenId・HouseholdId・住所/活動
  MOVEMENTS_AND_TRIPS     所有区間・進捗・目的地・到着予測
  TRANSIT_AND_LOGISTICS   便/乗客ID・貨物・在庫
  UTILITIES_ENVIRONMENT   供給/需要・汚染・災害
  FINANCE_AND_POLICIES    取引元帳・税率・地区政策
  EVENTS_AND_WAITS        未処理イベント・待機購読・sequence・revision
  CAPACITY_RESERVATIONS   容量の予約・確定・使用量
  RANDOM_AND_PROGRESSION  RNG状態/カウンタ・シナリオ/アンロック
```

1. すべてのID/数字は単位とエンディアン/符号/範囲を定義する。上記はまず**論理構造**であり、バイナリオフセットや圧縮方式は未確定。
2. 各チャンクに版・長さ・チェックサムを持たせ、部分破損を検出。CRC等は**偶発的破損の検知**であり署名/攻撃耐性を意味しない。
3. 格納順を安定化して再現性を支える。ただし内部の派生GroupIdは正規世界の識別子として永続保証しない。
4. 派生索引・CityView・描画proxyは保存必須でない。保存する場合はロード時に正規データと版を照合し、不一致なら再構築。
5. 未来のイベントと待機購読は、**再構築に必要な情報**を失わない。単に現在時刻・現在の数値だけを保存すると、横断待機や列車到着が再開しない。

## 6.6 安全な書き込み・ロード

### Save

1. ProcessがSnapshotを確定し、Dataへ渡す。
2. Dataが**同じファイルシステム上の一時ファイル**にヘッダー/チャンクを書込。
3. 長さ・チェックサム・目次整合を検証し、可能な環境ではflush/fsyncを実施。
4. 元のセーブをバックアップ/世代管理してから、サポートされる場合は原子的renameで置換。
5. renameの原子性が利用できない環境では、安全なバックアップと復旧検査を行い、不可なら保存失敗を返す。
6. 成功時だけDataResult(Completed)を返す。失敗時は旧データ/バックアップを保護。

### Load

1. Dataがヘッダー・形式版・チャンク長・チェックサム・必要コンテンツの存在を検証。
2. Processが**現行WorldStateと分離した一時World**に全正規データを復元。
3. Entity参照、定員/乗車/元帳の保存則、各State、未完了イベント、予約版、待機購読を検証。
4. 派生Groupや逆引き・表示キャッシュを再構築する。
5. すべて成功してから**確定境界で現行世界を置き換える**。不合格なら元の都市へ戻り、UIへ理由を表示。

**互換性方針:** formatMajorが異なるセーブは明示的な変換器がない限り拒否。minor差は定義済のマイグレーション経路とテストがある場合だけ受入。任意コードを実行するModはv1非対象。欠落した必須施設テンプレートを黙って別の施設に置換しない。

## 6.7 セーブと乱数の再現性

Snapshotは`worldSeed`に加え、利用した擬似乱数の必要状態/カウンタ、イベントsequence、適用ルール/テンプレートの版を保持する。個体差は`(worldSeed, EntityId, purpose, occurrence)`等の決定的キーに基づく方式を優先。

リプレイでは入力列（CommandId、確定時刻、確定順）を記録する。UIのクリック時刻やOSスレッドの完了順を論理イベントの順に使用しない。ゲーム本体の更新でルールが変化した場合、同じセーブを読めても**異なるルール版で完全同結果は保証できない**ため、再現性は同一ルール版を対象にする。

## 6.8 失敗・復旧・観測

| 失敗 | 責務 | UIに返す意味 |
| --- | --- | --- |
| 地形/施設の設置条件不成立 | Process | どの条件/区画が問題か、未確定 |
| IDや予約revisionが古い | Process | 再取得・再評価が必要。世界の半更新なし |
| Data書込/圧縮失敗 | Data→Process | 保存失敗、旧世代を保持 |
| ファイル破損 | Data→Process | 読込不可・バックアップを確認 |
| 保存版/コンテンツ不一致 | Dataが事実検出、Processが扱い判断 | 対応版・不足コンテンツを表示 |
| レンダリングデバイス復旧 | UI | リソースの再生成。都市の論理State不変 |
| イベントの矛盾・循環 | Process | 診断、実行停止/安全な復元導線。黙ってIDを消さない |

ログに個別の住民情報が必要な場合はIDと原因・EventIdなど最小限にし、無制限に全IDの行動を出力しない。性能計測のログI/OはDataアダプタへ委譲する。

## 6.9 受入試験

- PlaceRoadが資金不足なら道路も残高も変わらない。同じCommandIdで二度請求されない。
- 古いworldRevisionでの確定がConflictを返し、現在のプレビューへ更新できる。
- セーブ中に時間が進んでもスナップショットの時刻/数値が揃い、保存操作がProcessを書き換えない。
- ディスク不足、途中電源断、壊れたチャンク、未知formatMajor、不足packで元セーブ/現行都市が守られる。
- 信号待機者、走行車、列車、未完了税イベントを含む状態を保存→ロード→再開して同じ結果を得る。
- UIを描画しないヘッドレスでも、コマンドと保存/ロードのProcess契約をテストできる。
