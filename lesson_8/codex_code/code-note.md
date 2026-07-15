# `codex_code` のTitanic生存予測コード解説

## 1．概要

### 1.1 目的

このディレクトリの中心となる [`titanic_f1_high_performance.ipynb`](./titanic_f1_high_performance.ipynb) は，`train_titanic.csv` に含まれる712人の乗客情報と生存ラベルを使って，`test_titanic.csv` の179人が生存したかを予測するノートブックである．

予測対象の `Survived` は，死亡を `0`，生存を `1` とする二値分類である．正式な評価指標は，生存者を正例としたF1スコアである．F1スコアは，生存と予測した結果の正確さを示す適合率と，実際の生存者をどれだけ見つけたかを示す再現率のバランスを評価する．

この実装の大きな特徴は，モデルや閾値を選ぶ処理を含めた評価にNested Cross-Validation（Nested CV）を使い，単一モデル，重み付きblend，cross-fit stackingの3方式を比較していることである．

### 1.2 コード全体の流れ

ノートブックは，おおまかに次の順序で処理を行う．

1. 実験設定，乱数シード，通常実行と短縮実行の条件を決める．
2. `train_titanic.csv` と `test_titanic.csv` を読み込み，行数，列順，ラベル，IDを検査する．
3. 性別だけで予測する簡単なベースラインを確認する．
4. 氏名，家族，チケット，客室などから特徴量を作る．
5. 学習foldだけを使って欠損補完，グループ集計，target encodingを学習する．
6. CatBoost，LightGBM，XGBoost，ExtraTrees，Random ForestをOptunaで調整し，ロジスティック回帰も比較する．
7. Nested CVで，最良単一モデル，重み付きblend，cross-fit stackingを公平に比較する．
8. 学習データ全体を使う最終探索で，各モデルの設定，閾値，blend重み，stacking設定を決める．
9. 採用モデルを複数の乱数シードで学習し直し，テストデータの生存確率を計算する．
10. 3方式の生存確率をそれぞれの閾値で0または1へ変換する．
11. 179人分の `PassengerId` と `Survived` を持つ3つの提出CSVを保存し，読み戻して検査する．
12. 正解配布後は [`answer.ipynb`](./answer.ipynb) でF1スコアを確認する．

入力から出力までの関係は次のようになる．

```text
train_titanic.csv
    ↓ fold内特徴量作成・Optuna・Nested CV
6種類のベースモデルと3方式の設定
    ↓ 全train再学習
test_titanic.csvに対する6列の生存確率
    ↓ 単一／重み付き平均／2段階学習
3種類の提出CSV
```

## 2．3つの予測方法

各ベースモデルは，乗客ごとの生存確率を出力する．3方式は，この確率をどのように使って最終的な生存判定を作るかが異なる．確率が方式ごとの閾値以上なら `Survived=1`，未満なら `Survived=0` とする．

| 提出ファイル | 予測方法 | 簡潔な説明 | 閾値 | 現在の生存予測数 |
| --- | --- | --- | ---: | ---: |
| [`submission_best_single_nested.csv`](./submission_best_single_nested.csv) | 最良単一モデル | 最終OOF評価で安定性が最も高かったLightGBMを使う | `0.393` | 71人 |
| [`submission_weighted_blend_nested.csv`](./submission_weighted_blend_nested.csv) | 重み付きblend | 6モデルの確率を，学習データで決めた重みで平均する | `0.465` | 66人 |
| [`submission_crossfit_stacking.csv`](./submission_crossfit_stacking.csv) | cross-fit stacking | 6モデルのOOF確率から，2段目のロジスティック回帰が組合せ方を学ぶ | `0.432` | 67人 |

### 2.1 最良単一モデル

最良単一モデルは，複数モデルの確率を混ぜず，選ばれた1種類のモデルだけで予測する方法である．

このコードでは，最終repeated OOF評価の `robust_score` が最大のモデルを選ぶ．`robust_score` は平均F1からfold間のばらつきに対する罰則を引いた値である．

```text
robust_score = mean_f1 - 0.25 × std_f1
```

現在はLightGBMが選ばれている．乱数による揺れを抑えるため，シード42，123，2026，3407，7777の5モデルを学習し，テストデータの生存確率を平均する．

### 2.2 重み付きblend

重み付きblendは，6モデルの生存確率を，モデルごとに異なる重みで平均するソフト投票である．単純平均と異なり，OOF予測で成績が良く，ほかのモデルを補いやすいモデルへ大きな重みを与えられる．

現在の重みは次のとおりである．

| モデル | 重み |
| --- | ---: |
| Logistic Regression | `0.404046` |
| CatBoost | `0.195089` |
| Random Forest | `0.151552` |
| XGBoost | `0.123648` |
| ExtraTrees | `0.115779` |
| LightGBM | `0.009886` |

重みはすべて0以上で，合計が1になるように正規化する．現在の最良単一モデルはLightGBMだが，blendではLightGBMの重みが小さい．これは，単体性能だけでなく，ほかのモデルと組み合わせたときのF1が高くなるように重みを選んでいるためである．

### 2.3 cross-fit stacking

stackingは，6モデルの予測確率を新しい特徴量として2段目のモデルへ入力する方法である．このコードでは，2段目にロジスティック回帰を使う．

単純に学習データへ当てた確率を使うと過学習しやすいため，各乗客を学習に使っていないベースモデルのOOF確率を使う．さらに，2段目のロジスティック回帰もfoldごとに学習対象と検証対象を分けるcross-fit方式で評価する．

現在の2段目ロジスティック回帰は `C=30.0` である．テスト用の6確率も，ベースモデルの交差検証予測を平均して作る．

## 3．ディレクトリ内の主なファイル

| ファイル | 役割 |
| --- | --- |
| [`titanic_f1_high_performance.ipynb`](./titanic_f1_high_performance.ipynb) | 特徴量作成，Nested CV，最終探索，テスト予測，提出CSV作成を行う本体 |
| [`answer.ipynb`](./answer.ipynb) | 正解配布後に3提出のF1スコアと混同行列を確認する |
| [`train_titanic.csv`](./train_titanic.csv) | 712人分の乗客情報と `Survived` を持つ学習データ |
| [`test_titanic.csv`](./test_titanic.csv) | 179人分の予測対象データ |
| `test_titanic_with-answer.csv` | 正解配布後の答え合わせ専用データ |
| `submission_*.csv` | 3方式が出力した `PassengerId,Survived` 形式の提出データ |
| [`pyproject.toml`](./pyproject.toml) | Pythonのバージョンと依存ライブラリを管理する |
| `uv.lock` | 依存ライブラリの具体的なバージョンを固定する |
| [`main.py`](./main.py) | `uv init` で作られた雛形であり，予測処理には使われない |
| [`README.md`](./README.md) | 現在は空であり，予測処理には使われない |

予測ノートブックが読むデータは `train_titanic.csv` と `test_titanic.csv` だけである．`test_titanic_with-answer.csv` は，モデル，特徴量，閾値，アンサンブルを決めた後の答え合わせにだけ使用する．

## 4．処理の詳細

### 4.1 実験設定

`ExperimentConfig` は，交差検証の分割数，Optunaの試行数，乱数シードなどをまとめて管理する．通常実行の主な設定は次のとおりである．

| 設定 | 通常実行の値 | 意味 |
| --- | --- | --- |
| `outer_splits` | `5` | Nested CVの外側fold数 |
| `outer_repeats` | `3` | 外側CVの繰り返し回数 |
| `inner_splits` | `4` | ハイパーパラメータ選択用の内側fold数 |
| `final_cv_seeds` | `(42, 123, 2026)` | 最終設定を再評価する分割シード |
| `final_fit_seeds` | `(42, 123, 2026, 3407, 7777)` | 全train再学習に使うモデルシード |
| `blend_trials` | `500` | blend重みを探索する試行数 |

環境変数 `TITANIC_FAST_MODE=1` を指定すると，fold数，Optunaの試行数，シード数を減らした短縮設定になる．短縮設定は構造確認には使えるが，保存済みの通常実行結果とは条件が異なる．

### 4.2 データの受入検査

`DATA_DIR = Path.cwd()` としているため，`codex_code` を作業ディレクトリにして実行する必要がある．読み込み直後に次をassertで確認する．

- 学習データが712行12列，テストデータが179行11列である．
- 列名と列順が想定どおりである．
- `Survived` が0と1だけである．
- 学習データとテストデータの `PassengerId` に重複や交差がない．
- 学習ラベルの件数が死亡439人，生存273人である．

`PassengerId` は提出結果の対応付けにだけ使い，モデル入力から除外する．性別が女性なら生存とする単純な基準のF1は，学習データ全体で `0.7137` である．

### 4.3 特徴量エンジニアリング

`TitanicFeatureEngineer` が元の乗客情報から特徴量を作る．主な特徴量は次のとおりである．

#### 基本属性

- `Pclass`，`Sex`，`Age`
- `SibSp`，`Parch`
- `Fare`，`Embarked`

#### 氏名と家族

- `Title`：氏名から抽出した敬称
- `NameLength`：氏名の長さ
- `FamilySize`：`SibSp + Parch + 1`
- `IsAlone`：単独乗船か
- `FamilySizeBand`：家族人数の区分
- `IsChild`，`IsMother`
- `FamilyPeerCount`：学習側に同じ家族キーを持つほかの乗客が何人いるか

#### チケット，運賃，客室

- `TicketPrefix`，`TicketDigits`，`TicketPeerCount`
- `FareLog`，`FarePerPerson`
- `CabinKnown`，`CabinCount`，`Deck`

#### 組合せ特徴

- `SexPclass`：性別と客室等級の組合せ
- `TitlePclass`：敬称と客室等級の組合せ

生の `PassengerId`，`Name`，`Ticket`，`Cabin`，姓，家族キーは，最終的なモデル入力へ直接渡さない．

### 4.4 3種類の特徴profile

Optunaは，モデルのハイパーパラメータとともに次の特徴profileも選ぶ．

| profile | 内容 |
| --- | --- |
| `base` | 通常の乗客特徴とグループ人数を使い，目的変数によるencodingは使わない |
| `group_te` | 家族とチケット全体の生存率target encodingを追加する |
| `role_te` | `group_te` に加え，女性・子ども群と成人男性群を分けたtarget encodingを追加する |

target encodingは，同じ家族やチケットの生存傾向を数値化する処理である．学習行では自分自身の件数とラベルを除くleave-one-outを行う．検証foldとテストデータには，学習foldで計算した統計だけを適用する．未知の家族やチケットは，学習側全体の生存率へ戻す．

### 4.5 欠損補完とカテゴリ変換

`TitanicFeatureEngineer` と `build_model_matrices()` は，次の順で前処理する．

- `Age`：敬称・客室等級・性別ごとの中央値，次に客室等級・性別ごとの中央値，最後に全体中央値を使う．
- `Fare`：客室等級ごとの中央値，次に全体中央値を使う．
- `Embarked`：学習側の最頻値を使う．
- 数値列：中央値補完後に `StandardScaler` で標準化する．
- カテゴリ列：最頻値補完後にOne-hot encodingする．頻度の低いカテゴリや未知カテゴリを扱える設定にする．

補完器，標準化器，One-hot encoderは，各foldの学習側だけでfitする．検証側やテスト側から補完値や平均値を学習しない．

### 4.6 比較するモデルとOptuna

調整対象は次の5モデルである．

- CatBoost
- LightGBM
- XGBoost
- ExtraTrees
- Random Forest

ロジスティック回帰は `C=0.5`，`role_te`，`target_smoothing=10.0` の固定設定で比較へ加える．

Optunaは，モデルに応じて木の本数，深さ，学習率，正則化，サンプリング率，クラス重みなどを探索する．通常実行の最終探索は，CatBoost，LightGBM，XGBoostが各150 trial，ExtraTreesとRandom Forestが各80 trialである．

### 4.7 閾値の選び方

F1スコアは，生存確率を0または1へ変える閾値によって変わる．通常実行では `0.05` から `0.95` まで `0.002` 刻みで探索する．

`find_plateau_threshold()` は最大F1との差が `0.002` 以内にある閾値群の中央値を選ぶ．1点だけの偶然の最大値に依存しにくくするためである．

さらに，`cross_fitted_threshold_summary()` は，あるfoldを評価するとき，そのfold以外のOOF予測だけで閾値を決める．閾値を決めたデータと評価するデータを分け，楽観的な評価を抑える．

### 4.8 Nested CV

Nested CVは，外側と内側の2段階で交差検証を行う．

1. 外側の学習部分と検証部分を分ける．
2. 外側の学習部分の中で，内側CVを使ってモデル設定，閾値，blend重み，stacking設定を決める．
3. 決めた設定を変更せず，外側の未使用検証部分を1回だけ評価する．
4. 5-foldを3回繰り返し，合計15個の外側評価をまとめる．

この外側評価を，3方式の期待性能を比較する主な根拠とする．一方，最終提出の具体的なモデル設定は，その後に学習データ全体を使った最終探索で決める．

### 4.9 最終探索と全train再学習

最終探索では，分割シード42，123，2026で各設定を評価し，平均F1とばらつきから `robust_score` を計算する．

- 最良単一モデルは `robust_score` が最大のモデルを選ぶ．
- blendは6モデルのOOF確率に対する非負の重みをOptunaで決める．
- stackingは2段目ロジスティック回帰の `C` と閾値を決める．

設定を固定した後，ベースモデルを学習データ全体で再学習する．単一モデルとblend用の非ロジスティックモデルは5 seedの確率を平均する．stacking用のテスト確率は，3種類のCV seedと各foldで得た予測を平均する．

### 4.10 提出CSVの保存

`validate_submission()` は次を検査する．

- 列順が `PassengerId,Survived` である．
- 179行である．
- IDに重複や欠損がなく，`test_titanic.csv` と同じ順番である．
- `Survived` が欠損のない整数0または1である．

`safe_save_submission()` は同名ファイルがなければ排他的に新規保存する．既存ファイルがある場合は内容が完全一致するときだけ再利用し，異なる場合は `FileExistsError` で停止する．予期しない上書きを防ぐ仕組みである．

## 5．現在保存されている結果

以下は，2026年7月15日時点でノートブックとCSVに保存されている結果である．再実行時にはOptunaの探索順序などにより変化する可能性がある．

### 5.1 Nested CVによる方式比較

| 方式 | 外側fold平均F1 | 標準偏差 | `robust_score` |
| --- | ---: | ---: | ---: |
| 重み付きblend | `0.7882` | `0.0456` | `0.7768` |
| cross-fit stacking | `0.7750` | `0.0347` | `0.7663` |
| 最良単一モデル方式 | `0.7589` | `0.0411` | `0.7486` |

この表は，モデルや設定を選ぶ内側処理から外側検証データを分離した評価である．3方式の期待性能を比較するときは，この表を優先して見る．

### 5.2 全trainを使った最終設定のOOF比較

| 提出方法 | 最終OOF平均F1 | 標準偏差 | `robust_score` | 閾値 |
| --- | ---: | ---: | ---: | ---: |
| 重み付きblend | `0.8133` | `0.0514` | `0.8005` | `0.465` |
| LightGBM単体 | `0.8058` | `0.0448` | `0.7946` | `0.393` |
| cross-fit stacking | `0.8016` | `0.0588` | `0.7869` | `0.432` |

この表は最終提出設定を決めるために学習データ全体を使っている．モデル選択を含むため，Nested CVの外側評価より楽観的になり得る．ファイル名の `nested` は実装全体がNested CVを含むことを表すが，最終LightGBMと閾値を外側foldが直接選んだという意味ではない．

### 5.3 正解配布後の答え合わせ

| 提出ファイル | F1スコア | 正解数 |
| --- | ---: | ---: |
| `submission_best_single_nested.csv` | `0.7571` | 145／179件 |
| `submission_weighted_blend_nested.csv` | `0.7407` | 144／179件 |
| `submission_crossfit_stacking.csv` | `0.7500` | 145／179件 |

この表は `test_titanic_with-answer.csv` が配布された後に `answer.ipynb` で計算した事後評価であり，学習，モデル選択，閾値選択には使用していない．学習データ上で最も高かった方式が，179人のテストデータでも必ず最良になるとは限らない．

## 6．実行方法と注意点

### 6.1 実行方法

このディレクトリを作業ディレクトリとして環境を同期し，ノートブックを上から順に実行する．

```bash
uv sync
uv run jupyter notebook titanic_f1_high_performance.ipynb
```

構造確認だけを短時間で行う場合は，次のように短縮設定を有効にできる．

```bash
TITANIC_FAST_MODE=1 uv run jupyter notebook titanic_f1_high_performance.ipynb
```

通常実行はNested CV，Optuna，複数seed再学習を含む．保存済みログではNested CVだけで約72分，最終探索でさらに約108分かかっており，環境によっては数時間必要になる．

### 6.2 再現性

乱数シードは固定しているが，Optunaを複数jobで並列実行する場合はtrialの完了順が変わり，探索結果が完全には一致しない可能性がある．再実行結果が既存CSVと異なると，`safe_save_submission()` は上書きせず停止する．

### 6.3 `answer.ipynb` の位置付け

`answer.ipynb` は正解配布後だけ使用する．提出CSVと正解を `PassengerId` で結合し，F1スコア，正解数，混同行列を表示する．不足または余分なIDは警告するが，重複ID，179行，0／1整数を厳密に停止条件として検査するわけではない．提出形式の保証は本体ノートブック側の `validate_submission()` で行う．

### 6.4 結果を読むときの注意

- 公平な方式比較にはNested CVの外側評価を使い，最終設定の選択込みOOFとは分けて読む．
- `base` profileではtarget encodingを使わないため，Optunaが選ぶ `target_smoothing` はそのprofileの予測へ影響しない．
- ノートブックの最終パラメータ表は表示幅の都合で一部省略される．表示されていない値を推測して補わない．
- 正解データや兄弟ディレクトリの予測は，学習や提出作成へ使わない．
- `main.py` は機械学習の実行入口ではない．

## 7．要点

この実装は，fold内で完結する特徴量変換，leave-one-out target encoding，Optuna，Nested CV，cross-fitted閾値を組み合わせている．最終的には，LightGBM単体，6モデルの重み付きblend，6モデルを2段階で組み合わせるcross-fit stackingという3方式を提出する．

方式評価用のNested CVと，提出設定を決める全train上の最終OOFは目的が異なる．両者を区別し，正解配布後の答え合わせも別の事後評価として読むことが重要である．
