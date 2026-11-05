# 定量モデル v1.0 — 需要・人口成長・建物・供給・都市財政

> **実装可能な計算ルールの初期仕様**。ゲーム上の結果は [POPULATION_ECONOMY](POPULATION_ECONOMY.md)、[CITY_SERVICES_ENVIRONMENT](CITY_SERVICES_ENVIRONMENT.md) を正本とする。時刻・イベント確定順は [RUNTIME_PROTOCOL](RUNTIME_PROTOCOL.md) に従う。各係数・閾値は [GAME_BALANCE_RELEASE](GAME_BALANCE_RELEASE.md) とバランスデータに置き、プレイテストで調整する。数値は未検証の初期案。

## 1. 時間と数量の扱い

ゲーム内1日は内部の `5400 s`（`5,400,000 ms`）に対応。常に単調増加するSimClockから日数を求める。

```text
elapsedDays = (t1 - t0) / 5_400_000
quantity(t1) = quantity(t0) + ratePerDay × elapsedDays
```

- 資金・在庫・容量・電力/水の計算は単位を明記。資金はCreditの1/100単位の64bit整数。数量は適切な整数最小単位＋余りを保持する。
- 率×時間で増減する量は**変更時刻で前の率を確定し、そこから新しい率で積分**する。過去にさかのぼって新しい率を当てない。
- レートが一定期間続く場合、毎フレーム加算しない。容量上限に到達する予測時刻を予約し、条件が変わったら失効する。
- 多数の地区/建物の同種レートは共有してよいが、施設の接続・容量・残高・個人の行先など、個別に必要な情報は保持する。

## 2. 住宅需要と転入の定量モデル

住宅需要は**住戸という在庫（戸数）**、転入・世帯分離は**時間当たりの発生率（世帯/ゲーム日）**として区別する。日当たりの値と空き住戸数を単純に引き算しない。

各地区dについて、初期の予測期間 `demandHorizonDays = 7`（調整可能）を使う。

```text
freeHomes_d = max(0, housingCapacity_d - occupiedHouseholds_d - reservedHomes_d)

externalMigrationRate_d = externalMigrationPotential_d // 世帯/ゲーム日
formationRate_d = householdFormationPotential_d        // 既存市民の世帯分離/ゲーム日

attraction_d = clamp(
    0.35 × satisfaction_d/100 +
    0.25 × jobAccess_d/100 +
    0.15 × serviceScore_d/100 +
    0.15 × transportScore_d/100 +
    0.10 × affordability_d/100,
    0, 1
)

wantedHomes_d = ceil(
  (externalMigrationRate_d + formationRate_d)
  × attraction_d × demandHorizonDays
)

housingShortage_d = max(0, wantedHomes_d + unhousedHouseholds_d - freeHomes_d)
housingDemandScore_d = clamp(
  100 × housingShortage_d / max(1, wantedHomes_d + unhousedHouseholds_d),
  0, 100
)
```

**実際の入居/転入は需要バーの値から直接生成しない。** その日の候補発生は固定小数点の端数を蓄積して整数候補にする。

```text
migrationExpectation_d =
  migrationCarry_d + externalMigrationRate_d × attraction_d × migrationSpeedPerDay
migrationCandidates_d = floor(migrationExpectation_d)
migrationCarry_d = migrationExpectation_d - migrationCandidates_d

formationExpectation_d =
  formationCarry_d + formationRate_d × attraction_d × formationSpeedPerDay
formationCandidates_d = floor(formationExpectation_d)
formationCarry_d = formationExpectation_d - formationCandidates_d
```

- `migrationSpeedPerDay` 初期案0.20、`formationSpeedPerDay` はゲームバランスで定める調整値。率に端数があっても毎日切り捨てて失わない。**例：毎日0.2世帯の期待値なら5日で1世帯候補を発生できる。**
- 住宅空き容量と候補の優先順位（候補発生時刻→固定seedによる安定順）で入居を決める。空きのない候補は都市内の待機世帯または域外の未入居希望として別管理し、二重の住民IDを作らない。長期に溜まる域外候補には待機期限/離脱率を設ける（係数は調整値）。
- **域外からの転入**は住戸を予約した場合だけ新しい`CitizenId`/`HouseholdId`を生成する。
- **既存市民の世帯分離**は、既存のCitizenIdを新しいHouseholdIdへ移す。人口を増やさず、元の世帯と新しい世帯の所属・住戸予約を一度に確定する。空き住戸がなければ保留する。
- `externalMigrationPotential` は域外道路接続・雇用余力・人口規模等の関数。人口0でも住宅/職が揃えば正の値になり得るようにし、最初の入居が永久に発生しない構造を避ける。
- 職がない場合も小規模な転入・失業世帯があり得るが、長期の雇用不足は転出率へ影響する。
- 出生/死亡/転出は年齢・医療・満足度・雇用を使う独立イベント列。出生だけが人口+1、既存市民の世帯分離では人口が増えない。
- 転居時は新住戸予約後に旧住戸を解放。HouseholdIdとCitizenIdの参照が同時に二重居住にならない。

受入試験：`externalMigrationRate=1世帯/日、attraction=1、migrationSpeedPerDay=0.2` で、空き住戸がある場合に5日経過した時点で少なくとも1世帯候補が生じること。既存市民の世帯分離のみを発生させた場合、世帯数は増えるが人口は変化しないこと。固定seed・同じ論理時間で再現可能なこと。

## 3. 商業・工業・オフィスの需要

| 用途 | 初期需要量のモデル |
| --- | --- |
| 商業 | `householdConsumption + tourismConsumption - effectiveRetailSupply` |
| 工業 | `localGoodsShortage + externalExportOrders - effectiveIndustrialSupply` |
| オフィス | `qualifiedWorkforce × targetOfficeJobShare + businessServicesDemand - occupiedOfficeCapacity` |

それぞれ目的の単位（商品/ゲーム日、職枠、床面積等）に合わせ、単位が異なる場合は建物テンプレートの換算率を使う。需要スコアは `0..100`。

```text
shortage = max(0, targetQuantity - effectiveCapacity)
demandScore = clamp(100 × shortage / max(targetQuantity, minimumScale), 0, 100)
```

- **初期の外部需要は正の基準値を持つ。** 外部道路に接続した新規都市で最初の工業/商業が発生できる。
- `effectiveCapacity` は単なる敷地数ではなく、稼働状態、雇用枠、電力/水、物流、施設レベルによる処理能力を使う。
- 供給能力不足/空き店舗/工場停止は各用途の需要を変える。バーは表示上なめらかでも、Processの不足数を隠さない。
- 新築需要が正でも全区画に即時建築するわけではない。候補を競争させて建設予定時刻に予約し、需要が満たされたら超過建設を抑える。

## 4. 建物の出現・成長・衰退

区画の建設適性:

```text
eligible = zoned
        && roadConnected
        && terrainAllowed
        && notAlreadyOccupied
        && requiredUtilityConnectionsExist

buildPriority = 0.30 × demandScore
              + 0.20 × landSuitability
              + 0.20 × transportAccess
              + 0.15 × serviceScore
              + 0.15 × developerReturn
              - penalties
```

- 候補敷地を需要と `buildPriority` の降順に評価し、同点は安定seed/ParcelIdで選定。既に建築予定を予約した敷地には二重建築しない。
- 建築予定→建設中→完成のState遷移。建設中の人口/雇用枠は有効な供給に含めず、完成イベントで有効化。
- 初期の建設時間案：小規模住宅・商店は1ゲーム日、中規模は3日、高密度は5日、大型公共施設は種類別データ。必要なら調整。
- 建物レベル1→2は、利用率80%以上、サービススコア60以上、地価スコア50以上が3日継続した場合に候補化。
- 2→3は、利用率85%以上、サービススコア70以上、地価スコア65以上が5日継続した場合に候補化。
- 逆に利用率50%未満または重大な供給不足が7日続いたら劣化・空室増・放棄の候補化。停電1回で即廃墟にしない。
- レベルアップ時に増える住戸/雇用枠は `BuildingCapacityChanged`。居住IDの数は自動的には増やさず、受入要求を処理する。

## 5. 通勤・雇用のマッチング

```text
jobScore(citizen,job) =
    0.40 × educationFit
  + 0.25 × expectedIncome
  + 0.25 × commuteAccessibility
  + 0.10 × personalPreference
```

- 求人ごとに職種・教育レベル・賃金帯・建物容量を登録する。マッチングは新規求職者/求人が生じた時点で、その関係する地区群だけ実行。
- すでに働いている人を全員毎日自動で再選考しない。転職の需要/施設閉鎖・税/交通の大きな変化などのイベントで必要な集団だけ再評価。
- 雇用成立はJobId/EmployerIdとCitizenIdの対応を記録し、施設が閉鎖されたら雇用関係を解除して失業State/再就職待機へ。
- 通勤の移動は住所と勤務先の座標で経路探索し、通勤コストが長いほど満足度・生産性に緩いペナルティを持つ。個別の経路IDは必須に応じて保持。

## 6. 満足度・地価・移住の反応

満足度は [POPULATION_ECONOMY](POPULATION_ECONOMY.md) の5項目重み付けによる `0..100` 指標を用いる。基準値の変化だけで毎回転入/転出率を瞬間変更させず、**日次の平滑化**を行う。

```text
smoothedScore(T) = 0.75 × smoothedScore(T−1) + 0.25 × currentScore(T)
```

**高頻度の揺れを避ける条件:** 入居/転居/退去の再評価は同一世帯について一定日数のクールダウンを設ける。例：不満が閾値未満の状態が7日続いて初めて転出候補とする。

地価は交通到達性・公共サービス・公園/水辺・汚染・騒音・税/住宅負担を合計して `0..100` のスコアへ正規化。土地価格への換算率は地域/用途データで定義する。スコアが高いほど住宅の負担も増える**相互作用**を維持する。

## 7. 供給・廃棄・物流の蓄積モデル

```text
electricityKWhPerDay = Σ occupiedResidentialUnits×10
                     + Σ commercial/industrial/office demandByTemplate

waterM3PerDay = Σ residents×0.18 + Σ facilityDemandM3
sewageM3PerDay = suppliedWaterM3PerDay×0.8
wasteKgPerDay = Σ residents×1.0 + Σ facilityWasteKg
```

- 上水の未供給分が出た場合は**実際に供給した水量**を下水の発生基礎へ使う。需要量と供給量を混同しない。
- 発電はMW×時間をkWhへ換算し、送配電の損失・供給不足・優先配分を明示。施設が未接続ならその電力需要を「供給済」としない。
- ごみは地区/建物群の発生率を条件変更時に再アンカーし、満杯到達時刻を予測して回収要求を起動。ごみの消失・二重回収を許さない。
- 倉庫の在庫は `Opening + Production + Imported − Sales − DeliveredOut − Waste = Closing`。ShipmentIdは輸送中状態を保持し、発送と納品で同じ数量を二重計上しない。

## 8. 都市財政の計算

```text
taxRevenueMinorUnits =
    Σ(eligibleTaxBaseMinorUnits × taxRate × dayFraction)
serviceExpenseMinorUnits =
    Σ(facilityUpkeepMinorUnitsPerDay × budgetFactor × dayFraction)
cityBalanceNext = cityBalancePrev + postedIncome - postedExpense
```

- 期間内の積算が必要なら**取引IDと元帳**へ記帳し、日次決算時に確定する。端数は計算の余りとして繰越し、会計書類ごとに勝手な丸めをしない。
- 建設費/融資実行は確定時点で即時記帳。TaxPeriodClosedやDailyExpenseClosedのイベントは同じ日付/台帳種別に対して冪等。
- 税率・サービス予算が変更されたら、それ以前の期間は旧率、新しい期間は新率で確定。日付を遡って課税額が変化しない。
- 資金不足の建築要求は支出前に不成立。UIは現在の資金、必要額、想定維持費を表示する。
- 財政悪化が続く場合は予算調整、ローン、税率調整、撤去/再開発といった回復策がある。サンドボックスに強制終了を設けない。

## 9. 計算モデルの安定性・正当性

- 市民・世帯・企業・施設の全モデルで、**個体が存在することと毎フレーム計算することを混同しない**。
- 各グループの「人数」は必要な数量計算に使うが、誰がどこに到着するかを追う対象はIDを失わない。
- `calcVersion`・`dataVersion`を保持し、ルールや単位が変わる場合はセーブ互換を検証する。
- ある指標が他の指標へ再帰的に影響する場合、1時刻内のイベント処理は有限回で収束する設計とし、収束しない状態を黙ってスキップしない。
- 100人、1000人、10万人、100万人で、同じseedの相対的な需要・費用・人口変化が連続的で説明可能なことをプレイテストする。

## 10. 最小のテスト例

1. 空住戸100・転入希望120世帯に対し、入居100・待機20で世帯IDが重複しない。
2. 商業供給の不足で商業需要が増え、店の開店後に不足が減少する。
3. 高密度住宅が建設中は人口と税基礎が増えず、完工後の入居時点で増える。
4. 住宅の魅力を改善すると一定の日数を経て転入傾向が変化し、同時刻に全住民が瞬間移動/転出しない。
5. 税率を日中に変更しても、日次会計で変更前/後の期間を正しく分割できる。
6. 1000世帯のごみ・水・電力を長時間積算した結果と、短い更新間隔で積算した参照版が誤差許容内で一致する。
7. 初期資金500,000 Creditで外部道路/住宅/商業/工業/基本インフラを作り、初回の入居・税収を確認できる。

**パラメータ調整で変えてよいのは数値であり、ID/定員/在庫/時刻の正しさではない。**
