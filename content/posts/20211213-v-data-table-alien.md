---
title: "Vuetifyのv-data-tableのヘッダがズレてしまう"
date: 2021-12-13T10:51:28Z
draft: false
tags:
  - Nuxt.js
  - Vue.js
  - Vuetify
---

### 概要

Vuetifyで[v-data-table](https://vuetifyjs.com/ja/components/data-tables/)を使っていてカラムがずれる症状がでました。悩みましたが解決したのでレポートします。

### 問題

以下のような`v-data-table`を作成してみます。

```xml
<template>
  <v-data-table
    :headers="headers"
    :items="cards"
    :items-per-page="5"
    class="elevation-1"
    :mobile-breakpoint="0"
  ></v-data-table>
</template>
```

すると以下の画像のようになります。

{{< figure src="/images/20211213-v-data-table-alien/zure.png" width="700" height="70" >}}

ヘッダと値の位置が大きくずれていて非常に見づらい表になっていまっています。

### 解決

以下のように`v-data-table`の前に`v-app`をはさみます。

```xml
<template>
  <v-app>
    <v-data-table
      :headers="headers"
      :items="cards"
      :items-per-page="5"
      class="elevation-1"
      :mobile-breakpoint="0"
    ></v-data-table>
  </v-app>
</template>
```

すると以下のようになります。

{{< figure src="/images/20211213-v-data-table-alien/nozure.png" width="700" height="70" >}}

きれいに列が揃っています。詳しく理由はわかりませんが修正方法は以上です。UI系って「なんだかよくわからないがこうすれば動く」みたいなもの多いですね。理由を知っている方おりましたらDMでもなんでもいただけましたらと思います。
