---
title: "Raspbianのiptables問題を解消する"
date: 2021-10-17T06:24:27Z
draft: false
tags:
  - kubernetes
  - k3s
  - network
  - raspberrypi
---

### 概要

ラズパイでk3sクラスタを構築しそこでシステムを稼働させているのですが、ある日内部の処理がかなり低下していることに気づきました。raspbianに最初から入っているiptablesに無駄なエントリが徐々に増えていくバグがあり、iptablesをアンインストールすることで解消しました。

### 環境

raspberry pi3 b+ と pi4が混在したクラスタを構成しています。

```
$ cat /etc/os-release
PRETTY_NAME="Raspbian GNU/Linux 10 (buster)"
NAME="Raspbian GNU/Linux"
VERSION_ID="10"
VERSION="10 (buster)"
VERSION_CODENAME=buster
ID=raspbian
ID_LIKE=debian
HOME_URL="http://www.raspbian.org/"
SUPPORT_URL="http://www.raspbian.org/RaspbianForums"
BUG_REPORT_URL="http://www.raspbian.org/RaspbianBugs"
```


### 詳細

kubernetesで稼働しているプロセス溜まっており、詳しく見てみると通常数百ms程度で動作していた処理が数秒かかっているようでした。

不要なログは伏せています

{{< figure src="/images/long-update.png" title="遅い処理時間" width="620" height="340" >}}

単純なAPIリクエストからレスポンスまでの時間を測っているだけですので数秒かかるのは遅すぎます。

Nodeの負荷もかなり高かったためプロセスを確認すると以下のとおりksoftirqdとiptablesが怪しそうです。

{{< figure src="/images/ksoftirqd.png" title="重いnodeのプロセス" width="720" height="140" >}}

そこでGitHubを漁っていたら[High load due to ksoftirqd, growing iptables rules](https://github.com/k3s-io/k3s/issues/3117)というissueを見つけました。

こちらのコメントの通りルールをリストしてwcかけてみるとやはり大きいようです、3MBもある

```
$ sudo iptables -L | wc
# Warning: iptables-legacy tables present, use iptables-legacy to see them
  18606  393165 3159290
```

また[こちら](https://github.com/k3s-io/k3s/issues/3117#issuecomment-808727015)で言及されているiptablesのバージョンにも近いようです。これはraspbianインストール時に最初から入っているiptablesです。

```
$ iptables --version
iptables v1.8.2 (nf_tables)
```

iptablesのエントリ増加に伴ってksoftirqedの負荷が高まるということでかなり今回の現象に近いです。ksoftirqdはカーネルスレッドのソフト割り込み負荷が高くなったときに別のコンテキストへ移す働きをするよう。iptablesはファイアーウォールでルールに違反している通信を取り締まったりする。kubernetesでネットワーク通信する際に膨大なiptalesルールのチェックが通常のソフト割り込みでは間に合わなくなったのかもしれないです。

膨大なiptablesルールをなんとかする必要がありますが、[ここ](https://github.com/k3s-io/k3s/issues/3117#issuecomment-810660207)で言われている通りiptablesのバグのようです。最初から入っているiptablesを削除してnodeを再起動して容量が膨れ上がることは無くなりました。

```
$ sudo /var/lib/rancher/k3s/data/current/bin/aux/iptables -vnL KUBE-ROUTER-INPUT | wc
# Warning: iptables-legacy tables present, use iptables-legacy to see them
     10     201    1836
```

k3sとしてはインストールされているiptablesを優先しますが、存在しない場合は起動時に別途iptablesをフェッチしてくれるようなので、プリインストールのiptablesを削除してもk3sの動作的には問題はありません。

また問題の処理性能も数日経っても高速で一定です。

{{< figure src="/images/improved-update.png" title="改善後の処理性能">}}

nodeを再起動するだけでもiptablesルールがフラッシュされるので数日は高速になるのですがすぐに重くなってきますので根本解決をしたほうがよさそうです。
