---
title: "Hasura GraphQLで、配列にある値を含むレコードのみでフィルタする"
date: 2021-12-22T21:22:16Z
draft: false
tags:
  - GraphQL
  - Hasura
---

### 概要

HasuraはバックエンドにPostgresQLを持つGraphQLサーバーです。PostgresQLのテーブル要素に配列があった場合にSQLであれば`@>`といった演算子でフィルタします。Hasura GraphQLにおいても同等のことができましたのでレポートします。

### フィルタ方法

例えば`mytable`の`myarray`カラム(型はjsonb)に以下のようなデータが入っていたとします。

```
id: 1, myarray: ['foo', '1', '2']
id: 2. myarray: ['3', '4']
id: 3. myarray: ['5', 'foo']
```
例えばid=1のmyarrayカラムには`['foo', '1', '2']`というjsonbが入っている状態です。

ここでmyarrayに`foo`が入っているレコードだけを引いてきたいとします。その場合のクエリは以下のとおりです。

```
query MyQuery {
  mytable(where: {myarray: {_contains: "foo"}}) {
    id
  }
}
```

これで`1`,`3`のレコードを取得できます。HasuraのUI上から見る限りですとjson型ではcontainsオペレータが存在しません。[公式ドキュメント](https://hasura.io/docs/latest/graphql/core/databases/postgres/queries/query-filters.html#jsonb-operators-contains-has-key-etc)でもJSONB Operatorと述べられている通り、jsonb限定の機能のようです。
