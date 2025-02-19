---
title: "付録：コマンド逆引き"
---
## 🖥️コマンドの逆引きが欲しい

欲しい…ほしくない？
コマンドなんて覚えられんもん
ただ、全部のせたらヤバいので、使用したものの中で繰り返し使用しそうなものを列挙する。

### Ubuntuの操作

#### ホスト名の取得

```sh
hostname
```

#### シャットダウン

```sh
shutdown -r now
```

#### 再起動

```sh
sudo reboot
```

### DNS・ドメイン周辺

#### ファイヤウォールの状態確認

```sh
sudo ufw status verbose
```

#### DNS正引き

```sh
dig *DOMAIN*. @*DNSServer*
```

```sh
dig *DOMAIN*. @224.0.0.251 -p 5353
```

```sh
avahi-resolve -n *DOMAIN*
```

`*DOMAIN*`：対象ドメインを指す。

#### DNS逆引き

```sh
dig -x *IP* @224.0.0.251 -p *PORT*
avahi-resolve -a *IP*
```

`*DOMAIN*`：対象ドメイン
`*PORT*`：リゾルバのポート（通常 `53`、mDNSなら `5353`）

#### ポートの利用状況確認

```sh
sudo lsof -i:*PORT*
```

`*PORT*`：確認対象ポートを指す。

### SSH関係

#### 接続

```sh
ssh {user}@{host} -p {port}
```

| オプション | 概要               |
| ---------- | ------------------ |
| `{user}` | ログイン先ユーザ名 |
| `{host}` | ログイン先ホスト名 |
| `{port}` | ポート番号         |

### docker周辺

#### 起動イメージの一覧

```sh
docker ps
```

#### イメージのログ出力

```sh
docker logs *CONTAINER ID|NAMES*
```

#### 起動中のコンテナシェルの実行

```sh
docker exec -i -t *CONTAINER ID* /bin/bash
```
