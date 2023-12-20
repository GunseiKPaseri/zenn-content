---
title: "n8nの導入"
---

# n8nとは
iPaaS(Integration Platform as a Service)，クラウド統合サービス，複数のクラウドサービスを組み合わせた自動化などを実現できるることができるノーコードツールの一つ．
Zapier，IFTTT，Make.com等様々あるが，本製品の特徴はローカルでホストすることで無制限に利用できるという点だ．
じゃあiPaaSじゃねぇかもな．

ちなみにNodemationの略でn-eight-nと読むそう．

# インストール

以下を参考にdocker-composeファイル・.env・init-data.shを作成する．

https://github.com/n8n-io/n8n/tree/master/docker/compose/withPostgres

``` yml:docker-compose.yml
version: '3.1'

services:

  postgres:
    image: postgres:11
    restart: always
    environment:
      - POSTGRES_USER
      - POSTGRES_PASSWORD
      - POSTGRES_DB
      - POSTGRES_NON_ROOT_USER
      - POSTGRES_NON_ROOT_PASSWORD
    volumes:
      - ./init-data.sh:/docker-entrypoint-initdb.d/init-data.sh:ro
      - ./db/postgres:/var/lib/postgresql/data
      - ./db/logs:/var/log
    expose:
      - '5432'
    networks:
      private:

  n8n:
    image: n8nio/n8n
    restart: always
    environment:
      - WEBHOOK_TUNNEL_URL=http://n8n.local.gunseikpaseri.cf
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_NON_ROOT_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_NON_ROOT_PASSWORD}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER
      - N8N_BASIC_AUTH_PASSWORD
    expose:
      - '5678'
    links:
      - postgres
    volumes:
      - ~/.n8n:/home/node/.n8n
    # Wait 5 seconds to start n8n to make sure that PostgreSQL is ready
    # when n8n tries to connect to it
    command: /bin/sh -c "sleep 5; n8n start"
    networks:
      private:
      public:
        aliases:
          - n8n.local.gunseikpaseri.cf
networks:
  private:
    external: false
  public:
    external: true
    name: public_networks

```

``` .env
POSTGRES_USER=changeUser
POSTGRES_PASSWORD=changePassword
POSTGRES_DB=n8n

POSTGRES_NON_ROOT_USER=changeUser
POSTGRES_NON_ROOT_PASSWORD=changePassword

N8N_BASIC_AUTH_USER=changeUser
N8N_BASIC_AUTH_PASSWORD=changePassword
```

changeUser・changePasswoordには適切な名前と複雑なパスワードを入力する．

``` sh:init-data.sh
#!/bin/bash
set -e;

if [ -n "${POSTGRES_NON_ROOT_USER:-}" ] && [ -n "${POSTGRES_NON_ROOT_PASSWORD:-}" ]; then
	psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
		CREATE USER ${POSTGRES_NON_ROOT_USER} WITH PASSWORD '${POSTGRES_NON_ROOT_PASSWORD}';
		GRANT ALL PRIVILEGES ON DATABASE ${POSTGRES_DB} TO ${POSTGRES_NON_ROOT_USER};
	EOSQL
else
	echo "SETUP INFO: No Environment variables given!"
fi
```

tmux内で起動
```
$ tmux new -s docker-n8n
$ docker-compose up
```

リバースプロキシの設定を書き換え，n8nを公開する．

``` diff nginx:reverseproxy/nginx.conf
...

http {
...
+# n8n
+   server {
+       listen 80;
+       listen 443;
+       server_name www.n8n.local.gunseikpaseri.cf;
+       return 301 https://n8n.local.gunseikpaseri.cf$request_uri;
+   }
+   server {
+       listen 80;
+       server_name n8n.local.gunseikpaseri.cf;
+       return 301 https://n8n.local.gunseikpaseri.cf$request_uri;
+   }
+   server {
+       listen 443 ssl;
+       server_name n8n.local.gunseikpaseri.cf;
+       ssl_certificate /etc/nginx/certs/ownserver-local.crt;
+       ssl_certificate_key /etc/nginx/certs/ownserver-local.key;
+       proxy_set_header Host $http_host;
+       proxy_set_header X-Forwarded-Proto $scheme;
+       client_max_body_size 10G;
+       location / {
+           proxy_pass http://n8n.local.gunseikpaseri.cf:5678/;
+           proxy_redirect off;
+       }
+   }
}
```

CorednsについてDNSのレコードを追加する．

``` diff :coredns/config/hosts
192.168.1.201 local.gunseikpaseri.cf
192.168.1.201 www.local.gunseikpaseri.cf
192.168.1.201 nextcloud.local.gunseikpaseri.cf
+192.168.1.201 n8n.local.gunseikpaseri.cf
```

n8n.local.gunseikpaseri.cfにアクセスすると，BASIC認証が開く．
ここに，`.env`の`N8N_BASIC_AUTH_USER`，`N8N_BASIC_AUTH_PASSWORD`に設定した値を入力する

Ownerアカウントの作成が求められるため，ここにも別途認証情報を設定する．
これでn8nが利用可能な状態になる．

## 問題

Twitterなんかうまく動かんのよな。これでモチベが……









