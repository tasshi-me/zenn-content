---
title: "Gitリポジトリをモノレポのサブディレクトリに移行する"
emoji: "🚚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git", "monorepo"]
published: false
---

## はじめに

複数のリポジトリをモノレポに統合したいとき、単純にコードをコピーするだけでは Git の履歴が失われてしまいます。

履歴を保持したい理由はいくつかあります。

- `git blame` や `git log` で変更の経緯を追えるようにするため
- 過去の実装者への敬意を示すため
- 外部 Contributor や退職済み Contributor との権利関係のトラブルを避けるため

この記事では、`git subtree` を使って別リポジトリのコードをコミット履歴ごとサブディレクトリに移行する方法を紹介します。

## 前提条件

この記事では以下の構成を想定しています。

- **移行元リポジトリ**: `source-repo`
  - 移行したいコードは `packages/target-package` ディレクトリにある
- **移行先リポジトリ**: `dest-repo`
  - `src/new-location` ディレクトリに移行する

## 手順

### 1. 移行元リポジトリで subtree split を実行する

移行元リポジトリで、移行したいディレクトリの履歴だけを切り出したブランチを作成します。

```bash
# 移行元リポジトリで実行
cd source-repo

git subtree split --prefix packages/target-package --branch subtree-target-package
git push origin subtree-target-package
```

`--prefix` には移行したいディレクトリを指定します。このコマンドで、指定したディレクトリに関するコミットだけを持つブランチが作成されます。

### 2. Contributor 一覧を抽出する

移行元のコミット履歴から Contributor 一覧を抽出します。これは後でマージコミットに `Co-authored-by` を付けるために使います。

```bash
# 移行元リポジトリの subtree ブランチで実行
cd source-repo
git switch subtree-target-package

git log \
  # Author と Co-authored-by の行を抽出
  | grep -e 'Author:' -e 'Co-authored-by:' \
  # 前後の空白とプレフィックスを削除
  | sed -e 's/^ *//' -e 's/ *$//' -e 's/Author: //' -e 's/Co-authored-by: //' \
  # 重複を削除
  | awk '!x[$0]++' \
  # 時系列順に並べ替え（古い順）
  | tac \
  > authors-full-list.txt

# Co-authored-by 形式に変換
cat authors-full-list.txt | sed -e 's/^/Co-authored-by: /' > co-authored-by.txt
```

### 3. 移行先リポジトリで subtree add を実行する

移行先リポジトリで、先ほど作成したブランチを取り込みます。

```bash
# 移行先リポジトリで実行
cd dest-repo

# フィーチャーブランチを作成
git switch -c feat/import-target-package

# 移行元リポジトリをリモートに追加
git remote add source-repo git@github.com:your-org/source-repo.git

# subtree add で取り込む
git subtree add --prefix src/new-location source-repo subtree-target-package
```

これで、`src/new-location` ディレクトリに移行元のコードが履歴ごと取り込まれます。

### 4. モノレポに合わせてコードを修正する

取り込んだコードは元のリポジトリの構成そのままなので、モノレポに合わせて修正が必要です。

- ディレクトリ構成の調整
- 依存関係の修正（`package.json` など）
- 設定ファイルの更新
- インポートパスの修正

修正内容は同じブランチにコミットしていきます。

### 5. PR を作成してマージする

移行先リポジトリでPRを作成し、マージします。マージする際に、コミットメッセージに `co-authored-by.txt` の内容を含めることで、全 Contributor を共著者として記録できます。

```
chore: import target-package from source-repo

Co-authored-by: Alice <alice@example.com>
Co-authored-by: Bob <bob@example.com>
...
```

## 補足

### なぜ Co-authored-by を付けるのか

`git subtree add` で取り込んだコミット履歴は保持されますが、GitHub の Contributors グラフには反映されません。マージコミットに `Co-authored-by` を付けることで、GitHub 上でも Contributor として認識されるようになります。

### リモートの削除

移行が完了したら、追加したリモートは削除しても構いません。

```bash
git remote remove source-repo
```

## まとめ

`git subtree` を使うことで、別リポジトリのコードをコミット履歴ごとモノレポのサブディレクトリに移行できます。また、`Co-authored-by` で引き継ぐことで、過去の Contributor の記録も残しながら移行できます。
