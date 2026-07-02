# python_seminar/ — 後輩向け Python セミナー教材

研究室の後輩（M1）向け Python・機械学習セミナーの教材リポジトリ。
**MI_Lab_cloud 直下にあるが独立した git リポジトリ**（remote: github.com/fukumoto-riku/python_seminar、ブランチ main）。

## 共通ルール

- 応答・教材・ドキュメントは日本語で作成する（コード識別子・コミットメッセージは英語可）。
- Google Drive 同期領域: パスに日本語を含むため、シェルコマンドではパスを必ず引用符で囲む。大量ファイル生成・一括リネーム・一括削除をしない。
- 既存ファイルの上書き・削除は実行前に確認する。`.DS_Store` 等のシステムファイルに触れない。

## 構成

- `第N回/` … 各回の資料。トピック解説の pptx + ハンズオン用の `lesson_N/`
- `第N回/lesson_N/` … uv 管理の Python プロジェクト（pyproject.toml / uv.lock / .python-version=3.11 / ノートブック / データ csv）
- `Pandas_100_knocks-master/`、`化学のためのPythonによるデータ解析・機械学習入門/` … 外部教材の取り込み。**改変禁止（参照のみ）**
- `Git_GitHub管理ガイド.md` … git / GitHub 初心者向け手順書（後輩向け教材を兼ねる）
- `メモ.md` … セミナー運営メモ（担当 M1 の役割・宿題方針）

## セミナー運営（メモ.md より）

- 担当 M1 は次回予告で提示された単語を調べ、1単語1枚程度・全2〜3枚のスライドにまとめ、M2 と協力して完成させる。
- 宿題は M1 全員が行う。わからない場合は「どこが・なぜわからないのか」を明確にする。

## 教材作成の規約

- 新しい回は `第N回/` を作り、ハンズオンは `uv init` で `lesson_N/` を作成する（既存 lesson_2〜7 の構成に合わせる）。
- ノートブックは日本語の説明セルを挟みながら段階的に進む構成にする。
- `.venv` は各 lesson 内に作られ Drive 同期が重いため、`uv sync` や依存インストールは必要時のみ実行する。
- 依存の追加は各 lesson の pyproject.toml で管理する（`uv add`）。

## Git 運用

- ブランチは main のみ。コミットメッセージは既存の流儀に合わせる（例: `add lesson7`, `update lesson 6 materials`）。
- **push はユーザーの確認を取ってから**行う。
- `.venv` / `__pycache__` / `.ipynb_checkpoints` / `.DS_Store` 等は .gitignore 済み。
