---
title: "Web Streams API 入門 ― 基本概念から実践まで"
emoji: "🚰"
type: "tech"
topics: ["typescript", "javascript", "web", "nodejs", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '26](https://summer-blog-fes.cybozu.io/2026/) (Frontend Team) DAY 8の記事です。
:::

こんにちは、[tasshi](https://twitter.com/tasshi_me) です。

この記事では、JavaScriptでストリーム処理を行うための標準API「Web Streams API（Streams API）」について、基本概念から実践的な使い方、Node.js Streamとの違いまでを解説します。

本記事は、TSKaigi 2025で発表した「Web Streams APIの基本と実践、TypeScriptでの活用法」をベースに、記事として再構成したものです。

@[speakerdeck](548673024e754ace9480f202843a555b)

:::message
発表は2025年5月時点の内容ですが、本記事のブラウザ・ランタイム対応状況は2026年7月時点の情報に更新しています。
また、コードサンプルはNode.js v24 で動作確認しています。
:::

## ストリームとは何か

Web Streams APIの話に入る前に、まず「ストリーム」という概念について整理します。

:::message
様々な技術領域・開発言語でストリームの定義があるため、ここでは概ね共通と思われる性質について話します。
:::

ストリームとは、**データ入出力を逐次的に、効率よく扱うためのデータ構造**です。
データをより細かく分割された単位（チャンク）の連続した流れとして表現します。

![ストリームの概念図：chunkが変換処理を順に流れていく](/images/web-streams-api-guide/stream.png)

ストリーム自体は昔からある概念で、多くの開発言語でストリーム操作のインターフェースが提供されています。
古くはUNIXの標準ストリーム、pipe、redirectなどが代表例です。

```shell
# 標準ストリームとパイプの例
# cat の出力が sed に流れ、sed の出力が tee に流れる
cat input.txt | sed "s/foo/bar/" | tee output.txt
```

### ストリームのメリット

ストリームを使う主なメリットは2つあります。

**1. メモリ空間を圧迫しにくい**

元のデータから処理する分だけを少しずつ取り出してメモリ上に展開し、処理が終わったデータは書き込み先に書き込まれてメモリから解放されていきます。
そのため、メモリ上に同時に載っているデータ量が少なくて済む場合が多いです。

大規模データや、時間経過で無限に増大するデータ（センサーの出力など）の処理に有効です。

**2. 低遅延**

先頭のデータが処理されるまでの時間が速くなります。
データ全体の読み込みを待たずに、届いた分から処理を始められるためです。

![ストリーム処理とバッチ処理の比較](/images/web-streams-api-guide/stream-vs-batch.png)

※ただし、最終的にデータ**全体**が処理されるまでの時間が早くなるとは限りません。

## Web Streams API

[Web Streams API](https://developer.mozilla.org/ja/docs/Web/API/Streams_API)は、JavaScriptでストリーム処理を行うためのAPIです。
[WHATWGのStreams Standard](https://streams.spec.whatwg.org/)として標準化されており、ブラウザだけでなくNode.jsやDenoなどのサーバーサイドランタイムでも利用できます。

:::message
正式名称は「Streams API」ですが、本記事ではNode.jsのStreamと区別しやすいように「Web Streams API」と表記します。
:::

Web Streams APIの特徴は次のとおりです。

- データは分割された断片（**chunk**）の連続した流れとして扱われる
- 役割の異なる**3種類のストリームオブジェクト**がある
- ストリームオブジェクト同士をパイプ接続（**パイプチェーン**）することで、chunkは流れるように終端まで処理される

![Web Streams APIの3種類のストリームオブジェクト](/images/web-streams-api-guide/stream-objects.png)

### 3種類のストリームオブジェクト

| オブジェクト | 役割 | 例 |
| ---- | ---- | ---- |
| `ReadableStream` | データの読み込み | `Response.body`、`Blob.stream()` |
| `TransformStream` | データの変換 | `TextDecoderStream`、`CompressionStream` |
| `WritableStream` | データの書き込み | `FileSystemWritableFileStream` |

#### ReadableStream（読み取り可能なストリーム）

![ReadableStream](/images/web-streams-api-guide/readable-stream.png)

基となるソース（underlying source）から流れるデータを表現するオブジェクトです。
ソースから流れるデータをchunkに分割し、ストリーム処理できる形で提供します。

- 基となるソース: ファイルシステム、ネットワークリソース、メモリ上の配列、など
- 身近な例: [Fetch API](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API)の`Response.body`、`Blob.stream()`

ソースにはPull型とPush型があります。

- **Pull型**: データをストリームから明示的に読み込む（例: ファイルアクセス。[仕様の例](https://streams.spec.whatwg.org/#example-rs-pull)）
- **Push型**: データはソースから勝手に送信され、イベントリスナーなどでストリームにenqueueする（例: 動画ストリーム、TCP/WebSockets。[仕様の例](https://streams.spec.whatwg.org/#example-rs-push-backpressure)）

#### TransformStream（変換ストリーム）

![TransformStream](/images/web-streams-api-guide/transform-stream.png)

ストリームに流れるデータをある形式から別の形式に変換するオブジェクトです。
標準で提供されている実装もあります。

- `TextEncoderStream` / `TextDecoderStream`: バイナリ⇔文字列の変換
- `CompressionStream` / `DecompressionStream`: データの圧縮・展開（gzip、deflate）

#### WritableStream（書き込み可能なストリーム）

![WritableStream](/images/web-streams-api-guide/writable-stream.png)

基となるシンク（underlying sink）に流れるデータを表現するオブジェクトです。

- 基となるシンク: ファイルシステム、データベース、など
- 例: [File System API](https://developer.mozilla.org/ja/docs/Web/API/File_System_API)の`FileSystemWritableFileStream`

### パイプチェーン

ストリームオブジェクト同士をパイプ接続すると、chunkは流れるように終端まで処理されます。
パイプ接続には2つのメソッドを使います。

- `ReadableStream.pipeThrough()`: 中間の`TransformStream`に接続
- `ReadableStream.pipeTo()`: 終端の`WritableStream`に接続

![パイプチェーンの図：pipeThrough()でTransformStreamに、pipeTo()でWritableStreamに接続](/images/web-streams-api-guide/pipe-chain.png)

3種類のストリームオブジェクトとパイプチェーンを使った基本的な例を見てみましょう。
数値を生成する`ReadableStream`から、値を2倍に変換する`TransformStream`を経由して、ログ出力する`WritableStream`にデータを流します。

```typescript
// 数値を流すReadableStream
const readable = new ReadableStream<number>({
  start(controller) {
    for (let i = 1; i <= 5; i++) {
      controller.enqueue(i);
    }
    controller.close();
  },
});

// 値を2倍にするTransformStream
const double = new TransformStream<number, number>({
  transform(chunk, controller) {
    controller.enqueue(chunk * 2);
  },
});

// 受け取ったchunkをログ出力するWritableStream
const writable = new WritableStream<number>({
  write(chunk) {
    console.log(`受け取ったchunk: ${chunk}`);
  },
});

await readable.pipeThrough(double).pipeTo(writable);
console.log("完了");
```

実行結果:

```
受け取ったchunk: 2
受け取ったchunk: 4
受け取ったchunk: 6
受け取ったchunk: 8
受け取ったchunk: 10
完了
```

このように、独自のストリームオブジェクトはコンストラクタに処理用のメソッド（`pull`、`transform`、`write`など）を渡すことで作成できます。
**chunkの型はGenericsで指定できる**ため、パイプチェーン全体を型安全に記述できます。

より実践的な例として、Webリソースをfetchしてファイルに保存する処理は次のように書けます。

```typescript
import { createWriteStream } from "node:fs";
import { Writable } from "node:stream";

const response = await fetch("https://example.com");

// Node.jsのWritableをWeb Streams APIのWritableStreamに変換（後述）
const fileStream = Writable.toWeb(createWriteStream("example.html"));

// レスポンスボディ（ReadableStream）をファイルに流し込む
await response.body!.pipeTo(fileStream);
console.log("保存しました");
```

レスポンス全体をメモリに展開することなく、受信したchunkから順にファイルへ書き込まれます。

このように、**データのフローと実際にやりたい処理を分離して書ける**のがストリームの強みです。

### ストリームの分配（tee）

`ReadableStream.tee()`を使うと、1つの`ReadableStream`を2つの`ReadableStream`（branch）に分配できます。
分配したストリームはそれぞれ異なる速度で読み取ることができるため、fetchしたデータをUIとキャッシュの両方に出力する、といった使い方ができます。

```typescript
const [branch1, branch2] = readable.tee();
```

![teeによるストリームの分配](/images/web-streams-api-guide/tee.png)

:::message alert
`tee()`で分配すると、読み取りが遅い側のbranchの内部キューにchunkが滞留します。
そのため、長すぎるデータストリームには適していません。
:::

:::details tee()でchunkが滞留する理由
`tee()`で分配された2つの`ReadableStream`は、**消費速度が速い方**の速度で背圧制御されます。
そのため、時間経過と共に消費速度が遅い方のbranchの内部キューにデータが滞留してしまい、長すぎるデータストリームには適していません。

![teeの背圧問題：遅い方のbranchにチャンクが溜まってしまう](/images/web-streams-api-guide/tee-backpressure.png)

この問題に対しては、背圧制御を変更するオプションが提案されています（[whatwg/streams#1235](https://github.com/whatwg/streams/issues/1235)）。
また、Cloudflare Workersでは`tee`の背圧制御を独自に修正しています（[cloudflare/workerd#85](https://github.com/cloudflare/workerd/pull/85)）。
:::

## 利用イメージ

Web Streams APIが活用できる場面を2つ紹介します。

### 1. 他サービスからのデータインポート

メモリ負荷を軽減できる例です。
他サービスからAPI経由で全レコードを取得し、加工して自サービスのDBに書き込んでいくケースを考えます。
APIのレスポンスは[JSONL形式](https://jsonlines.org/)（1行につき1つのJSON）とします。

処理の流れは次のようなパイプチェーンになります。

![データインポートのパイプチェーン](/images/web-streams-api-guide/jsonl-import-pipeline.png)

:::details 実装例（TransformStreamを継承した独自の変換ストリームを2つ作成）

```typescript
type User = { id: number; name: string };

// 行ごとに再分割するTransformStream
class LineSplitterStream extends TransformStream<string, string> {
  constructor() {
    let buffer = "";
    super({
      transform(chunk, controller) {
        buffer += chunk;
        const lines = buffer.split("\n");
        buffer = lines.pop() ?? ""; // 最後の行は次のchunkと連結するため保持
        for (const line of lines) {
          if (line.trim() !== "") controller.enqueue(line);
        }
      },
      flush(controller) {
        if (buffer.trim() !== "") controller.enqueue(buffer);
      },
    });
  }
}

// JSON文字列をオブジェクトに変換するTransformStream
class JsonParserStream<T> extends TransformStream<string, T> {
  constructor() {
    super({
      transform(chunk, controller) {
        controller.enqueue(JSON.parse(chunk) as T);
      },
    });
  }
}

// DBへの書き込みを模したWritableStream
const dbWriterStream = new WritableStream<User>({
  async write(user) {
    // 実際にはここでDBへのINSERTなどを行う
    await new Promise((resolve) => setTimeout(resolve, 100));
    console.log(`書き込み完了: id=${user.id}, name=${user.name}`);
  },
});

const response = await fetch("https://api.example.com/users.jsonl");

await response.body!
  .pipeThrough(new TextDecoderStream()) // Uint8Array → string
  .pipeThrough(new LineSplitterStream()) // 行ごとに再分割
  .pipeThrough(new JsonParserStream<User>()) // JSON → User
  .pipeTo(dbWriterStream); // DBに書き込み

console.log("インポート完了");
```
:::

レコード数がどれだけ多くても、メモリ上に展開されるのは処理中のchunkだけです。
また、背圧（詳しくは後述）によって、DBへの書き込み速度に合わせてレスポンスの読み込み速度も自動で調整されます。

### 2. LLMの回答をリアルタイム表示

リアルタイム性を実現できる例です。
API経由でLLM（GPTなど）の回答を取得し、画面に表示するケースを考えます。

- デフォルトの一括応答では、回答**全体**が生成されてからレスポンスが返却される
- ストリーミング応答を有効化すると、回答が生成された分ずつ返却される

![ストリーミング応答と一括応答の比較](/images/web-streams-api-guide/chat.gif)

ストリーミング応答は`Response.body`（`ReadableStream`）として受け取れるため、次のように生成された分から画面に反映できます。

```typescript
const response = await fetch("https://api.example.com/chat", {
  method: "POST",
  body: JSON.stringify({ message: "こんにちは、自己紹介して？", stream: true }),
});

for await (const chunk of response.body!.pipeThrough(new TextDecoderStream())) {
  appendToScreen(chunk); // DOM要素にテキストを追記する関数（実装は省略）
}
```

## 内部キューと背圧

パイプチェーンの処理速度を調整する仕組みについて説明します。

### ストリームオブジェクト間の処理速度の違いとメモリ使用量

ストリームオブジェクトは、未処理のchunkを保持する**内部キュー**を持っています。

パイプチェーン上のストリームオブジェクト間で処理速度に差がある場合（例えば終端のDBアクセスが遅い場合）、内部キューにchunkが溜まっていくことになります。
これは処理全体のメモリ使用量の増大につながり、ストリーム処理を行うメリットを損ねてしまいます。

![内部キューにchunkが溜まっていく様子](/images/web-streams-api-guide/internal-queue.png)

### 背圧（Backpressure）

そこで登場するのが**背圧**（Backpressure）という仕組みです。

- chunkを受け入れられない場合、ストリームは上流に停止信号を出す
- 停止信号を受けた上流のストリームはデータの送信を停止する
- 下流のストリームが送信を指示（pull）すると再び処理が再開する

![背圧の仕組み：内部キューが満杯になるとSTOP信号が上流に伝播する](/images/web-streams-api-guide/backpressure.png)

これにより、chunkの流速が自動調整されて、メモリを圧迫せずに処理できます。

### キューイング戦略と最高水準点（highWaterMark）

背圧は内部キューの状態から、**キューイング戦略**に基づいて通知されます。
現在は2種類のキューイング戦略が利用可能です。

| キューイング戦略 | 判定基準 |
| ---- | ---- |
| `CountQueuingStrategy` | キューに格納されたchunk数で判定 |
| `ByteLengthQueuingStrategy` | キューに格納されたバイト数で判定 |

![CountQueuingStrategyとByteLengthQueuingStrategyの比較](/images/web-streams-api-guide/highWaterMark.png)

キューイング戦略はストリームオブジェクトのコンストラクタで指定し、閾値は最高水準点（`highWaterMark`）として設定します。
どちらの戦略を使うかは、流すデータの種類に応じて判断します。

背圧の動作を確認できる例を示します。  
書き込みが遅い`WritableStream`に合わせて、上流のchunk生成が待たされる様子が観察できます。

```typescript
const readable = new ReadableStream<number>(
  {
    pull(controller) {
      const value = counter++;
      console.log(`生成: ${value}`);
      controller.enqueue(value);
      if (value >= 10) controller.close();
    },
  },
  new CountQueuingStrategy({ highWaterMark: 2 }), // 内部キューの最高水準点を2に設定
);
let counter = 1;

const slowWriter = new WritableStream<number>(
  {
    async write(chunk) {
      await new Promise((resolve) => setTimeout(resolve, 300)); // 遅い書き込み処理
      console.log(`書き込み: ${chunk}`);
    },
  },
  new CountQueuingStrategy({ highWaterMark: 1 }),
);

await readable.pipeTo(slowWriter);
```

実行結果:

```
生成: 1
生成: 2
生成: 3
書き込み: 1
生成: 4
書き込み: 2
生成: 5
書き込み: 3
...
```

最初に内部キューが埋まるまでchunkが生成された後は、書き込みが1つ完了するたびに次のchunkが1つ生成されるようになっています。
コード上では一切待つ処理は書いていませんが、流速が自動調整されているのがわかります。

:::details ネットワーク通信に背圧は反映されているのか？
`fetch()`のレスポンスをストリーム処理する場合、背圧はunderlying sourceへのネットワークリクエストにも反映されます。

HTTP/1.1（`Transfer-Encoding: chunked`）で検証したところ、次の挙動が確認できました。

- 内部キューが溜まってくると、TCPのウィンドウサイズが小さくなり、受信可能データを調整する
- それでも処理しきれずに受信を止める場合は、zero windowパケットが送信される

詳細は次のスライドにまとめています。

https://speakerdeck.com/tasshi/web-streams-api-and-tcp-flow-control
:::

## Promiseベースの非同期処理との相互運用性

Web Streams APIはPromiseベースの非同期処理と組み合わせやすく設計されています。

### ストリームをasync/awaitの中で使う

`pipeTo()`の返り値はPromiseです。
パイプチェーンでのデータ処理が終わるまで`await`で待つことができます。

```typescript
await readable.pipeThrough(transform).pipeTo(writable);
// ここに到達した時点で全chunkの処理が完了している
```

chunkを1つずつ操作したい場合は、reader/writerを取得します。

- `ReadableStream.getReader()` / `WritableStream.getWriter()`でreader/writerを取得
- `await reader.read()` / `await writer.write()`でchunkを1つずつ読み込み/書き込みできる

```typescript
const reader = readable.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log(value);
}
```

### ストリームを反復処理する

`ReadableStream`は非同期反復可能（`[Symbol.asyncIterator]()`を実装している）なので、`for await ... of`で反復処理できます。

```typescript
for await (const chunk of readable) {
  console.log(chunk);
}
```

また、`Array.fromAsync()`でデータを全て読み出して配列に格納することもできます。

```typescript
const chunks = await Array.fromAsync(readable);
```

:::message
`ReadableStream`の非同期反復は、Chrome/Edge 124以降、Firefox 110以降、Node.js、Denoで利用できます。
Safariは長らく非対応でしたが、Safari 27（2026年6月にベータ公開）で対応予定です。

`Array.fromAsync()`自体は全ての主要ブラウザとNode.js v22以降で利用できます。
:::

### イテラブルからストリームを作成する

[`ReadableStream.from()`](https://developer.mozilla.org/ja/docs/Web/API/ReadableStream/from_static)を使うと、反復可能オブジェクト（または非同期反復可能オブジェクト）から`ReadableStream`を作成できます。

```typescript
async function* generate() {
  yield "foo";
  yield "bar";
}

const readable = ReadableStream.from(generate());
```

:::message
`ReadableStream.from()`は、Firefox 117以降、Node.js v20.6以降、Denoで利用できます。
Chrome/Edgeは執筆時点（2026年7月）で未対応、SafariはSafari 27で対応予定です。
:::

## Node.js Streamとの違いと互換性

サーバーサイドJSでストリームといえば、[Node.jsのStream](https://nodejs.org/api/stream.html)（以下、Node Stream）を思い浮かべる方も多いと思います。

### Node Streamについて

Node StreamはNode.jsの組み込みモジュールで、Web Streams APIよりも先発です。  
かつては「Streamを制するものはNode.jsを制す」と言われていたとか。

Web Streams APIとはいくつか異なる点があります。

- `EventEmitter`を継承していて、イベント駆動的なインターフェース
- 3(+2)種類のストリームオブジェクト

| Node Stream | 対応するWeb Streams API |
| ---- | ---- |
| `Readable` | `ReadableStream` |
| `Writable` | `WritableStream` |
| `Duplex`（双方向） | ─ |
| `Transform`（変換） | `TransformStream` |
| `PassThrough`（パススルー） | ─ |

### Node.jsにおけるWeb Streams APIのサポート

Node.jsでも**v21でWeb Streams APIがStable**になりました。

さらに、Node StreamとWeb Streams APIは`toWeb()` / `fromWeb()`メソッドで相互に変換可能です。
これらの変換メソッドはv17で追加されてから長らくExperimentalでしたが、**v24でとうとうStable**になりました（v22.17.0にもバックポートされています）。

```typescript
import { createReadStream, createWriteStream } from "node:fs";
import { Readable, Writable } from "node:stream";

// Node Stream → Web Streams API
const webReadable = Readable.toWeb(createReadStream("example.html"));

// 書き込み側も変換できる
const webWritable = Writable.toWeb(createWriteStream("copy.html"));

await webReadable.pipeTo(webWritable);
console.log("コピー完了");
```

### Node Streamと比べて良くなった点

以下は私がWeb Streams APIがNode Streamと比べて良くなったと思う点です。

**1. Web標準であること**

- クロスブラウザに利用できる
- Fetch APIを始めとして、他の標準仕様にもWeb Streams APIベースのAPIが増えていっているため、使っていくことによるレバレッジが効きやすい
- [WinterTC Minimum Common API](https://min-common-api.proposal.wintertc.org/)にも含まれており、Node.jsに限らず各種サーバーサイドJSランタイムで利用できる

**2. インターフェースの改善**

- `EventEmitter`/Callbackな書き方からPromise/async/awaitな書き方に
- 利用者に公開されているメソッド・プロパティがかなり減っているため、学習コストが減った
  - （逆に細かい制御がしづらくなったとも言えます）

**3. 型情報**

- Node Stream（`@types/node`）: chunkの型が`any`
- Web Streams API: chunkの型を**Genericsで指定できる**

```typescript
// chunkの型を指定でき、パイプチェーンの型整合性がチェックされる
const stream = new TransformStream<string, number>({
  transform(chunk, controller) {
    // chunkはstring型、enqueueできるのはnumber型のみ
    controller.enqueue(chunk.length);
  },
});
```

**4. エラーハンドリング**

- Node Streamでは、上流のストリームがエラー終了しても下流のストリームが閉じない
  - `error`イベントのイベントリスナーで明示的に閉じる必要がある
- Web Streams APIでは、勝手に閉じてくれる
  - `pipeThrough()` / `pipeTo()`の`preventAbort`オプションで制御可能

**5. 変換ストリームの結合がやりやすい**

- Node Stream: `Duplex`でラップするが、イベントリスナーやメソッドの繋ぎ込みがかなり面倒
  - `stream.compose()`を利用すると簡単に結合できる（v26.2.0でStableになりました）
- Web Streams API: `TransformStream`でラップしてパイプ接続したらOK

### どっちを使えばいい？

私は、**これから書くコードは最初からWeb Streams APIで良い**と考えています。

- Web標準かつPromiseベースの書き方ができる
- Node Streamとも`toWeb()` / `fromWeb()`で相互変換できる

一方、昔からあるnpmパッケージはNode Streamを使っているため、当面は`toWeb()` / `fromWeb()`メソッドで変換しながら併用することになりそうです。

## まとめ

- Streams APIには3つのストリームオブジェクトがある
  - `ReadableStream`、`TransformStream`、`WritableStream`
- ストリームオブジェクトをパイプ接続してデータを逐次処理できる
- 背圧によって処理速度が自動調整され、メモリ使用量を適切に抑えることができる
- Node Streamとは相互に変換可能
- **新規に書くコードはWeb Streams APIで良い**
  - async/awaitな処理とも組み合わせやすい
  - chunkの型をGenericsで指定でき、TypeScriptとの相性も良い

この記事がStreams APIを知るきっかけになれば嬉しいです。

また、私たちは一緒にkintoneを改善してくれる仲間を探しています。
興味のある方はぜひ採用情報をご確認ください！

- [kintone開発エンジニア](https://cybozu.co.jp/recruit/entry/career/product-engineer-kintone.html)
- [エンジニアリングマネージャー](https://cybozu.co.jp/recruit/entry/career/engineering-manager.html)

## 参考

- [Streams API - MDN](https://developer.mozilla.org/ja/docs/Web/API/Streams_API)
- [Streams Standard - WHATWG](https://streams.spec.whatwg.org/)
- [Stream - Node.js Documentation](https://nodejs.org/api/stream.html)
- [Web Streams API - Node.js Documentation](https://nodejs.org/api/webstreams.html)
- [Streams—The definitive guide - web.dev](https://web.dev/articles/streams)
