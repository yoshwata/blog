---
title: "nodejsのイベントキューの動作確認"
date: 2021-11-16T10:16:06Z
draft: false
---

### 概要

Node.jsのイベントループの仕組みについて[Node.jsでのイベントループの仕組みとタイマーについて](https://blog.hiroppy.me/entry/nodejs-event-loop)という記事で学びました。しかし、こちらに掲載されている例だけではイベントキューの仕組みがピンとこなかったので自分なりに実験してみました。

### イベントキューの動作を見る

[Node.jsでのイベントループの仕組みとタイマーについて](https://blog.hiroppy.me/entry/nodejs-event-loop)では以下のコードが掲載されています。

```
setTimeout(() => console.log(1));
setImmediate(() => console.log(2));
process.nextTick(() => console.log(3));
Promise.resolve().then(() => console.log(4));
(() => console.log(5))();
```

出力は以下のようになります。

```
5
3
4
1
2
```

こうなる理由についてはhiroppyさんのブログの解説にまかせるとして、それぞれの命令がそれぞれのフェーズのキューに積まれる部分を理解したいなと思いました。

少し最初のコードを改造して以下のようにします。

```
setTimeout(() => console.log('1-1'));
setImmediate(() => console.log('1-2'));
process.nextTick(() => console.log('1-3'));
Promise.resolve().then(() => console.log('1-4'));
(() => console.log('1-5'))();

setTimeout(() => console.log('2-1'));
setImmediate(() => console.log('2-2'));
process.nextTick(() => console.log('2-3'));
Promise.resolve().then(() => console.log('2-4'));
(() => console.log('2-5'))();
```

実行結果は以下のようになります。

```
1-5
2-5
1-3
2-3
1-4
2-4
1-1
2-1
1-2
2-2
```

この動作から各命令は各フェーズのキューに積まれて積まれた準に実行されていることがわかります。操作の順番は5->3->4->1->2の順番をキープしつつ。キューの積まれた順番(1or2)は1->2をキープしています。setTimeoutについてはあくまでタイマーがスタートする順番が1->2なので引数を変えたりすると標準出力のタイミングがずれます。

もう少しわかり易い例を思いつきましたらupdateしていきます。
