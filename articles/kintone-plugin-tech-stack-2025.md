---
title: "kintoneプラグイン開発の技術スタックと開発環境"
emoji: "🧩"
type: "tech"
topics: ["kintone", "plugin", "typescript", "frontend"]
published: true
---

:::message
この記事は、[kintone Advent Calendar 2025](https://qiita.com/advent-calendar/2025/kintone) の3日目の記事です。
- 2日目の記事はこちら→ [kintoneローカルMCPでAIサンタを召喚してみた話](https://qiita.com/shinshin3/items/064472a5e39fde08d2d2)
- 4日目の記事はこちら→ Coming soon!
:::

こんにちは、[tasshi](https://twitter.com/tasshi_me) です。

この記事では、kintoneプラグイン開発において、私がどのような技術スタック・開発環境を採用しているかを紹介します。
併せて、サイボウズから提供されているプラグイン開発者向けの支援ツールなどについても紹介します。

**プラグイン開発の技術選定について、皆さんの意見や経験をコメントで共有していただけると嬉しいです!**

:::message
本記事の内容は筆者個人の見解であり、所属企業の公式見解を代表するものではありません。
:::

## kintoneプラグイン開発はフロントエンド技術の上に構築されている

kintoneのプラグイン開発は、フロントエンド開発の技術スタックの上に成り立っています。

そのため、普段のフロントエンド開発で使っている技術やツールをそのまま利用できる点が大きな特徴です。
一方で、kintoneプラグイン特有の制約も存在するため、いくつか注意点があります。

今回は、業界で一般的な技術の選択肢を確認しつつ、kintoneのプラグイン開発において何が適しているかを整理します。 

:::message
実際の開発では本記事で紹介するもの以外にもテストフレームワークや、CI/CD、依存関係管理ツールなど様々なツールが関わってきますが、本記事では触れません。

また、kintoneのエコシステムからも様々な開発支援ツールが公開されています。
これらについても紹介したいところなのですが、今回はフロントエンドの一般的な技術およびサイボウズ提供のツールに限定させていただきます。
:::

※ kintoneのプラグイン開発の概要については次の記事をご覧ください。

https://cybozu.dev/ja/kintone/tips/development/plugins/development-plugin/development-kintone-plugin/

## 開発言語: TypeScript

**選択肢**: JavaScript、TypeScript、など

**採用**: TypeScript

フロントエンド開発と同様に、kintoneプラグイン開発においてもTypeScriptを採用するメリットは大きいと考えています。
型安全性による実装時の安心感、エディタの補完機能による開発体験の向上など、TypeScriptを採用する理由は多くあります。

サイボウズからは、TypeScriptでのプラグイン開発を支援するツールとして、[@kintone/rest-api-client](https://github.com/kintone/js-sdk/tree/main/packages/rest-api-client) と [@kintone/dts-gen](https://github.com/kintone/js-sdk/tree/main/packages/dts-gen) が提供されています。

### APIクライアント (@kintone/rest-api-client)

[kintone REST API](https://cybozu.dev/ja/kintone/docs/rest-api/)のクライアントライブラリです。

https://cybozu.dev/ja/kintone/sdk/rest-api-client/kintone-javascript-client/

メソッド呼び出しの形式でkintone REST APIを利用でき、HTTPリクエストの組み立てやレスポンスのパースなどの煩雑な処理を意識せずにAPIを利用できます。
こちらのライブラリはTypeScriptの型定義も提供しているため、リクエスト/レスポンスについて静的解析の恩恵が受けられます。

```typescript
import { KintoneRestAPIClient } from '@kintone/rest-api-client';

const client = new KintoneRestAPIClient();

// レコードの取得
const resp = await client.record.getRecords({
  app: '1',
  query: 'status = "進行中"'
});

// resp.recordsは型推論により適切に補完される
console.log(resp.records);
```

### 型定義生成ツール (@kintone/dts-gen)

kintoneアプリのフィールド定義からTypeScriptの型定義を自動生成できるツールです。

https://cybozu.dev/ja/kintone/sdk/library/dts-gen/

アプリのフィールド構造を型として扱えるため、レコード操作の処理を安全に実装できます。

また、dts-genに同梱されている `kintone.d.ts` はkintoneのJavaScript APIの型定義を提供しており、こちらもプラグイン開発においても活用できます。

:::message
2025年はJavaScript APIがかなり拡充されたのですが、これら最新のAPIへの型定義の追従はまだ行われていません。
:::

## ビルドツール/バンドラー: Rsbuild

**選択肢**: Vite、Webpack、Rollup、Rsbuildなど

**採用**: Rsbuild

現代のフロントエンド開発では、TypeScriptを使った開発はもちろん、ReactやVue.jsなどのライブラリ/フレームワークを利用することが一般的かと思います。
これらを使う場合、複数のソースファイルや依存パッケージを1つにまとめるバンドラーが必要となってきます。

最近は設定が簡単で高速に動作するViteが広く使われているかなと思います。
しかし、kintoneプラグイン開発においては、プラグイン特有の制約を考慮する必要があります。

### kintoneプラグインの制約

kintoneプラグインにはいくつか通常のWebアプリケーション開発と異なる固有の仕様上の制約があります。

まず、マニフェストファイル(`manifest.json`)で指定した**エントリーポイントのJSファイルしか読み込めません**。
そのため、画面に読み込まれるJSファイルは全てマニフェストファイル内に指定する必要があります。

また、kintoneのプラグインで読み込まれるJS/CSSなどのコンテンツは、プラグイン内のディレクトリ構造・ファイル名を保持せず、kintoneのファイル配信の仕組みに基づく独自のURLで配信されます。
そのため、[ES Modules (ESM)](https://tc39.es/ecma262/#sec-modules)に準拠した相対パスによるJSファイル間のインポートが利用できません。

そのため、**バンドラーで全てを1つのファイルにまとめる**、またはグローバルスコープに必要なオブジェクトを露出させる形でビルドする必要があります。

### ViteおよびRollupとの相性問題

Viteは内部でRollupを使用していますが、RollupはCode Splittingを完全に無効化することができません（[参考issue](https://github.com/rollup/rollup/issues/2756)）。
kintoneプラグインでは`desktop.js`、`mobile.js`、`config.js`など複数のエントリーポイントが存在するため、これら複数エントリーポイントで共通に利用されるライブラリなどは自動的にSplitされたチャンクに変換されてしまいます。
Splitされたチャンクは相対パスで動的に読み込まれることを想定しているため、kintoneプラグインの制約に合致しません。
そのため、Viteのコンフィグを複数用意して複数回Viteを実行するなどの回避策が必要になってしまいます。

### Rsbuild

上記の理由から、Code Splittingを無効化して全ての依存関係を1つのファイルにまとめる必要があります。
これができるバンドラーは各種ありますが、私はビルド速度と設定の簡潔性の観点から **Rsbuild** を採用しています。

RSbuildではchunkSplitオプションで`all-in-one`を指定することで、エントリーポイントごとに1ファイルにまとめることができます。

```js
{
  chunkSplit: { strategy: "all-in-one" }  
}
```

:::message
この制約は将来的に改善されていくと嬉しいと思っています。
現在は、JavaScript間のモジュール参照だけではなく、画像をプラグインで読み込みたい場合も、data URIなどの形式でJavaScriptに埋め込む必要があり、不便です。
また、今後プラグイン開発にHot Module Replacement (HMR)などの開発体験向上の技術が導入される際にも、現状の制約は障害となる可能性があります。
:::

## Linter、Formatter: ESLint、Prettier、Biome

**選択肢**: ESLint + Prettier、Biome

**採用**: 現時点ではESLint + Prettier (特に`@cybozu/eslint-config`を活用)

コード品質を保つために、LinterとFormatterの導入を強くおすすめします。

Linterは潜在的なバグやコーディング規約の違反を自動的に検出してくれます。特にTypeScriptと組み合わせることで、型チェックだけでは検出できない問題（未使用変数、誤ったAPI使用パターンなど）を早期に発見できます。

Formatterはコードスタイルを自動的に統一してくれるため、インデントやセミコロンの有無といったスタイルの議論から開放され、本質的な実装に集中できます。チーム開発では特に、コードレビューでスタイルの指摘に時間を費やす必要がなくなるという大きなメリットがあります。

サイボウズからは [`@cybozu/eslint-config`](https://github.com/cybozu/eslint-config) というESLint設定が公開されており、これを使うことで簡単にある程度良い設定を手軽に利用するができます。

https://cybozu.dev/ja/kintone/sdk/development-environment/eslint-config/

### Biomeについて

以下の理由から、現時点ではまだESLintとPrettierの組み合わせを採用しています。

- サイボウズから `@cybozu/eslint-config` 相当の設定プリセットがBiome用に提供されていない
- 独自Rule（プラグイン機構）がBiomeではまだサポートされていない 

後者についてはBiome v2でGritQLベースのプラグインシステムが導入されるようです。

## パッケージングツール: @kintone/plugin-packer

kintoneプラグインは、専用形式でパッケージングする必要があります。

サイボウズからは、パッケージングのためのツールとして、`@kintone/plugin-packer` が提供されています。

https://cybozu.dev/ja/kintone/sdk/development-environment/plugin-packer/

こちらはCLI版だけでなくWebアプリ版もあります。

- Webアプリ版: https://plugin-packer.kintone.dev/

Webアプリ版のほうが手軽に利用できるのですが、私はCLI版を採用しています。
CLI版のメリットは以下の3つが挙げられます。

- 手動操作が不要なため、自動化しやすく、また開発者間で操作を統一できる
- watchモードがあるため、開発効率を向上できる
- CI/CDに組み込みやすい

:::details cli-kintone の `plugin pack` コマンドについて

### cli-kintone の `plugin pack` コマンドについて

Experimentalですが、cli-kintoneの`plugin pack`コマンドでも同等のことが可能です。

https://cli.kintone.dev/guide/commands/plugin-pack/

:::

## アップローダー: @kintone/plugin-uploader

作成したプラグインはプラグイン設定画面からインストールすることができますが、
開発中は頻繁にアップロードする必要があるため、機械的にアップロードできる方が便利です。

サイボウズからは、プラグインをkintoneにアップロードするためのツールとして、`@kintone/plugin-uploader` が提供されています。

https://cybozu.dev/ja/kintone/sdk/development-environment/plugin-uploader/

こちらもplugin-packerと同様にwatchモードが提供されており、開発効率を向上できます。

:::details cli-kintone の `plugin upload` コマンドについて

### cli-kintone の `plugin upload` コマンドについて

Experimentalですが、cli-kintoneの`plugin upload`コマンドでも同等のことが可能です。
こちらは従来のplugin-uploaderと異なり、プラグインインストールAPIを使用しているため、より動作が安定しています。

https://cli.kintone.dev/guide/commands/plugin-upload/

:::

## まとめ

今回はkintoneプラグイン開発における技術選定について紹介しました。

| カテゴリ             | 採用                                                                                                              |
|------------------|-------------------------------------------------------------------------------------------------------------------|
| 言語               | [TypeScript](https://www.typescriptlang.org/)                                                                     |
| ビルドツール           | [Rsbuild](https://rsbuild.dev/)                                                                                  |
| Linter/Formatter | [ESLint](https://eslint.org/) + [Prettier](https://prettier.io/) + [@cybozu/eslint-config](https://github.com/cybozu/eslint-config) |
| パッケージングツール       | [@kintone/plugin-packer](https://cybozu.dev/ja/kintone/sdk/development-environment/plugin-packer/)                |
| アップローダー          | [@kintone/plugin-uploader](https://cybozu.dev/ja/kintone/sdk/development-environment/plugin-uploader/)            |

kintoneプラグイン開発では、一般的なフロントエンド開発の技術スタックをベースにしつつ、「1ファイルへのバンドル」というプラグイン特有の制約に対応する必要があります。
この制約により、ViteやRollupよりもRsbuildのようなCode Splittingが無効化できるバンドラーが適しています。

また、サイボウズが提供する各種ツール([@kintone/rest-api-client](https://cybozu.dev/ja/kintone/sdk/rest-api-client/kintone-javascript-client/)、[@kintone/dts-gen](https://cybozu.dev/ja/kintone/sdk/library/dts-gen/)、plugin-packer、plugin-uploader)を活用することで、型安全性を保ちながら効率的に開発を進められます。

参考までに、実際にこれらの技術スタックを使ったテンプレートを以下のリポジトリで公開しています。

https://github.com/tasshi-me/kintone-plugin-template

この記事がkintoneプラグイン開発を始める方の参考になれば幸いです！
