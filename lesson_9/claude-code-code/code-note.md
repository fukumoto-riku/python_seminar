# `claude-code-code` の QM9 バンドギャップ予測コード解説

## 1．概要

### 1.1 目的

このディレクトリの中心となる [`qm9_gap_prediction.ipynb`](./qm9_gap_prediction.ipynb) は，`qm9_bandgap_train.csv` に含まれる 15,000 分子の SMILES と `gap_eV`（HOMO-LUMO ギャップ，単位 eV）を使って，`qm9_bandgap_test_without_answer.csv` の 4,000 分子の `gap_eV` を予測するノートブックである．

評価指標は MAE（平均絶対誤差）であり，正解値と予測値の差の絶対値の平均で表される．小さいほど誤差が小さい．

### 1.2 コード全体の流れ

ノートブックは，おおまかに次の順序で処理を行う．

1. ライブラリを読み込み，乱数シードを推奨値の 8 に固定する．
2. `qm9_bandgap_train.csv` と `qm9_bandgap_test_without_answer.csv` を読み込み，行数・欠損・重複を確認する．
3. RDKit で SMILES から 3 種類の特徴量（2D 記述子 217 個，Morgan カウントフィンガープリント 2048 次元，MACCS keys 167 次元）を生成する．
4. 学習データの統計量だけを使って前処理（中央値補完，定数列の除去）を行う．
5. `KFold(n_splits=5, shuffle=True, random_state=8)` の Out-of-Fold（OOF）予測で汎化性能を測る枠組みを作り，ベースライン LightGBM を評価する．
6. Optuna で LightGBM と XGBoost のハイパーパラメータを交差検証 MAE 最小化で探索する．
7. CatBoost を実績のある固定設定 + early stopping で学習する．
8. 3 モデルの OOF 予測から，非負・総和 1 の制約付き格子探索でブレンド重みを決める．
9. 学習データ全体で 3 モデルを再学習し，テストデータを予測して 3 つの提出 CSV を保存する．
10. 正解付きテストデータ配布後は，ノートブック 12 章のセルを再実行して test MAE を確認する．

入力から出力までの関係は次のとおりである．

```text
qm9_bandgap_train.csv (15,000 行)
    ↓ RDKit 特徴量生成（記述子 + Morgan カウント FP + MACCS）
特徴量行列 X_train（2,357 次元）
    ↓ 5-fold OOF で評価・Optuna チューニング・ブレンド重み最適化
LightGBM / XGBoost / CatBoost の 3 モデル
    ↓ qm9_bandgap_test_without_answer.csv (4,000 行) を予測
submission.csv（重み最適化ブレンド）ほか提出 CSV 3 種
```

## 2．特徴量

規則 5.1（データ追加禁止・SMILES からの特徴量生成は可）に従い，特徴量はすべて配布された SMILES から生成する．

| 特徴量 | 次元 | 内容 |
| --- | ---: | --- |
| RDKit 2D 記述子 | 217 | 分子量，LogP，TPSA，環数などの物理化学的記述子（`Descriptors.CalcMolDescriptors`） |
| Morgan カウント FP | 2048 | 半径 2 の部分構造の出現回数（ECFP4 相当のカウント版，`rdFingerprintGenerator`） |
| MACCS keys | 167 | 定義済み部分構造の有無（`MACCSkeys.GenMACCSKeys`） |

### 2.1 前処理とリーク防止

- 無限大は欠損に変換し，学習データの中央値で補完する．中央値はテストデータを含めずに計算する．
- 学習データ上で分散が 0 の列（全分子で同じ値のビットなど）を除去する．2,432 次元から 2,357 次元になる．
- SMILES の解析に失敗した行は，学習データでは除外し，テストデータでは中央値ベクトルで補完して 4,000 行の予測を厳守する（今回の配布データでは train・test とも解析失敗は 0 件）．

## 3．モデルと検証

### 3.1 検証設計

`KFold(n_splits=5, shuffle=True, random_state=8)` の OOF 予測で評価する．OOF は「各 fold の検証データに対する，その fold の学習データだけで学習したモデルの予測」を全行分つなげたものであり，学習に使っていないデータへの予測精度（汎化性能）を測れる．

### 3.2 LightGBM（Optuna でチューニング）

- Optuna（TPE サンプラー + MedianPruner）で，3-fold 交差検証の MAE を目的関数として最小化する（探索 fold 数を 5 から 3 に減らして時間を節約）．
- 損失関数（L1 / L2），学習率，葉の数，列・行サブサンプリング率，正則化係数を探索する．
- 各 trial は early stopping 付きで学習し，見込みのない trial は fold 単位で打ち切れる（pruning）．
- 最初の trial に実績のある設定を `enqueue_trial` で登録し，探索が不調でも一定の性能を確保する．
- 時間上限 1,800 秒の設定で 4 trials を実行した（実測 2,526 秒．1 trial が約 10 分と長く，上限判定が trial 間で行われるため設定値を超過した）．trial 数は少ないが，OOF MAE はベースラインより 0.0039 eV 改善した．

### 3.3 XGBoost（Optuna でチューニング）

LightGBM と同じ手順で探索する．損失関数は MAE を直接最適化する `reg:absoluteerror` と `reg:squarederror` を候補にする．時間上限 1,200 秒の設定で 4 trials（実測 1,497 秒）を実行し，`reg:squarederror`・`max_depth = 9` の設定が選ばれた．

### 3.4 CatBoost

CatBoost は 1 回の学習が重いため Optuna は使わず，実績のある固定設定（深さ 8，学習率 0.05，損失 RMSE，評価 MAE）+ early stopping で学習する．学習アルゴリズムが他の 2 モデルと異なるため，ブレンドの多様性に寄与する．5-fold の平均 best_iteration は 5,998 であり，上限の 6,000 にほぼ到達した．

### 3.5 ブレンド（OOF 重み最適化）

3 モデルの OOF 予測の重み付き平均について，重み（非負・総和 1）を 0.02 刻みの格子探索で MAE 最小になるように決める．重みの決定に使うのは OOF 予測だけであり，テストデータの情報は一切使っていない．

### 3.6 テストデータの予測

- CV で決めたパラメータを使い，学習データ全体で各モデルを再学習する．early stopping が使えないため，木の本数は「CV の平均 best_iteration × 5/4 + 50」とする（fold あたりの学習データが全体の 4/5 だったことの補正）．
- 予測値は学習データの `gap_eV` の範囲にクリップし，極端な外挿を抑える．

## 4．結果

OOF MAE は次のとおりである（値はノートブック 13 章の出力）．

| モデル | OOF MAE [eV] |
| --- | ---: |
| ベースライン LightGBM | 0.2054 |
| LightGBM（チューニング後） | 0.2015 |
| XGBoost（チューニング後） | 0.1967 |
| CatBoost | 0.1963 |
| 単純平均ブレンド | 0.1915 |
| 重み最適化ブレンド（本命） | 0.1912 |

- 採用したブレンド重みは LightGBM 0.22，XGBoost 0.34，CatBoost 0.44 である．
- 重み最適化ブレンドが最良で，ベースラインに対して OOF MAE を 0.0142 eV（6.9 %）改善した．
- LightGBM の gain 重要度の上位は `FractionCSP3`（sp3 炭素の割合）や `HallKierAlpha` など共役の程度と関係する記述子であり，HOMO-LUMO ギャップの化学的傾向と整合する．
- フル実行の所要時間は 152.8 分であった（Apple Silicon，CPU のみ）．

## 5．提出ファイル

規則 5.3 の「提出可能なモデルは最大 3 個」に対応して，次の 3 つを保存する．いずれも列順は `smiles,gap_eV`，4,000 行である．保存時に行数・列順・欠損・`smiles` の対応を assert で検証している．

| ファイル | 内容 |
| --- | --- |
| `submission.csv` | 本命：OOF で重み最適化した 3 モデルブレンド |
| `submission_2_lightgbm_single.csv` | チューニング済み LightGBM 単体 |
| `submission_3_mean_blend.csv` | 3 モデルの単純平均 |

複数提出時のファイル名は規則で未指定のため，本命のみ規定名 `submission.csv` とした．ブレンドを何モデルと数えるかは規則 8 章の未確定事項であり，主催者に確認する．

## 6．レギュレーションとの対応

| 規則 | 対応 |
| --- | --- |
| データの追加禁止（5.1） | 配布 CSV のみ使用．特徴量はすべて SMILES から生成 |
| モデルは 0 から学習（5.2） | 事前学習済みモデルを使わず，ノートブック内で学習 |
| LLM・NNP の禁止（5.2） | 勾配ブースティング決定木のみ使用 |
| 提出モデル最大 3 個（5.3） | 基礎モデルは LightGBM・XGBoost・CatBoost の 3 個 |
| 乱数シード（5.3） | 推奨値 8 に固定し，ノートブックに記録 |

## 7．再現方法

依存関係は `pyproject.toml` で管理している．次の手順で再現できる．

```bash
cd claude-code-code
uv sync                         # 依存関係の導入（初回のみ）
uv run jupyter lab              # ノートブックを開いて上から実行する
```

コマンドラインで一括実行する場合は次のとおりである．

```bash
uv run jupyter nbconvert --to notebook --execute --inplace qm9_gap_prediction.ipynb
```

- フル実行の実測は 152.8 分である（Apple Silicon，CPU のみ）．
- 動作確認だけを行う場合は，環境変数 `QM9_QUICK=1` を設定して実行すると 10 分弱で完了する（データを間引くため精度は参考値）．

## 8．当日の評価手順

1. 配布された `qm9_bandgap_test_with_answer.csv` を `claude-code-code/` または `lesson_9/` 直下に置く．
2. ノートブック 12 章「当日評価」のセルを再実行する．
3. 3 つの提出値それぞれの test MAE が表示される．
