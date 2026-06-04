# Python seminar を Git / GitHub で管理する方法

このファイルは、この `python_seminar` ディレクトリ全体を Git と GitHub で管理するための手順書です。

## 先に知っておくこと

- Git は、自分の PC 上で変更履歴を管理するツールです。
- GitHub は、Git の履歴をインターネット上に置いてバックアップ・共有するサービスです。
- 基本の流れは `変更する -> git add -> git commit -> git push` です。
- このディレクトリは、2026-06-03 時点では親フォルダ全体が Git 管理されていません。
- ただし、`第2回/lesson_2/.git` が存在します。親フォルダ全体を 1 つのリポジトリにしたい場合は、後述の「重要: 中に別の `.git` がある場合」を確認してください。

## おすすめの管理方針

この `python_seminar` ディレクトリ全体を 1 つの GitHub リポジトリとして管理するのがおすすめです。

理由:

- `第0回`, `第1回`, `第2回` などをまとめて管理できる。
- セミナー資料、ノートブック、メモ、課題を同じ場所で履歴管理できる。
- GitHub に push すればバックアップになる。

注意点:

- `.DS_Store`, `.ipynb_checkpoints`, `.venv`, `__pycache__` などは Git で管理しない方がよいです。
- 公開リポジトリにする場合、著作権のある資料や個人情報、未公開データを含めないようにしてください。
- `.pptx`, `.xlsx`, `.csv`, `.ipynb` は Git で管理できますが、差分は見づらいです。大きすぎるファイルは GitHub に向きません。

## 1. ターミナルでこのディレクトリに移動する

```bash
cd "/Users/fu-riku/Library/CloudStorage/GoogleDrive-fukumoto.riku.fr7@g.ext.naist.jp/マイドライブ/MI_Lab_cloud/python_seminar"
```

現在地を確認します。

```bash
pwd
```

## 2. Git が入っているか確認する

```bash
git --version
```

バージョンが表示されれば OK です。

例:

```text
git version 2.xx.x
```

もし Git が入っていない場合は、macOS なら次のどちらかで入れます。

```bash
xcode-select --install
```

または Homebrew を使っている場合:

```bash
brew install git
```

## 3. Git のユーザー名とメールアドレスを設定する

初回だけ設定します。

```bash
git config --global user.name "あなたの名前"
git config --global user.email "GitHubに登録しているメールアドレス"
```

確認:

```bash
git config --global --list
```

## 4. `.gitignore` を作る

Git で管理しないファイルを指定します。

このディレクトリ直下に `.gitignore` を作り、以下を入れるのがおすすめです。

```gitignore
# macOS
.DS_Store

# Python
__pycache__/
*.py[cod]
*.egg-info/
build/
dist/
wheels/

# Virtual environments
.venv/
venv/
env/

# Jupyter Notebook
.ipynb_checkpoints/

# Editor
.vscode/
.idea/

# Environment variables / secrets
.env
.env.*

# Temporary files
*.tmp
*.log
```

作成コマンド例:

```bash
touch .gitignore
open .gitignore
```

すでに Git 管理に入ってしまった `.DS_Store` などは、あとから `.gitignore` に追加しても自動では外れません。その場合は次のようにします。

```bash
git rm --cached .DS_Store
git rm --cached -r "**/.ipynb_checkpoints"
```

## 5. 重要: 中に別の `.git` がある場合

現在、このディレクトリ内には以下があります。

```text
第2回/lesson_2/.git
```

親フォルダ `python_seminar` 全体を 1 つの Git リポジトリとして管理したい場合、内側の `.git` は基本的に削除した方がわかりやすいです。

まず状態を確認します。

```bash
cd "第2回/lesson_2"
git status
cd ../..
```

もし `第2回/lesson_2` の変更履歴が不要で、親フォルダのリポジトリに統合してよいなら、次を実行します。

```bash
rm -rf "第2回/lesson_2/.git"
```

注意:

- `rm -rf "第2回/lesson_2/.git"` は、`lesson_2` 内の過去の Git 履歴を削除します。
- ファイル自体は消えません。
- 迷う場合は、実行前に `第2回/lesson_2` を別名でコピーしてバックアップしてください。

内側の `.git` を残したまま親フォルダで `git add` すると、通常のフォルダとしてではなく、別リポジトリやサブモジュールのように扱われて混乱しやすいです。

## 6. このディレクトリを Git 管理にする

`python_seminar` 直下で実行します。

```bash
git init
```

現在の状態を確認します。

```bash
git status
```

## 7. 最初のコミットを作る

まず、Git に追加される予定のファイルを確認します。

```bash
git status
```

問題なければ全体を追加します。

```bash
git add .
```

追加内容を確認します。

```bash
git status
```

最初のコミットを作ります。

```bash
git commit -m "Initial commit"
```

## 8. GitHub に新しいリポジトリを作る

GitHub の画面で作る方法:

1. GitHub にログインする。
2. 右上の `+` から `New repository` を選ぶ。
3. Repository name に例として `python_seminar` と入力する。
4. Public / Private を選ぶ。
5. `README`, `.gitignore`, `license` は GitHub 側では作らない。
6. `Create repository` を押す。

おすすめは `Private` です。

理由:

- セミナー資料、課題、データ、メモが含まれる可能性があるため。
- 公開してよい内容か後から確認できるため。

## 9. ローカル Git と GitHub をつなぐ

GitHub でリポジトリを作ると、URL が表示されます。

HTTPS の例:

```text
https://github.com/ユーザー名/python_seminar.git
```

このディレクトリで次を実行します。

```bash
git branch -M main
git remote add origin https://github.com/ユーザー名/python_seminar.git
git push -u origin main
```

`ユーザー名` は自分の GitHub ユーザー名に置き換えてください。

## 10. 日常の使い方

作業前に最新状態を取得します。

```bash
git pull
```

ファイルを編集します。

変更状態を確認します。

```bash
git status
```

変更内容を確認します。

```bash
git diff
```

変更を追加します。

```bash
git add .
```

コミットします。

```bash
git commit -m "第2回の課題メモを更新"
```

GitHub に送ります。

```bash
git push
```

## 11. コミットメッセージの書き方

短く、何をしたかがわかる文にします。

例:

```text
第1回資料を追加
第2回の課題ノートブックを更新
Pandas演習データを追加
READMEに環境構築手順を追記
不要なDS_Storeを削除
```

英語でも日本語でも構いません。自分が後で見てわかることが重要です。

## 12. ファイルを消した・戻したい場合

まだ `git add` していない変更を取り消す:

```bash
git restore ファイル名
```

`git add` を取り消す:

```bash
git restore --staged ファイル名
```

過去のコミット履歴を見る:

```bash
git log --oneline
```

直前のコミットで何を変えたか見る:

```bash
git show
```

## 13. ノートブック `.ipynb` を管理するときの注意

Jupyter Notebook は Git で管理できますが、差分が読みにくいです。

おすすめ:

- 実行結果が巨大なノートブックは、コミット前に出力を整理する。
- 不要な `.ipynb_checkpoints` は `.gitignore` で除外する。
- 重要なコードは、可能なら `.py` ファイルにも分ける。

ノートブックの出力を消したい場合は、Jupyter 上で次を実行します。

```text
Kernel -> Restart & Clear Output
```

その後に保存してコミットします。

## 14. GitHub に上げない方がよいもの

以下は GitHub に上げないように注意してください。

- パスワード
- API キー
- 個人情報
- 学内限定の資料
- 公開許可のない論文 PDF
- 公開許可のない講義資料
- 大きすぎるデータファイル
- `.env`
- 仮想環境 `.venv`

もし間違えて秘密情報を commit / push した場合は、単にファイルを削除して commit するだけでは履歴に残ります。すぐに相談してください。

## 15. よく使うコマンドまとめ

```bash
# 状態確認
git status

# 変更内容確認
git diff

# 変更を追加
git add .

# コミット
git commit -m "メッセージ"

# GitHub へ送る
git push

# GitHub から取ってくる
git pull

# 履歴を見る
git log --oneline

# リモート確認
git remote -v
```

## 16. 初回セットアップの最短コマンド例

以下は、`第2回/lesson_2/.git` の履歴を削除して、親フォルダ全体を 1 つのリポジトリにする場合の例です。

実行前に、GitHub で空の `python_seminar` リポジトリを作成してください。

```bash
cd "/Users/fu-riku/Library/CloudStorage/GoogleDrive-fukumoto.riku.fr7@g.ext.naist.jp/マイドライブ/MI_Lab_cloud/python_seminar"

# 内側の Git 履歴を削除する場合のみ
rm -rf "第2回/lesson_2/.git"

# Git 管理開始
git init

# ブランチ名を main にする
git branch -M main

# ファイル追加
git add .

# 最初のコミット
git commit -m "Initial commit"

# GitHub と接続
git remote add origin https://github.com/ユーザー名/python_seminar.git

# GitHub に送る
git push -u origin main
```

## 17. このディレクトリで特に気をつけること

このディレクトリには以下のようなファイルがあります。

- `.pptx`
- `.xlsx`
- `.csv`
- `.ipynb`
- `.md`
- `.py`

これらは Git で管理できます。

ただし、`.pptx` や `.xlsx` は変更差分が見づらいため、更新したらコミットメッセージを具体的にしてください。

例:

```bash
git commit -m "第2回スライドに次回予告を追加"
```

また、Google Drive 配下のフォルダを Git 管理する場合、同期中の一時ファイルが混ざることがあります。`git status` で見慣れない一時ファイルが出てきたら、コミット前に確認してください。

