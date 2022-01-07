---
title: "NextCloudの導入"
---
# NextCloudとは
オンプレミス型ウェブストレージ，とでもいうべきでしょうか．ようはDropbox・GDrive等のオンプレ版です．
類似サービスにはフォーク元の**OwnCloud**がありますが，こちらの方が開発が積極的です．

# NextCloudの導入
このへんとか？

https://denor.jp/docker-for-windows%E3%81%A7nextcloud%E3%82%B5%E3%83%BC%E3%83%90%E6%A7%8B%E7%AF%89

```
$ mkdir nextcloud
$ vim ./nextcloud/docker-compose.yml
$ vim ./nextcloud/.env
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
appの`hostname`は，アプリでも利用されるので，自身のアドレスを入力して上げる必要があります．

また，`~/webserver/docker-compose.yml`のポートも修正しておきます．
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

ディレクトリを作っておかないと止まります．ファイヤウォールの`8080`ポートも空けておきます．
```
$ mkdir /mnt/hdd/nextcloud
$ mkdir /mnt/hdd/nextcloud/html
$ mkdir /mnt/hdd/nextcloud/mysql
$ sudo ufw allow 8080
```
いざ実行します．
```
$ docker-compose up -d
```

以下のようなエラーが出た場合は`$ sudo apt install linux-modules-extra-raspi`しましょう．
```
Error response from daemon: failed to create endpoint nextcloud-db-1 on network nextcloud_default: failed to add the host (veth26aca14) <=> sandbox (veth848aad0) pair interfaces: operation not supported
```

なんか上手く行かなければ，`docker-compose logs`でログを見たり，`docker ps`でコンテナIDを取得して、`docker exec -i -t *CONTAINER_ID* /bin/bash`でシェルを操作したりできます．


起動できると，`https:*domain*:8080`でNextCloudの設定画面になります．

データフォルダは，既にdocker-composeでマウントしているのでそのまま．
MySQL/MariaDBを選択
| 項目名                | 設定値                   |
| --------------------- | ------------------------ |
|データベースのユーザ名 | `nextcloud`              |
|パスワード             | `ENV_MYSQL_PASSWORD`の値 |
|データベース名         | `nextcloud`              |
|データベースのホスト名 | `db:3306`                |

名前解決が`docker-compose.yml`で決めた名前になるのはここでも同じ．


1分～2分くらいロード時間が入りますので辛抱強く待ちます．俺は90秒くらいだった．
そこから推奨アプリのインストール画面になる．


### host名設定ミスによる再設定
一部ドメインが`app`になってしまった．`docker-compose.yml`で指定したサービス名のせいだろうが，その関係で思うようにリンクが飛ばないことが多い．（例：ログインのリダイレクト通知ページのロゴ、アクティビティの画像）．
ここの項目はリバースプロキシの設定の際改めて設定するが，
`/mnt/hdd/nextcloud/html/config/config.php`の`'overwrite.cli.url' => 'http://app'`のところを変更すれば良い．


# NextCloudの設定
## ファイル暗号化
アプリを開く
`Default encryption module`を有効にする．

設定を開き、管理＞セキュリティタブを開く．
再ログインが求められている．
「サーバサイド暗号化を有効にする」をチェック

## ユーザの追加
ユーザタブで新しいユーザを追加すれば良し

