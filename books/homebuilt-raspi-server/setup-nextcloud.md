---
title: "NextCloudの導入"
---

## ☁NextCloudとは

オンプレミス型ウェブストレージ、とでもいうべきでしょうか。ようはDropbox・GDrive等のオンプレ版です。
類似サービスにはフォーク元の**OwnCloud**がありますが、NextCloudの方が積極的に開発されています。

## ⏬NextCloudの導入

このへんとか？

https://denor.jp/docker-for-windows%E3%81%A7nextcloud%E3%82%B5%E3%83%BC%E3%83%90%E6%A7%8B%E7%AF%89

```sh
mkdir nextcloud
vim ./nextcloud/docker-compose.yml
vim ./nextcloud/.env
```

``` yml:docker-compose.yml
version: '3.7'
volumes:
  nextcloud:
    driver_opts:
      type: none
      device: /mnt/hdd/nextcloud/html
      o: bind
  db:
    driver_opts:
      type: none
      device: /mnt/hdd/nextcloud/mysql
      o: bind
services:
  web:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ../webserver/certs:/etc/nginx/certs:ro
    depends_on:
      - app
    ports:
      - 443:443
  app:
    image: nextcloud
    hostname: *domain*
    restart: always
    volumes:
      - nextcloud:/var/www/html
    depends_on:
      - db
  db:
    image: mariadb
    restart: always
    volumes:
      - db:/var/lib/mysql
    expose:
      - '3306'
    command:
      - "--character-set-server=utf8mb4"
      - "--collation-server=utf8mb4_unicode_ci"
      - "--innodb_strict_mode=OFF"
      - "--innodb_read_only_compressed=OFF"
    environment:
      - MYSQL_ROOT_PASSWORD=${ENV_MYSQL_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${ENV_MYSQL_PASSWORD}
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
```

appの`hostname`は、アプリでも利用されるので、自身のアドレスを入力してあげる必要があります。

また、`~/webserver/docker-compose.yml`のポートも修正しておきます。

``` diff yml:docker-compose.yml
version: "3"
services:
 web:
    image: nginx
    ports:
      - "80:80"
-      - "443:443"
+      - "8080:443"
    volumes:
      - ./src:/usr/share/nginx/html
      - ./default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
```

```nginx:nginx.conf
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile       on;
    tcp_nopush     on;
    tcp_nodelay         on;
    types_hash_max_size 2048;
    client_max_body_size 8g;

    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;

        server_name _;
        ssl_certificate     /etc/nginx/certs/ownserver-local.crt;
        ssl_certificate_key /etc/nginx/certs/ownserver-local.key;

        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_session_tickets on;

        proxy_set_header    Host               $host;
        proxy_set_header    X-Real-IP          $remote_addr;
        proxy_set_header    X-Forwarded-Host   $host;
        proxy_set_header    X-Forwarded-Server $host;
        proxy_set_header    X-Forwarded-For    $proxy_add_x_forwarded_for;

        location / {
            proxy_pass http://app/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_read_timeout 300;
            proxy_connect_timeout 300;
        }
    }

    include /etc/nginx/conf.d/*.conf;

    map $http_upgrade $connection_upgrade {
        default Upgrade;
        ''      close;
    }
}
```

``` :.env
ENV_MYSQL_ROOT_PASSWORD=安全なパス
ENV_MYSQL_PASSWORD=安全なパス
```

コンテナ名で名前解決できるんですねぇ……

ディレクトリを作っておかないと止まります。ファイヤウォールの`8080`ポートも空けておきます。

```sh
mkdir /mnt/hdd/nextcloud
mkdir /mnt/hdd/nextcloud/html
mkdir /mnt/hdd/nextcloud/mysql
sudo ufw allow 8080
```

いざ実行します。

```sh
docker-compose up -d
```

以下のようなエラーが出た場合は`$ sudo apt install linux-modules-extra-raspi`しましょう。

```txt
Error response from daemon: failed to create endpoint nextcloud-db-1 on network nextcloud_default: failed to add the host (veth26aca14) <=> sandbox (veth848aad0) pair interfaces: operation not supported
```

うまくいかなければ、`docker-compose logs`でログを見たり、`docker ps`でコンテナIDを取得して、`docker exec -i -t *CONTAINER_ID* /bin/bash`でシェルを操作したりできます。


起動できると、`https:*domain*:8080`でNextCloudの設定画面になります。

データフォルダは、すでにdocker-composeでマウントしているのでそのまま。
MySQL/MariaDBを選択する。

| 項目名                | 設定値                   |
| --------------------- | ------------------------ |
|データベースのユーザ名 | `nextcloud`              |
|パスワード             | `ENV_MYSQL_PASSWORD`の値 |
|データベース名         | `nextcloud`              |
|データベースのホスト名 | `db:3306`                |

名前解決が`docker-compose.yml`で決めた名前になるのはここでも同じ。

1分～2分くらいロード時間が入りますので辛抱強く待ちます。俺は90秒くらいだった。
そこから推奨アプリのインストール画面になる。

### ⚙host名設定ミスによる再設定

一部ドメインが`app`になってしまった。`docker-compose.yml`で指定したサービス名のせいだろうが、その関係で思うようにリンク先に飛ばないことが多い。（例：ログインのリダイレクト通知ページのロゴ、アクティビティの画像）。
ここの項目はリバースプロキシの設定の際改めて設定するが、
`/mnt/hdd/nextcloud/html/config/config.php`の`'overwrite.cli.url' => 'http://app'`のところを変更すればよい。

## ⚙NextCloudの設定

### 🔒ファイル暗号化

アプリを開く
`Default encryption module`を有効にする。

設定を開き、管理＞セキュリティタブを開く。
再ログインが求められている。
「サーバサイド暗号化を有効にする」をチェックする。

### 👤ユーザの追加

ユーザタブで新しいユーザを追加すれば良し。

## 🆙NextCloudのアップデート

NextCloudのバージョンアップはメジャーアップデートごとに行う必要がある。
今回は22.2.0から24.0.2に上げる。
自分のバージョンは以下のようなコマンドで取得できる（ユーザIDは自分の環境では33だったが指示に合わせ変更する）

```sh
docker exec -it -u 33 *CONTAINER ID* /var/www/html/occ --version
```

### 💾バックアップ

まずNextCloudで作成されたファイルについてバックアップを行い、操作ミスが発生して破壊してしまっても復元できるようにする。

対象と名ᵣうディレクトリサイズを確認しておく。単位はバイト。
また、数十GB単位のアクセスなので、数秒ほど時間が取られます。

```sh
$ sudo time -p du -bs /mnt/hdd/nextcloud/
32979507236     /mnt/hdd/nextcloud/
real 5.13
user 0.65
sys 4.43
```

このディレクトリに対してファイル圧縮を掛けて保存する。様々な圧縮形式があるが、処理時間と圧縮率はトレードオフです。
ファイルサイズが大きいので圧縮率で選び.tar.xz形式を利用する。
Ubuntuにはもともと入っている。バージョン1.22以降のtarでは.tarを経由せずに圧縮できる。

```sh
$ tar --version
tar (GNU tar) 1.34
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by John Gilmore and Jay Fenlason.
```

backupディレクトリを作成しtarコマンドで圧縮する。
非常に時間がかかるので、内部のファイルを操作できるなら、巨大なファイルは予め除いておくとよいかもしれない。

```sh
$ mkdir -p /mnt/hdd/backup/nextcloud
$ cd /mnt/hdd/backup/nextcloud
$ sudo time -p tar Jcvf backup$(date +%Y%m%d%H%M).tar.xz /mnt/hdd/nextcloud
...
real 36272.12
user 35364.00
sys 829.91
$ ls -la
-rw-r--r-- 1 root          root          24987623420 Jul  1 10:57 backup202207010053.tar.xz
```

圧縮率は75%、保存されている多くが圧縮ファイルだったので仕方ない。処理時間は約半日。余裕があるときにやった方がよいだろう。

解凍時tarをsuの権限で実行すれば、所有者情報も含めて復元される。

### 🆙バージョンアップ

#### 22 →23

1. nextcloudのdockerを停止する
2. バージョンを変更する

``` diff yml:~/nextcloud/docker-compose.yml
...
  app:
-    image: nextcloud
+    image: nextcloud:23.0.6
...
```

3. 再実行

```sh
dokcer-compose up
```

4. バージョンが上がっていることを確認する

#### 23→24

同様に行う。

```sh
docker pull nextcloud:24.0.2
```

#### 📎不要なイメージの削除

NextCloudは1つで800MB以上使用するので、不要なイメージは削除しておく
今利用しているバージョンのTAG以外(latestを使わないならばそれも)削除してしまう。

```sh
docker image ls | grep nextcloud
docker rmi *CONTAINER IMAGE*
```
