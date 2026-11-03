# JVM技術構成・モジュール・描画方針 v1.0

> **完成版の初期技術選定（設計判断）**。採用前に最小プロトタイプでビルド・3OS表示・性能を検証し、重大な不適合があれば選定を改訂する。データ構造・エンジンの動作は [IMPLEMENTATION_SPEC](IMPLEMENTATION_SPEC.md) / [RUNTIME_PROTOCOL](RUNTIME_PROTOCOL.md)、OOPとUPDは [CODING_RULES](CODING_RULES.md) を優先する。

## 1. JVM言語と描画基盤

| 項目 | 設計上の基準 |
| --- | --- |
| JDK | Java 21 LTSをターゲットにする。別JDKへの更新は互換性・性能測定を伴う |
| Kotlin | ゲームの組立・UI・コマンド・設定・テストなど高レベルの責務を担当。採用バージョンはビルド時に固定 |
| Java | 大量ID・イベントキュー・経路探索・メモリ/CPUに敏感なエンジン処理を担当。性能は計測で判断し、すべてを無条件にJavaへ移すわけではない |
| ビルド | Gradle Kotlin DSL。Wrapper、toolchain、依存バージョン、検証用タスクを固定して再現可能にする |
| 描画/UI | **libGDX + Desktop LWJGL3 backend**を第一実装案とする。3D描画は独自の都市チャンク/LOD/インスタンス管理を載せ、UIは既存のlibGDX UI部品を活用する |
| グラフィックスの最低共通経路 | DesktopのOpenGL 3.3相当を目安。macOSを含む基準経路で動くことを優先し、OpenGL 4.3以上必須の実装にしない |
| サウンド | libGDXの音声基盤を候補に、距離・地区単位でミキシングして負荷を制御 |
| 永続化 | JVM内部オブジェクトを丸ごとシリアライズせず、Data層のバージョン付きチャンク形式 |
| 初期マルチスレッド | Processの状態確定は安定順序で単一スレッド。表示・I/O・独立計算は適切なスレッドに分ける |
| ソース互換 | Kotlin/Java間で使う値/インターフェースは明示した契約に限定。各言語固有の高次構文を境界で強制しない |

**ライブラリ候補の根拠:** libGDXはWindows/Linux/macOS向けのJavaゲームフレームワークとデスクトップ描画APIを提供する（https://libgdx.com/ 、https://libgdx.com/features/）。デスクトップでのOpenGL経路とmacOSの最大4.1制約は公式説明を参照（https://libgdx.com/wiki/graphics/opengl-es-support）。GradleのJava21対応は公式互換表とリリースノートを参照（https://docs.gradle.org/current/userguide/compatibility.html）。

**留保:** 上記は「3D都市で100万人を描画しても性能保証できる」という証明ではない。最初のプロトタイプで少なくともWindows/Linux/macOSのGLコンテキスト、instancing、カメラ、描画チャンクを検証し、ライブラリ版を固定する。

## 2. プロジェクトモジュール案

\`\`\`text
:shared-contracts       Immutable DTO / ID値 / コマンドと結果の契約だけ
:process:core           ID・正規State・集団・SimClock・イベント・容量予約
:process:transport      道路/徒歩/車両/公共交通のルール
:process:city           建物/住民/経済/サービス/環境のルール
:data                   マップ・アセット定義・セーブの読込/保存
:ui                     CityViewの表示・カメラ・UI操作・音
:desktop                起動と依存組立だけ（都市ルールを持たない）
:benchmark              ヘッドレスの正しさ・性能比較
\`\`\`

実装の簡易化のため、最初は\`:process:core\`と\`:process:transport\`などを同一ビルドモジュールにまとめてよい。ただし**UPDの責務境界はファイル/パッケージの置き方に関わらず守る**。

### 依存方向

- UI → Process契約・共有値。UI → Dataは絶対に禁止。
- Process → Dataへは保存/取得の専用契約で依頼する。DataがProcessのゲームルールを判断しない。
- Data → Shared contractsは許可。DataからUIを操作しない。
- Desktopのcomposition rootは起動・依存組立だけを行い、UPDの新たな処理層にはしない。
- SharedはDTOや値などの共通要素。ビジネスロジックをSharedに移して層境界を迂回しない。

## 3. 重要なスレッド境界

\`\`\`text
Process確定スレッド
    ↓ 読み取り専用のCityView/WorldSnapshot
UI/描画スレッド (libGDX render)
    ↓ CityCommand
Process確定スレッド

Process要求 → Data I/O worker → 結果 → Process → UI
\`\`\`

- Processの**唯一の正規WorldState**は描画/IOスレッドが直接書き換えられない。
- 画面描画は二重バッファまたは不変スナップショットから読み取る。カメラが動いても論理Stateを進めたり時刻を変えない。
- 時間的LODで予約する次到着イベントはSimClockと安定順序に従う。イベントの処理順がJVMワーカーの完了順で変わらない。
- 並列処理は独立の経路候補計算や地形メッシュ生成などに限定し、Process確定時に安定ソート/順序で適用する。無条件に地区単位で同時書き込みしない。

## 4. 描画実装の設計

都市の見た目を担当する独自のRendererは、Processから次を受け取る：地形/道路/建物チャンクのrevision、CityViewの可視ID一覧、座標補間に必要な時刻・区間・State、各施設の外観テンプレートID。

- **地形:** 32×32セルのチャンクメッシュとズームLOD。地形改変時だけ該当チャンクを更新する。
- **建物:** 建物テンプレートをインスタンス化して描画し、遠方は簡易メッシュ/テクスチャへ切替。個々の建物をCPU側で毎フレーム別々に組み立てない。
- **車/歩行者:** 近距離では必要な詳細表現、遠方では密度を落とした代表描画。**論理的な全ID/交通量は維持**。
- **レイヤー:** 交通/水道/汚染/地価の数値マップはProcessが生成した地区/セル集約結果をテクスチャ等で表示。UIでゲームルールを再計算しない。
- **GPU API:** 基準経路で利用できるinstancing・頂点バッファ・テクスチャアトラス等を検討。Compute Shader必須にはしない。
- **性能:** CPU側のdraw call、GPUメモリ、シミュレーションのCPU時間、メッシュ更新数、可視オブジェクト数を別計測する。
- **可読性:** 斜め俯瞰と道路・設備の判読性を優先し、超高精細の物理シェーダーを最初の目標にしない。

## 5. 厳格なOOP規約との両立

- 具象クラスの継承は禁止。第三者ライブラリでよくある「Adapterを継承して起動」を避け、**公開インターフェースの実装や移譲**でアダプタを作る。ライブラリが具象継承を強制する場合は設計の採否を再検討する。
- 具象クラスはfinal。Kotlinも原則明示的にfinalを付ける。
- 自作APIでオーバーロードを定義しない。外部ライブラリが提供するオーバーロード呼び出しは既存規約の範囲で認める。
- 1クラス1責務、1関数1処理、制御ネスト最大2段、カプセル化。
- データ指向の配列やメモリレイアウトを利用しても、管理クラスの内部可変配列はprivateで保持する。外部は型安全な契約で操作する。
- レンダリングのリソースハンドルをProcessへ渡す、Processに画面描画させるなどUPD境界を破らない。

## 6. 検証して選定を確定する手順

1. Java21/Kotlin/Gradleの最小multi-moduleが3OSでビルド可能。
2. libGDX/LWJGL3でウィンドウ・カメラ・インスタンシング・チャンクメッシュを描画できる。
3. RendererをOFFにしてProcessコアだけ起動し、同じseed/入力で等しい論理結果が出る。
4. 1万/10万/100万IDと、代表的な建物・道路・車/人の可視密度でCPU/GPU/GCを計測する。
5. ボトルネックが描画基盤にあり契約通りのスケールを達成できない場合、代替描画実装を比較して根拠を残す。Process/UPDにゲームルールの修正を強制しない。
6. 採用した具体的な依存ライブラリとGradle Wrapperのバージョンをlockfile等へ固定し、CIで検証する。

この技術選定は[#67](https://github.com/tomiya7688/city-builder/issues/67)と[#68](https://github.com/tomiya7688/city-builder/issues/68)を通じて実装/検証する。未検証の技術条件を「全OSサポート済み」と記さない。
