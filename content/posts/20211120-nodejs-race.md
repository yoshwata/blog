---
title: "Nodejs raceでスクリプトをタイムアウトさせる"
date: 2021-11-20T10:49:10Z
draft: false
---

### 概要

kubernetes上でcronjobでnodejsのスクリプトを実行させていると、どうしてもapiへのリクエストなどで時間がかかってしまうことがあります。いつまでも終了しないcronjobのpodが滞留してくるとクラスタ全体が重くなってmasterに影響がでたりすることもあります。そうならないようスクリプトは適宜タイムアウトするような作りにしておくと良いですが、その際便利だったのがPromise.raceという仕組みでした。

###　実験

まずは基本のサンプルコードです。

```
const TIMEOUT = Number(process.env.TIMEOUT) || 5000;

const timeout = async (msec) => {
    return new Promise((_, reject) => setTimeout(() => {
        reject(new Error("timeout"))
    }, msec))
}

function sleep(ms) {
    return new Promise((resolve) => {
        setTimeout(resolve, ms);
    });
}

const exec = async () => {
    try {
        await sleep(1000);
    } catch (error) {
        throw new Error(error);
    }
}

(async () => {
    return Promise.race([exec(), timeout(TIMEOUT)])
        .then(() => {
            console.log('bar');
        })
        .catch(error => {
            console.error(error);
        });
})()
```

出力結果:
```
$ time node 01.js
bar

real    0m5.151s
user    0m0.095s
sys     0m0.087s
```

出力結果にみるとおり、1秒かかる処理を終えたあと、thenの処理が実行されてbarが出力されています。しかしプロセス終了までに5秒かかっています。raceはあくまで、配列内のコールバックのどれかが終了したらthenに進むというものでtimeout関数は裏で実行されつづけています。

そのためタイムアウトを実現する場合には以下のようにexitを追加します。

```
(async () => {
    return Promise.race([exec(), timeout(TIMEOUT)])
        .then(() => {
            console.log('bar');
						process.exit(0);
        })
        .catch(error => {
            console.error(error);
						process.exit(1);
        });
})()
```

これでexec()終了後すぐにプロセスが終了します。

```
bar

real    0m1.172s
user    0m0.072s
sys     0m0.056s
```

またsleepを長くして処理がスタックした状態を再現すると、以下のようにタイムアウトされます。

```
Error: timeout
    at Timeout._onTimeout (/home/yoshwata/nodejs-race/01.js:5:16)
    at listOnTimeout (internal/timers.js:554:17)
    at processTimers (internal/timers.js:497:7)

real    0m5.170s
user    0m0.072s
sys     0m0.072s
```

稀にネットワークの不調などの問題でkubeのmasterがおかしくなっていたのですが原因はこの滞留するjob podたちでした。クリティカルなジョブでなければ通常はこのようにタイムアウトを設けておいて、スタックしたpodは潔く終了させるのがクラスタ運用目線では良さそうです。
