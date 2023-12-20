---
title: "付録：コマンド逆引き"
---

# コマンドの逆引きが欲しい

欲しい…ほしくない？
コマンドなんて覚えられんもん
ただ，全部のせたらヤバいので，使用したものの内，繰り返し使用しそうなものを列挙

## Ubuntuの操作

### ホスト名の取得

```
$ hostname
```

### シャットダウン

```
$ shutdown -r now
```

### 再起動
```
$ sudo reboot
```

## DNS・ドメイン周辺

### ファイヤウォールの状態確認

```
$ sudo ufw status verbose
```

### DNS正引き

```
$ dig *DOMAIN*. @*DNSServer*
or
$ dig *DOMAIN*. @224.0.0.251 -p 5353
or
$ avahi-resolve -n *DOMAIN*
```
`*DOMAIN*`：対象ドメイン
### DNS逆引き

```
$ dig -x *IP* @224.0.0.251 -p *PORT*
$ avahi-resolve -a *IP*
```
`*DOMAIN*`：対象ドメイン
`*PORT*`：リゾルバのポート（通常`53`，mDNSなら`5353`）

### ポートの利用状況確認

```
$ sudo lsof -i:*PORT*
```

`*PORT*`：確認対象ポート

## SSH関係

### 接続

```
$ ssh {user}@{host} -p {port}
```

|オプション|概要|
|----------|------------------|
|`{user}`  |ログイン先ユーザ名|
|`{host}`  |ログイン先ホスト名|
|`{port}`  |ポート番号|

## docker周辺

### 起動イメージの一覧

```
$ docker ps
```

### イメージのログ出力

```
$ docker logs *CONTAINER ID|NAMES*
```

### 起動中のコンテナシェルの実行
```
$ docker exec -i -t *CONTAINER ID* /bin/bash
```

