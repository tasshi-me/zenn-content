---
title: "Web Streams APIの背圧とTCPフロー制御"
emoji: "📡"
type: "tech"
topics: ["javascript", "nodejs", "web", "tcp", "network"]
published: false
publication_name: "cybozu_frontend"
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '26](https://summer-blog-fes.cybozu.io/2026/) (kintone Team) DAY 11の記事です。
:::

こんにちは、[tasshi](https://twitter.com/tasshi_me) です。

[前回の記事](https://zenn.dev/cybozu_frontend/articles/web-streams-api-guide)では、Web Streams APIの基本概念と、背圧（Backpressure）によってパイプチェーンの処理速度が自動調整される仕組みを紹介しました。

さて、この背圧による制御はどこまで有効なのでしょうか？
例えば`fetch()`のレスポンスをストリーム処理する場合、背圧が調整するのはJavaScriptの世界の中だけで、**ネットワークからは受信し続けて、溜まったデータがメモリを圧迫してしまう**のでしょうか。それとも**通信にもブレーキがかかる**のでしょうか。

![Responseへの背圧（STOP）は、Serverとのネットワーク通信にも伝わるのか？](/images/web-streams-api-backpressure-tcp/backpressure-question.png)
*`Response`への背圧（STOP）は、Serverとの通信制御にも伝わるのか？*

この記事では、Web Streams APIの背圧制御とネットワーク通信の制御について、パケットキャプチャによる検証とNode.jsの実装コードをもとに解説します。

:::message
検証は2024年10月のものです。文中のNode.jsのバージョンやライブラリの状況については、2026年8月時点の情報を補足しています。
:::

## 前提: 背圧（Backpressure）とは

Web Streams APIのストリームオブジェクトは、未処理のchunkを保持する内部キューを持っています。
下流の処理が遅れて内部キューが閾値（`highWaterMark`）まで溜まると、ストリームは上流にchunkの送信停止を通知します。これが背圧という仕組みです。

![背圧の仕組み：内部キューが満杯になるとSTOP信号が上流に伝播する](/images/web-streams-api-backpressure-tcp/backpressure.png)
*背圧の仕組み: 内部キューが満杯になると、上流に停止が伝播する*

背圧によってchunkの流速が自動調整されるため、パイプチェーン全体はメモリを圧迫せずにデータを処理できます。

詳しくは前回の記事をご覧ください。

https://zenn.dev/cybozu_frontend/articles/web-streams-api-guide

## 前提: TCPフロー制御

検証結果を見る前に、TCP側の流量調整の仕組みを簡単におさらいします。

TCPには**フロー制御**の仕組みがあり、受信側が処理しきれないデータを送信側が送り続けないよう調整しています。

- 受信側は、ACKパケットで**ウィンドウサイズ**（自分が今どれだけ受信できるか）を送信側に通知し続ける
- 受信側のバッファが溜まってくるとウィンドウサイズは小さくなり、送信側はそれに合わせて送信量を絞る
- 全く受信できなくなると、ウィンドウサイズ0（**Zero Window**）が通知され、データ転送は一時停止する
- 受信側のバッファに空きができると、更新されたウィンドウサイズを通知するACK（**Window Update**）が送られ、データ転送が再開する

つまり、もしWeb Streams APIの背圧制御が「受信側のバッファ消費の停滞」としてOSのTCPスタックまで届いているなら、**ウィンドウサイズの縮小やZero Windowという形でパケットに現れる**はずです。

## 検証

### 検証環境

次の構成で、背圧を意図的に発生させながらパケットをキャプチャしました。

- **Server**: JSONLを約63MB（100万行）返却するAPI（Express）
- **Batch Runner**: APIを呼び出し、レスポンスをストリーム処理する
  - 終端の`WritableStream`はchunkを受け取ると**一定時間待機**して処理遅延を発生させる（意図的に背圧を発生させる）
  - 受け取ったchunkのサイズ（バイト数）をログに出力する（1回の読み出し量を観察するため）
- 通信はHTTP/1.1、`Transfer-Encoding: chunked`
- パケットはWiresharkでキャプチャ

![検証環境の構成図。ServerとBatch RunnerがHTTPで通信し、その間のパケットをWiresharkでキャプチャする](/images/web-streams-api-backpressure-tcp/setup.png)
*検証環境: ServerとBatch Runnerの間の通信をWiresharkでキャプチャする*

終端の待機時間を変えることで背圧の発生頻度を変えながら、通信の変化（ウィンドウサイズ）と受信chunkサイズを観察します。

```typescript
// 終端のWritableStream（概略）: 待機時間を変えて背圧の強さを調整する
const slowWriter = new WritableStream<Uint8Array>({
  async write(chunk) {
    console.log("chunk size:", chunk.length);
    await setTimeout(delayMs); // 0ms / 100ms / 1000ms / 10000ms
  },
});

await response.body!.pipeTo(slowWriter);
```

検証に使ったコードは再現リポジトリとして公開しています。

https://github.com/tasshi-playground/demo-stream-api-backpressure

### 処理遅延なしの場合

まず、終端の処理遅延がない（背圧がほぼ発生しない）ケースです。

- ウィンドウサイズは約400KB（400,000バイト前後）の大きい値を維持していた
- 受信側のchunkサイズも63〜126バイトと小さい（受信速度より終端の処理速度が速い）

なお、以降に掲載するキャプチャ画像は、受信側（クライアント）が送信するACKパケットを抜き出したものです。`Win=`の値が受信側の広告ウィンドウサイズなので、この値の変化に注目してください。

![処理遅延なしの場合のパケットキャプチャ。ウィンドウサイズ（Win）が約400KBの大きい値を維持している](/images/web-streams-api-backpressure-tcp/capture-delay-0ms.png)
*処理遅延なし: ウィンドウサイズ（Win）は約400KBを維持したまま*

![処理遅延なしの場合のクライアントのログ。chunk sizeが63〜126バイトと小さい](/images/web-streams-api-backpressure-tcp/chunks-delay-0ms.png)
*処理遅延なし: 受信chunkサイズは小さい（処理が受信に追いついている）*

通信は流量制限を受けず、スムーズに流れています。

### 処理遅延100msの場合

終端のストリームで100ms待機させると、様子が変わります。

- 一定期間ごとに**ウィンドウサイズが急激に小さくなる**ことが確認できた（最小64バイトまで縮小し、Window Updateで約270KB（269,184バイト）まで回復するサイクル）
- chunkサイズは約16KB（16,506バイト）で頭打ちになっていた

![処理遅延100msの場合のパケットキャプチャ。ウィンドウサイズが65,728から64まで縮小した後、Window Updateで回復している](/images/web-streams-api-backpressure-tcp/capture-delay-100ms.png)
*処理遅延100ms: ウィンドウサイズが急激に縮小し、Window Updateで回復するサイクルを繰り返す*

![処理遅延100msの場合のクライアントのログ。chunk sizeが16,506バイトで頭打ちになっている](/images/web-streams-api-backpressure-tcp/chunks-delay-100ms.png)
*処理遅延100ms: 受信chunkサイズは約16KBで頭打ちになる*

処理が受信に追いつかなくなるたびにウィンドウサイズが絞られており、背圧の発生がウィンドウサイズに反映されていると考えられます。
chunkサイズの約16KBという値は、`Response.body`の内部受信バッファの`highWaterMark`（Node.jsの`Readable`のデフォルト値16KiB）に由来します。バッファに溜まった分を1回の読み出しでまとめて受け取っている状態です。この受信バッファは、後半のNode.jsの実装の章にも登場します。

### 処理遅延1000msの場合

さらに遅くすると、ウィンドウサイズの縮小だけでは足りなくなります。

- **Zero Window**が送信されていた（ウィンドウサイズ0、データ転送の一時停止）
- データ転送の一時停止中は、サーバーがウィンドウの回復を確認する**Zero Window Probe**を送り、クライアントが**ZeroWindowProbeAck**で応答していた
- その後**Window Update**でウィンドウサイズが更新され、データ転送が再開していた

![処理遅延1000msの場合のパケットキャプチャ。ウィンドウサイズが縮小してZero Windowに至り、ZeroWindowProbeAckのやり取りの後、Window Updateでデータ転送が再開している](/images/web-streams-api-backpressure-tcp/capture-delay-1000ms.png)
*処理遅延1000ms: ウィンドウサイズ縮小 → Zero Window → ZeroWindowProbeAck → Window Update のサイクル*

内部キューの処理にさらに時間がかかるようになった結果、データ転送の一時停止と再開が繰り返されています。

### 処理遅延10000msの場合

極端に遅くすると、Zero Window送信後、**長時間にわたってデータ転送が再開しませんでした**。
下流の内部キューがほとんど処理できておらず、受信を完全に止めて待ち続けている状態と考えられます。

![処理遅延10000msの場合のパケットキャプチャ。Zero Window送信後、ZeroWindowProbeAckだけが延々と繰り返されている](/images/web-streams-api-backpressure-tcp/capture-delay-10000ms.png)
*処理遅延10000ms: Zero Windowのまま、Zero Window Probeへの応答（ZeroWindowProbeAck）だけが続く*

## 検証結果: 背圧制御はTCPフロー制御にもウィンドウサイズの変化として伝播する

検証結果から、Web Streams APIの背圧によるデータ転送の制御は`fetch()`のネットワーク通信まで伝播していると考えられます。

1. 下流の処理が遅れて背圧が発生すると、まず**TCPのウィンドウサイズが小さくなり**、受信可能なデータ量が調整される
2. それでも処理しきれず受信を止める場合は、サーバーに**Zero Window**パケットが送信され、データ転送自体が一時停止する

![背圧の伝播のイメージ図。Responseへの背圧（STOP）が、Zero WindowとしてServerまで伝わる](/images/web-streams-api-backpressure-tcp/backpressure-zero-window.png)
*`Response`への背圧（STOP）は、Zero WindowとしてServerまで伝わる*

処理が滞った際に、JavaScriptの世界で背圧が発生するだけでなく、ネットワーク通信も制御されることが確認できました。

:::message
検証はHTTP/1.1で行ったものです。ストリームごとのフロー制御を持つHTTP/2では挙動が異なる可能性があります。
:::

## Node.jsの実装を確認する

ここまでは観察したパケットの挙動からの推察でしたが、ここからは背圧発生時のNode.jsのコードを確認してみましょう。

:::message
以下は[Node.js v26.6.0](https://github.com/nodejs/node/tree/v26.6.0)（2026年8月時点の最新リリース）のコードです。検証当時とはバージョンが異なりますが、仕組み自体は変わっていません。
:::

先に全体像を示します。背圧の発生を起点とするデータ転送の停止は4つの層を通ってカーネルまで伝播します。

```
Web Streams層（undici fetch）
  Response.bodyがpullされない → 内部バッファ満杯 → dispatcherにpause指示
    ↓
HTTP/1パーサー層（undici client-h1）
  パーサーがpaused状態になり、socket.read()を呼ぶのをやめる
    ↓
Node.js Stream層（net.Socket）
  Readableの内部バッファがhighWaterMarkに達し、handle.readStop()
    ↓
libuv層
  uv_read_stop()でソケットfdからの読み取りを停止
    ↓
カーネル（Node.jsの外）
  受信バッファが溜まり、TCPウィンドウ縮小 → Zero Window
```

順に見ていきます。

### Web Streams層: 内部キューが満杯になった場合、レスポンスデータの取得を停止する

Node.jsの`fetch()`には[undici](https://github.com/nodejs/undici)というHTTPクライアントの実装が使われています。
`fetch()`が返す`Response.body`の`ReadableStream`は、undiciの中で次のように作られています（[fetch/index.js#L2024-L2060](https://github.com/nodejs/node/blob/v26.6.0/deps/undici/src/lib/web/fetch/index.js#L2024-L2060)）。

```js
// 11. Let pullAlgorithm be an action that resumes the ongoing fetch
// if it is suspended.
const pullAlgorithm = () => {
  return fetchParams.controller.resume()
}
// ...
const stream = new ReadableStream(
  {
    start (controller) {
      fetchParams.controller.controller = controller
    },
    pull: pullAlgorithm,
    cancel: cancelAlgorithm,
    type: 'bytes'
  }
)
```

データの読み進めは`pull`、つまり**消費者が読んだときにだけ**行われます。下流の処理が遅れて`Response.body`の内部キューが`highWaterMark`まで溜まっていれば`pull`は呼ばれないので、それ以上データを要求しません。

一方、受信したデータをこのストリームに供給する側はこうなっています（[fetch/index.js#L2336-L2359](https://github.com/nodejs/node/blob/v26.6.0/deps/undici/src/lib/web/fetch/index.js#L2336-L2359)）。

```js
onResponseData (controller, chunk) {
  // ...
  if (this.body.push(bytes) === false) {
    controller.pause()
  }
},
```

受信バッファ（`this.body`）への`push()`が`false`（満杯）を返すと、`controller.pause()`でHTTPクライアントに受信の一時停止を指示します。

### HTTP/1パーサー層: socket.read()を呼ぶのをやめる

pause指示を受けたHTTP/1クライアントでは、レスポンスボディを処理する`onBody()`がHTTPパーサー（llhttp）に`ERROR.PAUSED`を返し、パーサーが`paused`状態になります（[client-h1.js#L728-L759](https://github.com/nodejs/node/blob/v26.6.0/deps/undici/src/lib/dispatcher/client-h1.js#L728-L759)）。

```js
onBody (buf) {
  // ...
  if (request.onResponseData(buf) === false) {
    return constants.ERROR.PAUSED
  }

  return 0
}
```

重要なのは、ソケットからデータを読み出すループです（[client-h1.js#L302-L310](https://github.com/nodejs/node/blob/v26.6.0/deps/undici/src/lib/dispatcher/client-h1.js#L302-L310)）。

```js
readMore () {
  while (!this.paused && this.ptr) {
    const chunk = this.socket.read()
    if (chunk === null) {
      break
    }
    this.execute(chunk)
  }
}
```

`paused`の間は、**`socket.read()`を呼ぶこと自体をやめます**。
undiciはソケットを`readable`イベント＋`read()`のpull型で読んでいるため、読み出しが止まると今度は`net.Socket`側にデータが溜まり始めます。

### Node.js Stream層: ハンドルの読み取りを止める

`socket.read()`が呼ばれなくなると、`net.Socket`（実体は`stream.Readable`）の内部バッファが`highWaterMark`に達します。
ソケットにデータが届いたときに呼ばれる`onStreamRead()`に、次の処理があります（[stream_base_commons.js#L166-L198](https://github.com/nodejs/node/blob/v26.6.0/lib/internal/stream_base_commons.js#L166-L198)）。

```js
result = stream.push(buf);
// ...
if (!result) {
  handle.reading = false;
  if (!stream.destroyed) {
    const err = handle.readStop();
```

`push()`が`false`（バッファ満杯）を返すと、`handle.readStop()`でハンドルからの読み取りを停止します。

このような内部実装までNode.js Streamの背圧の仕組みで制御されているのは非常に面白いところです。

### libuv層: uv_read_stop()

`handle.readStop()`の実体はC++の`LibuvStreamWrap::ReadStop()`で、やっていることはlibuvの`uv_read_stop()`を呼ぶだけです（[stream_wrap.cc#L217-L219](https://github.com/nodejs/node/blob/v26.6.0/src/stream_wrap.cc#L217-L219)）。

```cpp
int LibuvStreamWrap::ReadStop() {
  return uv_read_stop(stream());
}
```

これで、Node.jsはソケットからの読み取りを完全にやめます。

### カーネル: ウィンドウ縮小とZero Window

Node.jsのコードで行っているのは「ソケットを読むのをやめる」ことまででした。
そこから先の通信制御は、OSカーネルのTCPスタックが行います。
TCPスタックは受信バッファの空き容量に応じて広告ウィンドウを自動的に縮小し、空きがなくなればZero Windowを送出します。

検証で観測したウィンドウサイズの制御は、この層で行われていました。

### 再開側の経路

データ転送の再開は対称的です。

1. 消費者が`Response.body`を読む → `pull` → `controller.resume()`
2. パーサーが`paused`を解除し、`socket.read()`のループを再開（[client-h1.js#L280-L300](https://github.com/nodejs/node/blob/v26.6.0/deps/undici/src/lib/dispatcher/client-h1.js#L280-L300)）
3. `net.Socket`のバッファに空きができると`_read()` → `tryReadStart()` → `handle.readStart()`（[net.js#L926-L948](https://github.com/nodejs/node/blob/v26.6.0/lib/net.js#L926-L948)）→ `uv_read_start()`
4. カーネルの受信バッファが消費され、Window Updateが送信されてデータ転送が再開

## まとめ

- Web Streams APIの背圧は、JavaScriptのパイプチェーンの中だけでなく`fetch()`のネットワーク通信まで伝播する
  - まずTCPのウィンドウサイズが絞られ、限界を超えるとZero Windowでデータ転送が一時停止する
- Node.jsの実装から、読み取りの停止はJavaScriptのランタイムで行われて、最終的なウィンドウ制御はカーネルで行われていることが確認できた

実は、パケットキャプチャによる検証を行った2024年10月時点でも、Node.jsのコードで該当する箇所を探してみたのですが、想像していたよりも内部に入る必要があり断念していました。
この記事を書くに当たって改めてAIにコード探索してもらったのですが、一瞬でundiciのpause指示からTCPのウィンドウ制御までの経路を特定してくれて、良い時代になったのを感じました。

## 参考

- [Streams Standard - WHATWG](https://streams.spec.whatwg.org/)
- [検証の再現リポジトリ](https://github.com/tasshi-playground/demo-stream-api-backpressure)
- [fix: increased memory in finalization first appearing in v6.16.0 - nodejs/undici#3445](https://github.com/nodejs/undici/issues/3445)
- [Web Streams API 入門 ― 基本概念から実践まで](https://zenn.dev/cybozu_frontend/articles/web-streams-api-guide)

本記事のベースとなった発表スライドです。

@[speakerdeck](5ed2933d684f47db955b39b08d8e78ba)

## 余談: 検証のきっかけになったメモリリークについて

そもそもこの検証のきっかけは、業務のバッチ処理で遭遇したメモリの問題でした。`fetch()`のレスポンスをストリーム処理していたにもかかわらずメモリ使用量が急増し、「ネットワーク通信には背圧による制御が反映されず、`Response.body`の内部キューにchunkが溜まり続けているのでは？」と疑ったのが本記事の出発点です。

しかし本編で見たとおり、背圧によるデータ転送の停止は通信まで反映されていました。メモリ急増の真犯人は、当時のundiciに混入していたメモリリークでした（[nodejs/undici#3445](https://github.com/nodejs/undici/issues/3445)）。undici v6.19.7で修正されNode.js v22.7.0に取り込まれているため、現在のNode.jsを最新に保っていれば遭遇することはありません。

:::details 当時の調査と計測の詳細
Webアプリケーションから全レコードをJSONL形式のAPIで取得し、加工して自サービスのDBに書き込んでいくバッチ処理を、次のようなパイプチェーンで実装していました。

```
Response.body → TextDecoderStream → LineSplitterStream → JsonParserStream → WritableStream
（読み込み）    （UTF-8デコード）    （行ごとに再分割）    （オブジェクトに変換）  （DBに書き込み）
```

ストリームを使っているのでメモリに全レコードを載せることなく処理できるはずが、実際に動かすとメモリ使用量が急速に増加していきました。

![Node.js v18.20.4でのメモリ使用量のグラフ。RSSが約3.5GBまで急増している](/images/web-streams-api-backpressure-tcp/memory-node-v18.png)
*Node.js v18.20.4での計測。heapUsedやarrayBuffersはほぼ横ばいなのに、RSSだけが約3.5GBまで増加している*

パイプチェーンのどこに原因があるのか、ストリームを1つずつ置き換えて切り分けました。

| 置き換え対象 | 置き換え内容 | 効果 |
| ---- | ---- | ---- |
| 終端ストリーム | DB書き込み → ファイル書き込み、`setTimeout` | 効果なし |
| 中間ストリーム | 独自実装を除去 | 効果なし |
| 始端ストリーム | `fetch()` → ファイル読み込み | **効果あり** |

始端を`fetch()`からファイル読み込みに変えるとメモリ増加が収まったため、原因は`fetch()`のレスポンスよりも前にあると絞り込めました。

検証当時（2024年10月）のLTSラインへのbackport状況は、次のとおりでした。

- v20.x: v20.18.0にbackport済み（ただし手元の計測ではメモリ増加が解消せず）
- v18.x: backportの予定は見つけられず

![Node.js v20.18.0でのメモリ使用量のグラフ。RSSが約3.5GBまで増加しており、メモリ増加が解消していない](/images/web-streams-api-backpressure-tcp/memory-node-v20.png)
*Node.js v20.18.0での計測。backport済みのはずだが、手元ではRSSの増加が解消しなかった*

![Node.js v22.10.0でのメモリ使用量のグラフ。RSSは300MB前後で横ばいを維持している](/images/web-streams-api-backpressure-tcp/memory-node-v22.png)
*Node.js v22.10.0での計測。RSSは300MB前後で安定し、リークは解消している*
:::

結果的に学びになったので良いですが、まずは既存のIssueを調べることもやはり大事ですね。
