---
title: "Dockerでリバースプロキシ"
---

## ◀️Dockerでリバースプロキシの起動

DNSの項目で様々なサブドメインから自分のサーバのIPアドレスに変換し、アクセスを集中させることができるようになりました。
では、このアクセスをどうそれぞれのアプリケーションに配るか、という話になります。

今回、公開するウェブページを各コンテナ間で接続、相互に名前接続をし、名前解決・リバースプロキシを行う、という方針で行います。

### 🐋リバースプロキシ用のコンテナの作成

まず、利用するdockerネットワークを作成します。

```sh
$ docker network create --driver bridge public_networks
$ docker network ls
NETWORK ID     NAME                DRIVER    SCOPE
...
xxxxxxxxxxxx   public_networks     bridge    local
...
```

``` yml:reverseproxy/docker-compose.yml
version: '3'

services:
  reverse-proxy:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    ports:
      - 80:80
      - 443:443
    networks:
      - public

networks:
  public:
      external: true
      name: public_networks
```

``` nginx:reverseproxy/nginx.conf
events {
    worker_connections  1024;
}

http {
    # webserver
    server {
        listen 80;
        listen 443;
        server_name www.local.gunseikpaseri.cf;
        return 301 $scheme://local.gunseikpaseri.cf$request_uri;
    }
    server {
        listen 80;
        server_name local.gunseikpaseri.cf;
        location / {
            proxy_pass http://www.local.gunseikpaseri.cf:80/;
            proxy_redirect off;
        }
    }
    server {
        listen 443 default_server ssl;
        server_name local.gunseikpaseri.cf;
        ssl_certificate /etc/nginx/certs/ownserver-local.crt;
        ssl_certificate_key /etc/nginx/certs/ownserver-local.key;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Proto $scheme;
        location / {
            proxy_pass http://www.local.gunseikpaseri.cf:80/;
            proxy_redirect off;
        }
    }
}
```

いずれにもマッチしないような場合にはdefault_serverに飛びます。

`proxy_pass`のURLは`www`有りですが、これはdocker内部用の別の名前解決（WebServer側からあとで設定する）により行われています。
そのため、対外的にはwww無しに統一、内部ではwww.local.gunseikpaseri.cfに転送という若干ややこしい構成になっています。

今回練習向け・HTTP接続で証明書をDLできる（最悪）ようにするため、http->httpsのリダイレクトは設置していません。できればそれも行うことができるとよいでしょう。

### ⚙リバースプロキシ向けの設定変更

今回の変更に伴い、今までのネットワーク設定を変更していきます。

#### 🌐WebServer

``` diff yml:webserver/docker-compose.yml
version: "3"
services:
  web:
    image: nginx
+   hostname: www.local.gunseikpaseri.cf
-   ports:
-     - "80:80"
-     - "8080:443"
    volumes:
      - ./src:/usr/share/nginx/html
      - ./default.conf:/etc/nginx/conf.d/default.conf:ro
-     - ./certs:/etc/nginx/certs:ro
+   networks:
+     public:
+       aliases:
+         - www.local.gunseikpaseri.cf

+networks:
+ public:
+     external: true
+     name: public_networks
```

`networks.public.aliases`のところの`www.local.gunseikpaseri.cf`は、Docker内部DNS向けの名前解決（つまり、リバースプロキシで利用する名前）です。
DNSサーバによる名前解決ではなくこちらが実行されます。

``` diff nginx:webserver/default.conf
server {
    listen       80;
-   listen       443 ssl;
    server_name  ownserver.local;
-   ssl_certificate     /etc/nginx/certs/ownserver-local.crt;
-   ssl_certificate_key /etc/nginx/certs/ownserver-local.key;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

この状態でwebserverとreverse-proxyを`docker-compose up -d`すると、http://www.local.gunseikpaseri.cf/ https://www.local.gunseikpaseri.cf/ の両方が解決できるようになり、http://local.gunseikpaseri.cf/ または https://local.gunseikpaseri.cf/ にリダイレクトされ、そのページが表示されます。自分だけ。

#### ☁NextCloud

HTTPS関連をリバースプロキシが担当するので、ここのnginxコンテナは不要になります。

``` diff yml:nextcloud/docker-compose.yml
...
services:
- web:
-   image: nginx
-   volumes:
-     - ./nginx.conf:/etc/nginx/nginx.conf:ro
-     - ../webserver/certs:/etc/nginx/certs:ro
-   depends_on:
-     - app
-   ports:
-     - 443:443
  app:
    image: nextcloud
    restart: always
    volumes:
      - nextcloud:/var/www/html
    depends_on:
      - db
+   networks:
+     private:
+     public:
+       aliases:
+         - nextcloud.local.gunseikpaseri.cf
  db:
    image: mariadb
    restart: always
    volumes:
      - db:/var/lib/mysql
    expose:
      - '3306'
+   networks:
+     private:
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
+networks:
+ private:
+     external: false
+ public:
+     external: true
+     name: public_networks
```

`public_networks`には`app`のコンテナだけが触れればよく、`db`との通信は`private`で行うようにする。

``` nginx:reverseproxy/nginx.conf
...

http {
...
+# nextcloud
+   server {
+       listen 80;
+       listen 443;
+       server_name www.nextcloud.local.gunseikpaseri.cf;
+       return 301 https://nextcloud.local.gunseikpaseri.cf$request_uri;
+   }
+   server {
+       listen 80;
+       server_name nextcloud.local.gunseikpaseri.cf;
+       return 301 https://nextcloud.local.gunseikpaseri.cf$request_uri;
+   }
+   server {
+       listen 443 ssl;
+       server_name nextcloud.local.gunseikpaseri.cf;
+       ssl_certificate /etc/nginx/certs/ownserver-local.crt;
+       ssl_certificate_key /etc/nginx/certs/ownserver-local.key;
+       proxy_set_header Host $http_host;
+       proxy_set_header X-Forwarded-Proto $scheme;
+       client_max_body_size 10G;
+       location / {
+           proxy_pass http://nextcloud.local.gunseikpaseri.cf:80/;
+           proxy_redirect off;
+       }
+   }
}
```

`client_max_body_size`オプションは転送可能なファイルサイズ最大量になります。デフォルトでは1M、つまり1MBなのでこれより大きな画像ファイル等が転送できません。使用状況に合わせて設定してあげましょう。

この状態で、https://nextcloud.local.gunseikpaseri.cf を開くことはできますが、信頼できないドメインを介したアクセスと怒られます。
ここで言われている`config/config.php`は`/mnt/hdd/nextcloud/html/config/config.php`で操作できます。
以下のように変更します。

``` diff php:config/config.php
  'trusted_domains' =>
  array (
-   0 => 'ownserver.local',
-   1 => 'app',
+   0 => 'nextcloud.local.gunseikpaseri.cf',
  ),
...
- 'overwrite.cli.url' => 'http://app',
+ 'overwrite.cli.url' => 'https://nextcloud.local.gunseikpaseri.cf',
```

`overwrite.cli.url`は、activityページ等のURLで自動に置き換えられている箇所です。

これで正常に動作するはずです。正常に動作しなくなったときは、このページがデフォルト状態に戻ったときなので、都度直してあげればよいです。

### 必要のないポートを閉じる

必要なくなったポートは、閉じてしまう。

```sh
sudo ufw deny 8080
```

# 3⃣http3 対応

```sh
sudo ufw allow 443/udp
```

対応したHTTP3コンテナを利用
公開されたコンテナはraspi環境非対応なので手元でビルドする。

```sh
cd ~/workspace/
git clone https://github.com/macbre/docker-nginx-http3.git
cd docker-nginx-http3
docker pull ghcr.io/macbre/nginx-http3:latest
DOCKER_BUILDKIT=1 docker build . -t macbre/nginx --cache-from=ghcr.io/macbre/nginx-http3:latest --progress=plain
```

各ファイルを修正する。

``` diff yml: docker-compose.yml
  reverse-proxy:
-   image: nginx:latest
+   image: macbre/nginx:latest
...
    ports:
      - 80:80
      - 443:443
+     - 443:443/udp
```

``` diff nginx: nginx.conf
    server {
+       listen 443 http3 reuseport;
        listen 443 default_server ssl http2;
        server_name local.gunseikpaseri.cf;
        ssl_certificate /etc/nginx/certs/ownserver-local.crt;
        ssl_certificate_key /etc/nginx/certs/ownserver-local.key;
+       ssl_protocols TLSv1.2 TLSv1.3;
+       ssl_early_data on;
+       add_header alt-svc 'h3-27=":443"; ma=86400, h3-28=":443"; ma=86400, h3-29=":443"; ma=86400';
+       add_header QUIC-Status $http3;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Proto $scheme;
        location / {
            proxy_pass http://www.local.gunseikpaseri.cf:80/;
            proxy_redirect off;
        }
    }
```

各サーバについても同様に設定する。
このとき、IPアドレス:ポート番号の対比に対して`reuseport`は1つのサーバにのみにしか設定できないため、バーチャルホスト環境でのポート番号を変更する必要がある。
また、Dockerでの利用ポート・ヘッダ指定・portsを変更し、ファイヤウォールを開放する必要がある。
利用するポートはウェルノウンポートを避けるべきだろう。

再度起動します。

```sh
docker-compose up
```

初回はhttp2かも知れないが、2回目はhttp3通信をするようになっている。
