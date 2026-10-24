# QTC / EQTC 圧力投影研究 — 全体案内と再現手順

2026-10-24、最新のκ地図研究（K1〜K17）と4種類の速度測定・判定を含む保存版です。

この研究で扱っているのは、**共通の細格子上で、圧力を鉛直の区分線形基底へGalerkin縮約する、静的な単発圧力投影の代理実験**です。原QTC/EQTCの水平四分木、minimal/virtual-face離散化、移流、水面追跡、飛沫、時間発展を一式実装したものではありません。新規性の判定も行っていません。

目的は「計算現象の理解7割、軽量化3割」。自由度を減らしたことと、必要な精度を保って総計算時間を短くしたことを区別します。失敗、精度未達、予測の不合格、探索的な発見も保存しています。

## まず読むもの

1. **最新の成果・限界**：[最新検証のまとめ](docs/QTC-最新検証.md)
2. **導出と実験の本文**：[κ地図研究レポート](outputs/eqtc-e7-kappa-map/κ地図研究レポート.md)（§00が最新のまとめ、§11が速度測定）
3. **研究を再開する場合**：[最新の引き継ぎ](outputs/eqtc-e7-kappa-map/引き継ぎ.md)
4. **保存された場を見比べる**：[E7比較ビューアー](outputs/eqtc-depth-placement-lab/index.html)、[E6残差診断](outputs/eqtc-residual-analysis/index.html)、[E4比較](outputs/eqtc-stepped-bed-lab/index.html)

HTMLは保存結果の表示です。開くだけで求解や計時を再実行することはありません。

## 現時点の到達点

- 1列・追加節点1個の更新式を厳密に導き、直接求解で検算しました。圧力の誤差エネルギーが減っても、残差の二乗・発散が増える条件を分離できました。
- 3Dで独立列近似がよく合っていた理由には、水平の遮蔽と段差近傍のスピルが打ち消し合う効果があります。両方を含めるW予測を未使用条件で検証しました。スピル係数は現在も経験値です。
- 入力右辺だけを使う配置（IC）に加え、基準残差から配置を選ぶRSEを検証しました。一般の台地・二方向の棚でも改善しますが、探索した最良側の配置と常に一致するわけではありません。
- y方向に一様な底などの条件では縮約系を厳密に分解して速く解けます。一般の底への前処理も試しました。**E4での精度付き総費用は高速化未達のまま**で、後の限定された縮約系・E7パイプラインの改善とは、比較対象と計時範囲が異なります。

### 最新4測定

比は「候補時間 ÷ 比較対象時間」で、小さいほど速い値です。下表の中央値だけで合否を決めず、事前固定した各入力の規則で判定しています。

| 測定 | 比較と主な観測 | 事前判定 |
|---|---|---|
| K7：縮約系 | y余弦解法／従来縮約CG。新規準備ありの比の中央値0.421、再利用0.035 | 新規準備のT1は最大0.506で不合格。再利用T2・品質T3は合格 |
| K10b：全工程 | 高速な中点配置パイプライン／E7基準。補正8回の比の中央値0.480 | 品質Q1〜Q3合格、速度T1・選択費用T2は不合格。元K10の格子不一致も保存 |
| K13：一般の底 | 平底の代用行列による前処理／縮約CG。新規準備の比の中央値0.649、再利用0.374 | 速度T1・T4は不合格。反復数T2・品質T3は合格 |
| K16：二段RSE | 基準求解→RSE選択→最終求解／E7中点基準。補正8回の時間比の中央値0.711、誤差E比0.581 | R1〜R3はすべて合格。幾何表の構築約3.5秒は計時外 |

これらは同じ機械・今回の入力集合・単一スレッドBLASでの観測です。K7の再利用比を全工程の速度比とは扱いません。細格子の最速法や原EQTC全体への優位性、長時間の体積保存は主張しません。詳しい比較対象、範囲、最大値と例外は[最新検証](docs/QTC-最新検証.md)を参照してください。

## 実験の流れ

| 段階 | 問いと結果 | 一次資料 |
|---|---|---|
| E0：入口 | Galerkin代理の数学・境界・独立検算。静水21/21合格、擾乱の縮約72設定は精度未達 | [結果](outputs/eqtc-fluid-lab/結果.txt)、[README](outputs/eqtc-fluid-lab/README.txt) |
| E1：層厚校正 | 参照を使うオフライン校正。表層では縮約が残る一方、深部は全基底へ回復する例がある | [結果](outputs/eqtc-thickness-lab/結果.txt) |
| E2：参照不要の自動選択 | 未使用24件の精度は合格。縮約14・fine復帰10だが、選択を含む全費用は短縮せず | [結果](outputs/eqtc-auto-layer-lab/結果.txt) |
| E3：最初の候補を予測 | 主方式は細格子CG比0.938で宣言合格。ただし矩形のfine-DCT直接法がさらに速い | [結果](outputs/eqtc-predict-layer-lab/結果.txt) |
| E4：段差・台地・二方向棚 | 42入力×5方式の最終精度合格。主24入力の総時間は最速の試験済み細格子法比1.121で高速化未達 | [結果](outputs/eqtc-stepped-bed-lab/結果.txt)、[監査](outputs/eqtc-stepped-bed-lab/verification/checks.json) |
| E5：縮約行列の直接構築 | 同じ基底でRAPと直接構築の同等性を検証。今回の直接COO実装はRAPより遅い | [結果](outputs/eqtc-direct-break-lab/結果.txt) |
| E6：細格子Jacobi補正 | 52保存解・260計時。0回で精度未達の例が、最大8回で全品質合格へ転じた例は0 | [結果](outputs/eqtc-postsmooth-lab/結果.txt) |
| E6残差診断 | 保存場の空間・スペクトル後処理。深部の表せない成分を可視化し、配置の問いへ進んだ | [診断](outputs/eqtc-residual-analysis/診断と次の実験.txt) |
| E7：深部の節点配置 | 同じ自由度の19層より深部中点追加が全8比較で低い誤差E。ただし総時間短縮は未達、深部精度も基準未達 | [最終報告](outputs/eqtc-depth-placement-final-report.txt)、[比較](outputs/eqtc-depth-placement-lab/index.html) |
| E7機構解析 | 誤差E、残差、発散、Jacobi、物理量の定義と恒等式を調べた | [レポート](outputs/eqtc-e7-mechanism-analysis/E7機構解析レポート.md) |
| 節点オフセット掃引 | 置き場所によってEと発散が異なる変化をする条件を探索した | [レポート](outputs/eqtc-e7-knot-offset-sweep/節点オフセット掃引レポート.md) |
| 位相機構追試 | 事前予測、位相・列種別・相互作用、独立検算を進めた。初期解釈には訂正がある | [本文](outputs/eqtc-e7-phase-mechanism/E7追試レポート.md)、[訂正表](outputs/eqtc-e7-phase-mechanism/00_訂正表.md) |
| κ地図研究：最新 | 1D厳密式→未知条件の予測→3D機構→RSE配置と解法。K1〜K17で一区切り | [本文](outputs/eqtc-e7-kappa-map/κ地図研究レポート.md)、[引き継ぎ](outputs/eqtc-e7-kappa-map/引き継ぎ.md) |

古いレポートの「次の実験は未実施」は、その保存時点の記述です。原資料を改変せず、進行と訂正をこの案内で接続しています。参考ログの役割は[研究経緯](docs/QTC-研究経緯.md)に記載しています。

## 再現用ZIPの内容

配布ファイル名：`qtc-research-2026-10-24.zip`。開発ワークスペースでは `outputs/qtc-research-delivery-2026-10-24/` にZIP、SHA-256、検証記録を置きます。

```text
qtc-research-2026-10-24/
  README.md
  docs/                         最新結果と研究経緯
  requirements-qtc.txt          数値依存の固定版
  .eqtc-e4-deps/                Windows x64 / CPython 3.12用の数値依存
  scripts/qtc_repro/            共通の検証・実行・再梱包コマンド
  bundle-manifest.json          全収録ファイルのサイズとSHA-256
  outputs/
    eqtc-fluid-lab/             E0
    ...                        E1〜E7、診断、機構解析、掃引
    eqtc-e7-phase-mechanism/    位相機構追試
    eqtc-e7-kappa-map/          最新研究、失敗履歴、再開情報
```

入力、参照解、生のNPZ、宣言、ソース、探索・不合格・中断記録、レポート、既存の検証記録を収録します。旧ZIPとその展開・stagingの重複、別プロジェクト、実行キャッシュ、会話ログ全体は収録しません。E0〜E4などの由来照合が兄弟フォルダーを参照するため、`outputs/`の構造を保ってください。

### 1. ZIPを確認して展開する

Windows PowerShellで、ZIPと`.sha256`を同じ場所に保存して実行します。

```powershell
$zipPath = '.\qtc-research-2026-10-24.zip'
$expected = ((Get-Content -LiteralPath ($zipPath + '.sha256')).Split(' ')[0]).ToLowerInvariant()
$actual = (Get-FileHash -LiteralPath $zipPath -Algorithm SHA256).Hash.ToLowerInvariant()
if ($actual -ne $expected) { throw 'ZIP SHA-256 mismatch' }
Expand-Archive -LiteralPath $zipPath -DestinationPath '.\qtc-extracted'
Set-Location '.\qtc-extracted\qtc-research-2026-10-24'
```

新しい展開先を使ってください。ZIPにはPythonインタープリターは含めていません。

### 2. 保存結果を検算する（推奨の入口）

確認した環境は **Windows x64 / CPython 3.12.14**、NumPy 2.3.5、SciPy 1.16.3、PyAMG 5.3.0です。数値パッケージを同梱しているため、この環境では追加インストールやネット接続は不要です。

```powershell
py -3.12 -I -B -S scripts/qtc_repro/verify_bundle.py --out ..\qtc-check
```

`py`ランチャーがなければ、`py -3.12`をPython 3.12実行ファイルへのパスに置き換えます。`-I -B -S`でユーザー環境・site-packages・キャッシュ生成から切り離し、展開した依存を明示的に読みます。各検証JSONに実際のパッケージの読込元を記録します。

このコマンドは、全ファイルのSHA照合、E0〜E7の保存配列の監査、残差診断manifest照合、位相追試の選択した段の再計算、最新κ研究の宣言・主要数値・図の再生成を順番に実行します。結果は別フォルダーの `checks.json` と各段のJSON・ログに保存します。元の展開物が変わっていないことも再照合します。

共通入口は元の研究ソースを変更せず呼び出します。位相追試では、移動先で変わる依存フォルダーなどの環境情報`env`を数値JSONの比較から分離し、旧・新の環境を別項目に記録します。NumPy/SciPy版とスレッド設定の一致、すべての対象数値とraw配列の一致は別に検査します。子プロセスにも同梱依存の起動補助を適用します。

これは**全実験・全計時の最初からの再実行ではありません**。機構解析・オフセット掃引の全条件の再計算、当時のブラウザーQAの再実行も含みません。各検査の範囲は検証JSONに残します。全体検証は数分以上かかる場合があります。

```powershell
# 数値計算をせず収録ファイルだけ照合
py -3.12 -I -B -S scripts/qtc_repro/verify_bundle.py --checks hashes --out ..\qtc-hash-check

# 最新研究だけを再検算（全ファイル照合も実行）
py -3.12 -I -B -S scripts/qtc_repro/verify_bundle.py --checks E7-kappa --out ..\qtc-kappa-check
```

既存の `checks.json` を読むこと、SHAが一致すること、数値を再計算して一致すること、仮説が合格することは別の確認です。速度閾値に不合格があっても、保存された結果の監査は合格し得ます。

### 3. 実験を最初から再計算する

**参照用の展開物とは別の作業コピーを作り**、各段のREADMEと最新の引き継ぎに従って実行します。全条件の再計時は今回の納品時にはやり直していません。K12だけでも元の実行では約64分かかっており、全系列の再実行時間は未見積もりです。

旧スクリプトを隔離モードで実行するための補助入口：

```powershell
# 別コピーのルートで。例：小モデルの既存selfcheckを実行
py -3.12 -I -B -S scripts/qtc_repro/run_script.py outputs/eqtc-depth-placement-lab/run.py --selfcheck
```

`run_script.py`はスクリプトのフォルダーと同梱依存をimport先に加えるだけで、判定規則を変更しません。元スクリプトに書込処理がある場合は、その処理を実行します。

- E0〜E5：各段のREADMEに従い、新しい作業コピーで計測してください。旧CLIの監査は`checks.json`を上書きするため、保存版では上記の共通検証入口を使います。
- E6/E7：各段の`reproduce.py`・`run.py --out <新しい名前>`とREADMEを参照してください。既存の観測フォルダーに上書きしない構成です。
- κ研究：各`k*/run.py`が宣言・依存ソースのハッシュを確認し、`results/`が既存なら拒否します。作業コピー内だけで旧`results/`を別名へ退避し、対象の`run.py`と`judge.py`を実行します。失敗したK1/K4/K6/K10は、修正版K1b/K4b/K6b/K10bと区別してください。
- 再計時：他の重い計算を止めた状態で、固定した入力・規則・単一スレッド設定を維持して実行します。絶対秒数の一致は保証しません。新しい測定として保存します。

同梱バイナリーはWindows/CPython 3.12用です。他OS・Python版では、別コピーで対応する数値依存を用意する必要があり、この配布での動作検証対象外です。PDFや旧SVGレポートを再生成する場合は追加でReportLab 4.4.9が必要です。保存HTML/SVGの閲覧と上記の数値検証には不要です。

ブラウザーQA用の一部スクリプトには作者環境のChrome/Playwrightへの絶対パスがあります。別機での自動QAには設定変更が必要です。元ソースのハッシュを保つため、今回の配布では書き換えていません。

## 指標の読み方と、残っている問い

`E = (p−p_ref)ᵀA(p−p_ref)`は、条件をそろえた速度のM重み誤差に対応します。通常の残差二乗や、相対発散とは異なる評価です。最新研究の`W = ‖r_new‖²−‖r_18‖²`は正なら残差二乗が増えた意味です。`κ`は比なので分母が小さい所で巨大化し、巨大なκだけでは実害の大きさを判断できません。

各段の定義、符号、単位、境界条件、参照残差を参照してください。静的投影の誤差から長時間の体積減少・散逸・不安定性を断定しません。

未解決事項には、段差スピル係数の理論値、入力族で相互作用の換算係数が違う理由、RSEの種別間結合の近似誤差、前処理の最悪例があります。[引き継ぎ](outputs/eqtc-e7-kappa-map/引き継ぎ.md)に次回の候補と再開コマンドを保存しています。**今回は新しい研究へ進まず、測定・判定・保存を終えて待機しています。**

## 配布を作り直す場合

`scripts/qtc_repro/catalog.py`が対象フォルダーを明示します。`build_bundle.py`は既存成果物を変更せず、新しいZIPへ固定順序・固定日時で収録し、全ファイルのmanifestとZIPのSHA-256を作ります。既存のZIPは上書きしません。

```powershell
py -3.12 -I -B -S scripts/qtc_repro/build_bundle.py --root . --numpy-site .\.eqtc-e4-deps --out ..\qtc-rebuilt.zip
```

配布ZIPからの再梱包ではNumPyも`.eqtc-e4-deps`にあるため上記指定を使います。開発環境では`--numpy-site`にNumPy 2.3.5とその`.libs`・`.dist-info`のあるフォルダーを指定します。OS・圧縮実装が違う場合のZIPバイト一致は検証していません。データの同一性はmanifestとSHAで確認してください。
