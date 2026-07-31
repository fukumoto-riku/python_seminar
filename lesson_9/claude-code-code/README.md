# claude-code-code — QM9 バンドギャップ予測コンペ解法

第9回 Python セミナーのコンペ（`../lesson9_regulation.md`）に対する Claude Code 作の解法である．SMILES から RDKit で特徴量を生成し，LightGBM・XGBoost・CatBoost の 3 モデルと OOF 重み最適化ブレンドで `gap_eV` を予測する．乱数シードは推奨値の 8 に固定している．

## ファイル構成

| ファイル | 内容 |
| --- | --- |
| `qm9_gap_prediction.ipynb` | 解法ノートブック（実行済み，出力セル付き） |
| `submission.csv` | 本命の提出値（重み最適化ブレンド） |
| `submission_2_lightgbm_single.csv` | 提出候補 2（LightGBM 単体） |
| `submission_3_mean_blend.csv` | 提出候補 3（3 モデル単純平均） |
| `code-note.md` | コード解説（特徴量・モデル・検証・再現方法） |
| `qm9_bandgap_train.csv` ほか | 配布データのコピー |

## 結果（OOF MAE，5-fold，seed = 8）

| モデル | OOF MAE [eV] |
| --- | ---: |
| 重み最適化ブレンド（`submission.csv`） | 0.1912 |
| 単純平均ブレンド | 0.1915 |
| CatBoost | 0.1963 |
| XGBoost（チューニング後） | 0.1967 |
| LightGBM（チューニング後） | 0.2015 |
| ベースライン LightGBM | 0.2054 |

ブレンド重みは LightGBM 0.22，XGBoost 0.34，CatBoost 0.44 である．

## 実行方法

```bash
uv sync                                   # 初回のみ
uv run jupyter lab                        # ノートブックを開いて上から実行
```

フル実行の実測は約 2.5 時間（Apple Silicon，CPU のみ）．動作確認のみの場合は `QM9_QUICK=1` を設定して実行すると 10 分弱で完了する（精度は参考値）．正解付きテストデータ配布後の評価手順を含む詳細は `code-note.md` を参照．
