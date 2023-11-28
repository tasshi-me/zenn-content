---
title: "【Renovate】サードパーティアクションをコミットSHAで管理する"
emoji: "🧱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "renovate"]
published: true
---

## 結論

renovate.jsonに以下のように指定します。

```json:renovate.json
{
  "packageRules": [
    {
      "matchDepTypes": ["action"],
      "excludePackagePrefixes": ["actions/"],
      "pinDigests": true
    }
  ]
}
```

以下、背景情報です。

## GitHub Actions: サードパーティアクションの利用方法

GitHub Actionsではコミュニティで作成されたサードパーティアクションを利用できます。
サードパーティアクションはコミットSHAでバージョン指定することが推奨されています。

> 現在、アクションを不変のリリースとして使用する唯一の方法は、アクションを完全なコミット SHA にピン止めすることです。 特定の SHA にピン止めすると、有効な Git オブジェクトペイロードに対して SHA-1 衝突を生成する必要があるため、悪意のある人がアクションのリポジトリにバックドアを追加するリスクを軽減できます。

https://docs.github.com/ja/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions

これは安定性とセキュリティの観点から重要です。
こちらの記事に詳しくまとまっていたので、そちらを参照してください。

https://zenn.dev/snowcait/articles/53ec922a414dde

## Renovate: アクションのバージョン指定方法

[Renovate](https://www.mend.io/renovate/) はライブラリの自動アップデートツールです。

RenovateではアクションのバージョンをコミットSHAで指定しつつ、アップデートはバージョン（タグ）に追従させることができます。

https://docs.renovatebot.com/modules/manager/github-actions/#additional-information

`uses` の行末尾に `# renovate: tag=<tagname>` の形式でコメントを追加すると、バージョンに追従してアップデートが行われるようになります。

```yml:.github/workflows/workflow.yml
steps:
  - uses: actions/setup-node@64ed1c7eab4cce3362f8c340dee64e5eaeef8f7c # v3.6.0
```

:::message
タグの指定がない場合、Renovateのアップデートはデフォルトブランチの最新コミットに追従します（設定依存）
:::

また、自動的にコミットSHA指定にしたい場合は `helpers:pinGitHubActionsDigests` プリセットを利用します。

```json
{
  "extends": ["helpers:pinGitHubActionDigests"]
}
```

## サードパーティアクションをコミットSHAでバージョン指定する

先ほどの `helpers:pinGitHubActionsDigests` プリセットでは全てのアクションのバージョンがコミットSHAで指定されます。
これは非常に堅牢ですが、`actions/checkout` のような公式アクションではメジャーバージョン指定で楽をしたいです。

そのため、`helpers:pinGitHubActionsDigests` プリセットを少し改変して設定を作成します。

https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests

変更後の設定がこちら↓

```json:renovate.json
{
  "packageRules": [
    {
      "matchDepTypes": ["action"],
      "excludePackagePrefixes": ["actions/"],
      "pinDigests": true
    }
  ]
}
```

`excludePackagePrefixes` で [actions org](https://github.com/actions)内のアクションを対象外にしています。
自分で公開しているアクションがあれば、そちらも対象外にしても良いかもしれません。

これにより、サードパーティアクションのみコミットSHAで指定されるようになります。

```yml
steps:
  - uses: actions/checkout@v3
  - uses: ruby/setup-ruby@55283cc23133118229fd3f97f9336ee23a179fcf # v1.146.0
```

## 終わりに

ここまで読んでいただきありがとうございました。
