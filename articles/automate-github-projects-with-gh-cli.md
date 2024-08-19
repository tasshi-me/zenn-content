---
title: "GitHub ProjectsをGitHub CLIで自動化、そしてその先へ"
emoji: "🪄"
type: "tech"
topics: ["github", "githubactions", "go"]
publication_name: "cybozu_ept"
published: false
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '24](https://cybozu.github.io/summer-blog-fes-2024/) (kintone Stage) DAY 7の記事です。
- DAY 6の記事はこちら→ [kintone ナビゲーション / コミュニケーション系チームの紹介](https://blog.cybozu.io/entry/2024/08/18/080000_1)
- DAY 8の記事はこちら→ Comming Soon!
:::

こんにちは、サイボウズ株式会社の [tasshi](https://twitter.com/tasshi_me) です。
普段は kintone 新機能開発チームの拡張基盤🧩サブチームで API や SDK などのプラグイン開発者向け機能を担当しています。

今回 GitHub 関連の記事を書くに当たり、サイボウズ 生産性向上チームの Publication にお邪魔しています。
生産性向上チームの投稿する[生産性向上 セイサンシャインビーチ Stage](https://cybozu.github.io/summer-blog-fes-2024/) もぜひご覧ください！

## 概要

この記事では GitHub CLI を使って GitHub Projects を自動化し、それを再利用しやすくパッケージングする方法について紹介します。

- GitHub Projects を自動化することで、細かい手作業を減らし、本来の開発業務に集中できる
- Projects は GitHub CLI + GitHub Actions を利用して、簡単に自動化できる
- 自動化のロジックをアクションや GitHub CLI 拡張機能にすることで、メンテナンスコストを集約し、再利用しやすくできる

:::message
この記事は [Sansan vs サイボウズ 開発生産性Tips夏祭り](https://cybozu.connpass.com/event/322718/) の登壇内容を再構成したものです。
- [GitHub Projectsを自動化するGitHub CLIテクニック - Speaker Deck](https://speakerdeck.com/tasshi/automate-github-projects-with-github-cli)
:::

## GitHub Projects について

Projects は GitHub 公式のプロジェクト管理ツールです。 
GitHub と統合されたプロジェクト管理ツールとなっており、複数の Repository 上の Issue/Pull Request を集約的に扱うことができます。

https://docs.github.com/ja/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects

また、カスタムフィールドを使用することで締切日やスプリント、優先度などのメタデータを追加することもできます。

https://docs.github.com/ja/issues/planning-and-tracking-with-projects/understanding-fields

### 拡張基盤🧩チームの Projects の使い方

私の所属する拡張基盤🧩チームでは、主にプロダクトバックログ(PBI)の管理に Projects を利用しています。

![](/images/automate-github-projects-with-gh-cli/project-board.png)

また、OSS として公開している SDK/CLI などのリポジトリに登録される Issue/Pull Request も Projects に集約して管理しています。

## Projects を自動化する

Projects 上でプロジェクト/タスクを管理する上で、様々な細かい作業が発生します。
（Issue を Projects に追加する、ステータスを変更する、Issue を Close する、など）

このような細かい手作業を減らしていくことは、プロジェクトの実際の作業に集中することに繋がるため、Projects のベストプラクティスとして推奨されています。

> 意味の無い作業に費やす時間を減らし、プロジェクト自体にかける時間を増やすために、タスクを自動化できます。 手動でやることを覚えておく必要が減れば、それだけプロジェクトは最新の状態に保たれるようになります。

https://docs.github.com/ja/issues/planning-and-tracking-with-projects/learning-about-projects/best-practices-for-projects#use-automation

Projects の自動化には大きく次の2つの方法があります。

- 組み込みの自動化
- API/Actions を使用した自動化

### 組み込みの自動化 (Workflows)

**組み込みの自動化**は GitHub が提供するビルトインの自動化ワークフローです。
特定のイベントに基づいた処理や、特定の条件に合致するアイテムへの処理が用意されています。

https://docs.github.com/ja/issues/planning-and-tracking-with-projects/automating-your-project/using-the-built-in-automations

具体的には、以下のようなワークフローがあります。

| ワークフロー名                | 処理内容                                          |
|------------------------|-----------------------------------------------|
| Item added to project  | Issue/Pull Requestがプロジェクトに追加されたときに、ステータスを設定する |
| Item reopened          | Issue/Pull RequestがReopenされたときに、ステータスを設定する    |
| Item closed            | Issue/Pull RequestがCloseされたときに、ステータスを設定する     |
| Code changes requested | Pull Requestがコード変更をリクエストされたときに、ステータスを設定する     |
| Code review approved   | Pull RequestがApproveされたときに、ステータスを設定する         |
| Pull request merged    | Pull RequestがMergeされたときに、ステータスを設定する           |
| Auto-archive items     | 特定の条件を満たすアイテムを、アーカイブする                        |
| Auto-add to project    | 特定の条件を満たすIssue/Pull Requestを、プロジェクトに追加する      |
| Auto-close issue       | ステータスが条件を満たすIssueを、Closeする                    |

これらのワークフローだけで基本的な操作を自動化できますが、より複雑な処理には API を使う必要があります。

### API/Actionsを使用した自動化

**API** を利用して個別のチームの開発プロセスに合わせた柔軟な自動化を実現することができます。

Projects の操作には GitHub API v4 (GraphQL) を利用します。
GitHub のドキュメントでは、[GitHub CLI](https://docs.github.com/ja/github-cli/github-cli/about-github-cli)の`gh api graphql` コマンドを使って API を呼び出す方法を紹介しています。

https://docs.github.com/ja/issues/planning-and-tracking-with-projects/automating-your-project/using-the-api-to-manage-projects

自動化された処理を定期実行するには、GitHub CLI を利用するシェルスクリプトを GitHub Actions 上で実行します。

https://docs.github.com/ja/issues/planning-and-tracking-with-projects/automating-your-project/automating-projects-using-actions

## 自動化の例: スプリントを自動更新する

拡張基盤🧩チームでは、PBIのステータスに応じて次のようにスプリント（繰り返しフィールド）を更新するプロセスになっていました。

- PBI に着手したときは、現在のスプリントを入力する
- PBI がスプリントを跨いだ場合は、スプリントを更新する
- PBI を Icebox/Ready に戻したときは、スプリントをクリアする

スプリントの情報はストーリーポイントの集計や過去のベロシティの確認などに利用します。
しかしながら、開発に集中しているとスプリントの更新を忘れることが少なくありませんでした。

そのため、上記のスプリントの更新を自動化することにしました。

### GitHub CLI + Actions による自動化

GitHub CLI を使ってスプリントを更新するシェルスクリプトを作成します。

以下のようなフローを実現します。

1. プロジェクトの情報を取得
2. スプリントフィールドの情報を取得
3. プロジェクトのアイテムを取得
4. 現在のスプリントを入力/スプリントをクリア

基本的はプロジェクト操作は[`gh project view`コマンド](https://cli.github.com/manual/gh_project)で実装できますが、コマンドで非対応の処理は GraphQL 呼び出しで実装します。

このシェルスクリプトを Actions で定期実行します。

![スプリント更新のフローチャート](/images/automate-github-projects-with-gh-cli/script-flowchart.png)

### 問題点

こうして無事スプリントの更新を自動化することができました。
しかし、主にメンテナンス面で問題があります。

#### 長すぎるシェルスクリプト

今回作成したシェルスクリプトは100行を超えるものになっています。

:::details 作成したシェルスクリプト
```shell:assign-sprint.sh
#!/bin/bash
set -euxo pipefail

cd "$(dirname "$0")"
temp_dir=$(mktemp -d)
echo $temp_dir
trap 'rm -rf $temp_dir' EXIT

owner="myorg"
project_number=123

project_id=$(
  gh project view "${project_number}" \
  --owner "${owner}" \
  --format json \
  --jq '.id'
)

current_sprint=$(
  gh api graphql \
  --field project_id="${project_id}" \
  -f query='
    query($project_id: ID!){
    node(id: $project_id) {
      ... on ProjectV2 {
        field(name: "Sprint") {
          ... on ProjectV2IterationField {
            id
            name
            configuration {
              iterations {
                startDate
                id
              }
            }
          }
        }
      }
    }
  }' \
  --jq '.data.node.field.configuration.iterations[0]'
)

current_sprint_id=$(echo "${current_sprint}" | jq -r '.id')
current_sprint_startDate=$(echo "${current_sprint}" | jq -r '.startDate')

sprint_field_id=$(
  gh project field-list "${project_number}" \
  --owner "${owner}" \
  --format json \
  --jq '
      .fields[]
      | select(
        .name=="Sprint"
         and .type=="ProjectV2IterationField"
         )
      | .id
    '
)

gh project item-list "${project_number}" \
--owner "${owner}" \
-L 50000 \
--format json \
--jq '
       .items[]
       | select(
          (.repository != null)
          and .status != null
        )
       | {id,status,title,sprint}
     ' \
| jq -s '.' > "${temp_dir}/items.json"

cat "${temp_dir}/items.json" | jq '
  .[]
  | select(
         (
           (.status | endswith("Icebox"))
           or (.status | endswith("Ready"))
         ) and .sprint != null
  )
' | jq -s '.' > "${temp_dir}/items_to_clear.json"

cat "${temp_dir}/items.json" | jq --arg startDate "${current_sprint_startDate}" '
  .[]
  | select(
         (
           (.status | endswith("In progress"))
           or (.status | endswith("In review"))
           or (.status | endswith("In testing"))
           or (.status | endswith("In AC Check"))
         ) and .sprint.startDate != $startDate
  )
' | jq -s '.' > "${temp_dir}/items_to_assign.json"


items_to_clear_len=$(cat "${temp_dir}/items_to_clear.json" | jq 'length')
items_to_assign_len=$(cat "${temp_dir}/items_to_assign.json" | jq 'length')
echo "items_to_clear: ${items_to_clear_len}"
echo "items_to_assign: ${items_to_assign_len}"

export sprint_field_id
export project_id
export current_sprint_id

if [ ${items_to_clear_len} -gt 0 ]; then
  cat "${temp_dir}/items_to_clear.json" | \
  jq '.[] | .id, .title, .status' \
  | xargs -n3 bash -c '
  id=${1} && \
  title=${2} && \
  status=${3} && \
  echo id: ${id}, status: ${status}, title: ${title} && \
  gh project item-edit \
  --id ${id} \
  --field-id ${sprint_field_id} \
  --project-id ${project_id} \
  --clear
  ' bash
fi

if [ ${items_to_assign_len} -gt 0 ]; then
  cat "${temp_dir}/items_to_assign.json" | \
  jq '.[] | .id, .title, .status' \
  | xargs -n3 bash -c '
  id=${1} && \
  title=${2} && \
  status=${3} && \
  echo id: ${id}, status: ${status}, title: ${title} && \
  gh project item-edit \
  --id ${id} \
  --field-id ${sprint_field_id} \
  --project-id ${project_id} \
  --iteration-id ${current_sprint_id}
  ' bash
fi
```
:::

主な原因は、カスタムフィールドを操作するためにフィールドIDやイテレーションIDなどの、Web GUI では表示されない内部プロパティを GraphQL 呼び出しで取得する必要があったことです。

スプリントの更新漏れを防ぐという目的のためだけにしては少し割りに合わないように思います。

#### 他チームへの展開

同様の自動化を他チームにも利用してもらう場合に、シェルスクリプトではコピペ運用になってしまいがちです。

コピーされたシェルスクリプトの運用は難しいです。
まず、大元のチームでシェルスクリプトの改善・修正がコピー先のチームに反映されにくいです。
コピー先で独自のパッチを行なっている場合、追従はより困難なものになります。

また、何らかの理由でスクリプトが動かなくなった場合に修正できない/修正に学習コストがかかる状態になります。
このコストはコピー先のチームそれぞれに重複してかかることになります。

上記のような問題は、徐々にチーム内に「自動化の恩恵より保守コストが高いのでは」という疑念を生み、最終的に自動化の廃止に繋がります。
（後から入ったメンバーが再び手作業を自動化するという場合も、、、）

このような事態を防ぐためにも自動化のメンテナンスコストは1箇所に集約されているべきです。
自動化のロジック部分を適切にパッケージングすることで、再利用のコストを減らし、保守すべきコードを集約させることができます。

:::message alert
ここで言及している自動化は日常的な手作業を代替する程度の、作成者および有志で保守できる小規模のものです。
:::

## GitHub CLI 拡張機能 (gh extensions)

GitHub CLI を使った処理をパッケージングする方法として、GitHub CLI 拡張機能を紹介します。
拡張機能を導入すると、GitHub CLI で追加のカスタムコマンドを利用することができます。

https://docs.github.com/ja/github-cli/github-cli/using-github-cli-extensions

GitHub の[`gh-extensions`トピック](https://github.com/topics/gh-extension)で様々な拡張機能が公開されています。

### GitHub CLI 拡張機能の作成

拡張機能の作成は大まかに以下のような手順になります。

- 処理を記述して実行可能ファイルを作成する
- `gh-`から始まるリポジトリで公開する
- リポジトリに tag を作成してリリースする

詳細な手順は GitHub のドキュメントをご確認ください。
https://docs.github.com/ja/github-cli/github-cli/creating-github-cli-extensions

実行可能ファイルが生成できればプログラミング言語は自由に選択できます。
そのため、手軽にシェルスクリプトの内容を切り出してそのまま拡張機能にすることも可能です。
（Go で開発する場合は、GitHubの提供するヘルパーライブラリを利用できます）

### アクションとの相互再利用性について

GitHub Actions で実行する場合、他の選択肢としてアクション（third-party actions）があります。
`uses`ステップで簡単に利用できるようになるため、Actions での実行がメインになるのであればアクションを作成することをおすすめします。
（一方でローカルでの実行が主であれば GitHub CLI 拡張機能が良いです。）

アクションと GitHub CLI 拡張機能のどちらにも提供したい場合は、ロジック部分のコアライブラリを JavaScript で開発しておくと再利用しやすいです。
[JavaScript アクション](https://docs.github.com/ja/actions/sharing-automations/creating-actions/creating-a-javascript-action)で開発できますし、Node.js の [SEA](https://nodejs.org/api/single-executable-applications.html) などで実行ファイルを作成すれば GitHub CLI 拡張機能にもできます。

GitHub CLI 拡張機能を先に開発した場合は、[複合アクション](https://docs.github.com/ja/actions/sharing-automations/creating-actions/creating-a-composite-action)から拡張機能を呼び出す形でアクションを作成できます。

## gh-iteration

作成した拡張機能が [gh-iteration](https://github.com/tasshi-me/gh-iteration) です。
標準の `gh project` で繰り返しフィールドを操作するには GraphQL を使う必要がありました。
gh-iteration を利用すると、繰り返しフィールドを簡単に、また一括で操作できます。

https://github.com/tasshi-me/gh-iteration

現在は、以下の6つのコマンドが搭載されています。

| コマンド       | 機能                        |
|------------|---------------------------|
| field-list | 繰り返しフィールドの一覧を表示する         |
| field-view | 繰り返しフィールドの設定を閲覧する         |
| item-edit  | プロジェクトアイテムを更新する           |
| item-view  | プロジェクトアイテムを閲覧する           |
| items-edit | 複数のプロジェクトアイテムを一括更新する      |
| list       | 繰り返しフィールドのイテレーションの一覧を表示する |

### items-edit コマンド

この中でも、`items-edit` は複数アイテムのイテレーションを一括で更新するコマンドで、今回自動化したい処理を1コマンドで実行できます。
- 更新対象のアイテムをクエリで絞り込む
  - 例: `(Item.Type == "ISSUE") && Item.IsArchived`
- イテレーションはフィールドIDではなく、フィールド名やエイリアスを使用可能
  - `--current`: 現在
  - `--clear`: クリア
  - `--iteration <フィールド名>`: フィールド名指定

### exper-lang/expr

アイテムの絞り込み機能を実現するに当たって、[expr-lang/expr](https://github.com/expr-lang/expr) という Expression Language ライブラリを利用しました。

プロジェクトアイテムの構造体に対してクエリを評価した結果を真偽値で取得しています。
以下のように柔軟なクエリを使った絞り込みが実現できました。

- `Item.Type == "ISSUE"`
- `Item.StoryPoint > 3 && Item.StoryPoint < 7`
- `Item.Repository endsWith "sdk"`

## スクリプトを置き換えていく

作成した拡張機能を使って、シェルスクリプトを置き換えていきます。

![](/images/automate-github-projects-with-gh-cli/replace-script.png)

gh-iteration では、最初は list コマンドのような基礎的な小さいコマンドしか搭載しませんでした。
スクリプトの一部を置き換える基礎的なコマンドを実装し、その後に複数のコマンドを置き換える包括的なコマンドを実装することで、段階的に拡張機能を開発していきました。

![](/images/automate-github-projects-with-gh-cli/replace-script-iteration.png)

最終的にシェルスクリプトのコード量を137行から26行まで減らすことができました。

:::details 最終的なシェルスクリプト
```shell:assign-sprint.sh
#!/bin/bash
set -euxo pipefail

owner="myorg"
project_number=123
sprint_field="Sprint"

gh iteration items-edit \
  --owner "${owner}" \
  --project "${project_number}" \
  --field "${sprint_field}" \
  --query "(Item.Type == \"ISSUE\") && (
     (Item.Fields.Status.Name endsWith \"In progress\")
     || (Item.Fields.Status.Name endsWith \"In review\")
     || (Item.Fields.Status.Name endsWith \"In testing\")
     || (Item.Fields.Status.Name endsWith \"In AC Check\"))" \
  --current

gh iteration items-edit \
  --owner "${owner}" \
  --project "${project_number}" \
  --field "${sprint_field}" \
  --query "(Item.Type == \"ISSUE\") && (
   (Item.Fields.Status.Name endsWith \"Icebox\")
   || (Item.Fields.Status.Name endsWith \"Ready\"))" \
  --clear
```
:::

## 終わりに

GitHub Projects を自動化し、その処理を再利用可能な形にパッケージングする方法を紹介しました。
Projects は Repository や Actions との親和性が高く、プロセス自動化のハードルが低いのが魅力的ですね。
メンテナンスコストを上手くコントロールしつつ、どんどん手作業を減らして開発生産性を上げていきましょう！

kintone の新機能開発や改善活動に興味のある方は、以下の記事もご覧ください！

https://speakerdeck.com/cybozuinsideout/kintone-development-team-recruitment-information
https://cybozu.co.jp/recruit/entry/career/web-engineer-kintone.html

---
https://cybozu.github.io/summer-blog-fes-2024/
