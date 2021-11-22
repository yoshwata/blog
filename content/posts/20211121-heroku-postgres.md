---
title: "Heroku Postgres Tips"
date: 2021-11-21T11:30:30Z
draft: false
tags:
  - heroku
  - postgresql
---

### 概要

heroku環境でpostgresqlを運用しているのですがハマりどころが多く、運用の度に思い出しながら対応していました。そろそろなんとかしなければという思いで、準備から頻繁に利用する運用コマンドをまとめていきます。

### 環境

```
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.5 LTS
Release:        18.04
Codename:       bionic
```

### herokuのPostgresサーバーへ接続する

通常以下のコマンドでherokuのPostgresサーバーに接続できます。

```
$ heroku pg:psql
```

しかし、以下のようなエラーになることがあります。

```
EACCES: spawn psql EACCES
```

heroku cliのラッパーからPostgresへ接続する際にはローカルのpsqlコマンドを利用しているため、psqlがインストールされていない場合上記のようなエラーになることがあるようです。非常にわかりにくい。

ではローカルのUbuntu環境へpsqlをインストールしていきます。今回wgetとlsb_releaseを利用しますのでインストールしてください。

```
sudo apt-get install lsb-release wget
```

次に以下の通りパッケージリストへrepoを追加します。

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
```

あとはアップデートして

```
sudo apt update
```

インストールします。heroku環境に接続するだけならクライアントだけで十分です。インストールするクライアントのバージョンをサーバー環境とあわせないと実行できないコマンドなどもありますので`heroku pg:info`でバージョンを確認します。私の環境では13でした。

```
sudo apt install postgresql-client-13
```

これでPostgresサーバーへ接続できるようになります。

```
heroku pg:psql
```

### jsonにアクセスする

Postgresqlのカラムの型にはjsonを指定することができます。データの塊を一箇所に置いておくことができるので便利ですが、それなりに大きなjsonになると個別の値を確認するのが困難になってきます。

ただ、この場合でもjsonの各キーにアクセスできるクエリを書くことができます。（バージョン13であれば少なくとも可能でした）

```sql
select mycollumn->>'myKey' from mytable limit 10;
```

Postgresのお約束としてキャメルケースのキーやカラムをクエリに使う場合はクォートで囲う必要がある点に注意してください。
