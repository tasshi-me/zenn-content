---
title: "JavaScript APIの型定義の提供方法について"
emoji: "🫥"
type: "tech"
topics: ["typescript", "api"]
publication_name: "cybozu_frontend"
published: false
---

この記事は[Cybozu Frontend Advent Calendar 2023](https://adventar.org/calendars/9255)の11日目です。

- 10日目はこちら→ [プロジェクトを理解するためのReactデザインパターン](https://zenn.dev/cybozu_frontend/articles/design-patterns-in-react)
- 12日目はこちら→ Coming soon!

こんにちは、サイボウズ株式会社の[tasshi](https://twitter.com/tasshi_me)です。  
最近は[コーヒーを控えた生活](https://sizu.me/tasshi/posts/ku4e0czkb1ab)をしています。

## 概要

この記事では自社プラットフォームなどのJavaScript APIにおける、型定義の提供方法について考えた内容をまとめたものです。

APIの型定義の提供方法に悩んでいる方の参考になれば幸いです。  
また、有識者のご意見もどしどし募集しております。

- 自社プラットフォームのJavaScript APIの型定義パッケージを公開したい
- 一般的には DefinitelyTyped (`@types`) で公開する
- オーナーシップや読み込み制御の観点で自社 organization での公開にもメリットがありそう

## 背景情報

Webサービスでは、しばしばグローバル変数などを通してJavaScriptを介した機能・データへのアクセスが提供されることがあります。  
これはMDNでは[Client-side web APIs](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Client-side_web_APIs)、その中でも[Third-party APIs](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Client-side_web_APIs/Third_party_APIs)で紹介されています。  
実際のサービス上では、REST APIと区別してJavaScript APIと呼ばれることが多い印象です。

- kintone: [kintone JavaScript API](https://cybozu.dev/ja/kintone/docs/js-api/)
- Microsoft Office: [Office JavaScript API](https://learn.microsoft.com/ja-jp/office/dev/add-ins/reference/javascript-api-for-office)
- Google Maps: [Maps JavaScript API](https://developers.google.com/maps/documentation/javascript/overview?hl=ja)
- ServiceNow: [JavaScript API](https://docs.servicenow.com/ja-JP/bundle/tokyo-application-development/page/build/applications/concept/api-javascript.html)

これらのJavaScript APIにアクセスするようなWebページやプラグイン・ユーザースクリプトをTypeScriptで開発する場合、必要となるのは型定義です。

エコシステムの開発者としては、実行時エラーを気にせずに開発を進められるように、なるべく正確な型定義が欲しいですよね。  
サービス提供側としても、エコシステムの開発者体験を良くするために、型定義をファーストパーティから提供したいと考えるでしょう。

## 提供方法

JavaScript APIの型定義をnpmパッケージとして提供する場合、主に2つの方法があります。

- DefinitelyTyped (`@types`) 配下のパッケージとして公開する
- 自前の型定義パッケージを公開する

## DefinitelyTyped (`@types`) 配下のパッケージとして公開する

DefinitelyTyped (`@types`) はJavaScriptライブラリの型定義を一元管理して公開するコミュニティベースのOrganizationです。

https://github.com/DefinitelyTyped/DefinitelyTyped

型定義がバンドルされていないライブラリについては、基本的には第三者がDefinitelyTypedに型定義を追加・公開することになります。  
作成された型定義は`@types/(ライブラリのパッケージ名)`としてnpmに公開されます。

これはTypeScriptの型定義の提供方法として一般的な方法で、TypeScriptのドキュメントにおいても型定義ファイルを直接パッケージにバンドルするか、`@types`で公開するかの2択を紹介しています。

https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html

JavaScript APIはWebページでの実行時にロードされることが多いため、直接コードに型定義をバンドルするという選択肢は取れません。  
そのため、順当に選ぶとDefinitelyTypedで型定義を提供することになります。

実際に上述の [Maps JavaScript API](https://developers.google.com/maps/documentation/javascript/overview?hl=ja) はこちらの方法を選択しています。

https://developers.google.com/maps/documentation/javascript/using-typescript?hl=ja

## DefinitelyTyped で型定義を提供することのデメリット

デファクトスタンダードのDefinitelyTypedですが、提供側の目線ではいくつかデメリットもあります。

- リポジトリが巨大であること
- オーナーシップがないこと
- 型定義が常に読み込まれてしまうこと

### リポジトリが巨大であること

DefinitelyTyped/DefinitelyTyped リポジトリは既に8000以上のライブラリの型定義を内包しています。
GitHub上でも全てのファイルは閲覧できず、またGitでの操作も時間がかかります。

メンテナンス上のデメリットについてはPrettierメンテナの [@sosukesuzuki](https://zenn.dev/sosukesuzuki) さんのブログが非常に参考になるのでそちらもご覧ください。
https://sosukesuzuki.dev/posts/prettier-type-definitions/

### オーナーシップがないこと

当たり前ですが、メンテナンスフローやリリースサイクルなどはDefinitelyTypedのルールに従うことになります。  
CI/CDについても自分たちの都合で独自のものを入れることは難しそうです。

これはチームで持っている様々な資産のうち、型定義だけが異なるルール・サイクルの下で管理されることを意味します。
チームの生産性・アジリティを低下させる要因となりそうです。

また見たところDefinitelyTypedで公開される型定義パッケージでは、npmjsのReadmeは自動生成のようです。  
今回のような元となるパッケージがない場合では、できればReadmeには詳細な利用方法を書きたいため、これもデメリットになります。

（そもそもコミュニティベースの場所でファーストパーティの人間が積極的に主導権を取るのはあまり快く思われない場合もありそうですね）

### 型定義が常に読み込まれてしまうこと

`@types/~`として公開された型定義はインストールした時点で自動的に読み込まれます。  
例えばNode.js向けのアプリケーションを作る場合に`@types/node`をインストールすることで、`process.env`などのグローバル変数に型が効くようになります。

デフォルトで読み込む型定義は、tsconfigの`typeRoots`オプションで制御できます。
https://www.typescriptlang.org/tsconfig#typeRoots

これはメリットでもある一方で、プロジェクト内でファイルごとに実行環境が異なる場合では問題になることがあります。  
具体的にはUniversal (Isomorphic) なライブラリを開発しようとしたり、1つのnpmプロジェクトでNode.js用コードベースとブラウザ用コードベースを一緒に管理している場合です。  
このような場合、特定環境向けの`@types`がコードベース全体で読み込まれると、型の安全性を損ねてしまいます。

これを解決するにはtsconfigの`types`オプションを使うことになるのですが、これはAllow List形式なので利用する型定義を全て列挙する必要があります。

https://www.typescriptlang.org/tsconfig#types

typeRootsから特定のパッケージの除外するオプションはこの記事の執筆時点では存在しませんでした。  
TypeScriptリポジトリで議論はあったようですが、採用には至らなかったようです。

https://github.com/microsoft/TypeScript/issues/18588#issuecomment-704482601

## 自前の型定義パッケージを公開する

DefinitelyTyped で公開しない場合、自社のOrganizationなどで独自のパッケージを公開することになります。

実際に試してはいないのですが、大きく2つのメリットがありそうでした。

- オーナーシップがある
- 環境・設定に応じた型定義を出し分けることができる

### オーナーシップがある

DefinitelyTypedのデメリットのそのまま逆です。  
自分たちの開発プロセス・リリースサイクルを適用できるため、積極的にメンテナンスしやすくなります。

### 環境・設定に応じた型定義を出し分けることができる

typeRootsの外で型定義を提供した場合、コードベースに型を読み込むには型定義をimportしてもらう必要があります。
これは利用者の手間が増える一方で、読み込む型定義を細かく制御したい場合には有益です。

- ファイルごとに型定義を読み込むかどうかを制御できる
- Conditional exportsで複数の型定義を提供できる

#### ファイルごとに型定義を読み込むかどうかを制御できる

型定義はimportしたファイル内でのみ有効なので、ファイルごとに型定義の読み込みを制御できます。  
これは上述したUniversalなライブラリ開発などで有用です。

例えば、`@kintone/types`という型定義ライブラリがあるとして、ブラウザ用エントリポイントでは読み込むけど、Node.js用エントリポイントでは読み込まない、というような制御が可能になります。

```typescript:@kintone/types
// kintoneの画面上で実行できるJavaScript API
export {}

declare global {
  var kintone: {
    getLoginUser(): ()=> string
  }
}
```

```typescript:index.browser.ts
import "@kintone/types"

console.log(kintone.getLoginUser()) => 型OK、実行時も正常に実行できる
```

```typescript:index.node.ts
console.log(kintone.getLoginUser()) => 型エラー、実行時も存在しない
```

#### Conditional exportsで複数の型定義を提供できる

::: message
利用側で`moduleResolution`が`Node16`もしくは`Bundler`になっている必要があります。
:::

Conditional exportsを利用することで、1つの型定義パッケージ内で複数の異なるスコープの型定義を提供することができます。
これは特定画面や特定設定が有効時のみ利用可能なAPIの型定義を提供する場合に有用です。

```typescript:index.ts
import "@kintone/types" // 全画面で有効なAPIの型定義
import "@kintone/types/garoon" // 連携サービスGaroonでのみ利用可能なAPIの定義
import "@kintone/types/special-apis" // 特定の設定が有効な場合のみ実行可能なAPI
```

::: message
これはDefinitelyTypedでも実現可能ですが、このような目的で型定義をConditional exportsしているパッケージは[なさそう](https://github.com/search?q=repo%3ADefinitelyTyped%2FDefinitelyTyped+exports+language%3AJSON&type=code&l=JSON&p=1)でした。

加えて、READMEが自動生成であることから利用方法が分かりにくいことも予想されます。
そのため、自前公開した場合のメリットとして紹介させていただきました。
:::

## 終わりに

型定義の提供方法について、色々と検討した内容を紹介させていただきました。
DefinitelyTypedはJavaScriptの資産をTypeScriptで再利用できる素晴らしい仕組みですが、そこから離れて独自に型定義を提供することにもまたメリットがありそうです。

私の所属しているkintone DXチームではkintoneエコシステムの開発者体験（DX）を向上させる活動をしています。
興味のある方は以下の記事もご覧ください。

https://blog.cybozu.io/entry/2023/06/09/113000
https://cybozu.co.jp/recruit/entry/career/lead-engineer-kintone-ecosystem.html
