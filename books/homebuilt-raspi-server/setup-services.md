---
title: "様々なサービスを導入する"
---

## 🔧様々なサービスを導入する

基本的にはnextcloudと同じ要領でいろいろ動かしていけばよいです。
自分は以下のようなサービスを立ててみました。他にもいろいろあるのかなと思います。

### ⚙VaultWarden

https://github.com/dani-garcia/vaultwarden

BitWarden互換のパスワード管理サービスです。
クラウドにパスワードを保存するのは少し怖い、でもkeepassは使い勝手が悪いしフィッシングサイトにパスワードを間違ってコピペするのが怖い。
そんな人はVaultWardenをセルフホストしてみましょう。

手元ではBitWardenの拡張機能を入れ、ログイン時にセルフホストした自分のサービスを指すようにしてあげるとよいです。

:::details composeファイル

```yaml:compose.yaml
volumes:
  vaultwarden:
    driver_opts:
      type: none
      device: /mnt/hdd/vaultwarden
      o: bind
services:
  vaultwarden:
    image: ghcr.io/dani-garcia/vaultwarden:1.32.7-alpine
    hostname: vaultwarden
    container_name: vaultwarden
    environment:
      - TZ=Azia/Tokyo
      - SIGNUPS_ALLOWED=false
      - DOMAIN=https://vw.local.gunseikpaseri.cf
      - VIRTUAL_HOST=vw.local.gunseikpaseri.cf
      - VIRTUAL_PORT=80
    volumes:
      - vaultwarden:/data
    networks:
      private:
      public:
    restart: always
networks:
  private:
    external: false
  public:
    external: true
    name: tailscale-external
```

:::

### 🪼Jellyfin

https://jellyfin.org/

セルフホストできる音楽プレイヤー。
サブスク配信していない楽曲を聴くのに使っています。
でも内部動作見ると、最盛時に音声全部ダウンロードしているんですよね。ストリーミングしてほしいな。

:::details composeファイル

```yaml:compose.yaml
volumes:
  cache:
    driver_opts:
      type: none
      device: /mnt/hdd/jellyfin/cache
      o: bind
  config:
    driver_opts:
      type: none
      device: /mnt/hdd/jellyfin/config
      o: bind
services:
  jellyfin:
    image: jellyfin/jellyfin:10.10.0
    user: 1001:1001
    hostname: jellyfin
    restart: 'unless-stopped'
    environment:
      - JELLYFIN_PublishedServerUrl=https://jf.local.gunseikpaseri.cf
    networks:
      public:
    volumes:
      - cache:/cache
      - config:/config
      - /mnt/hdd/smb/ownserver:/media

networks:
  #  private:
  #  external: false
  public:
    external: true
    name: tailscale-external

```

:::

### 🍵Gitea

https://about.gitea.com/

GitHubはブランチ単位で非公開にできません。
プライベートブランチの代わりに自分用のGitサーバを用意してそっちにプッシュすることで、バックアップを残すことができます。

軽量という評価がされているGiteaを採用しました。

:::details composeファイル

```yaml:compose.yaml
volumes:
  data:
    driver_opts:
      type: none
      device: /mnt/hdd/gitea/data
      o: bind
  db:
    driver_opts:
      type: none
      device: /mnt/hdd/gitea/db
      o: bind
services:
  gitea:
    image: gitea/gitea:1.23.0
    container_name: gitea
    hostname: gitea
    restart: always
    networks:
      - public
      - private
    volumes:
      - data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:2222"
    environment:
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: db:5432
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: ${DB_PASSWORD}
      SSH_PORT: 2222
    depends_on:
      - db
  db:
    image: postgres:16.3-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: gitea
    networks:
      - private
    volumes:
      - db:/var/lib/postgresql/data

networks:
  private:
    external: false
  public:
    external: true
    name: tailscale-external
```

```env:.env
DB_PASSWORD=yourpassword
```

:::

### ◎RustDesk

https://rustdesk.com/ja/

TeamViewerのような遠隔操作ツール。

:::details composeファイル

```yaml:compose.yaml
version: '3'

services:
  hbbs:
    container_name: hbbs
    ports:
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21118:21118
    image: rustdesk/rustdesk-server:1.1.10-3-arm64v8
    command: hbbs -r example.com:21117
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    ports:
      - 21117:21117
      - 21119:21119
    image: rustdesk/rustdesk-server:1.1.10-3-arm64v8
    command: hbbr
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped

networks:
  rustdesk-net:
    external: false
  public:
    driver: bridge
    internal: false
    name: tailscale-external

```

:::

### 🪨Obsidian Live Sync

https://github.com/vrtmrz/obsidian-livesync/

Obsidianのファイルをリアルタイムで同期するプラグイン。
このサーバとしてCouchDBを利用している。

:::details composeファイル

```yaml:compose.yaml
volumes:
  ols:
    name: ols
    driver: local
    driver_opts:
      type: none
      device: /mnt/hdd/ols
      o: bind
services:
  olsdb:
    image: couchdb:3.3.3
    hostname: ols
    container_name: ols
    restart: always
    ports:
      - "5984:5984"
    environment:
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD=${COUCHDB_PASS}
    volumes:
      - ols:/opt/couchdb/data
      - ./local.ini:/opt/couchdb/etc/local.ini
    networks:
      public:
networks:
  public:
    external: true
    name: tailscale-external
```

```ini:local.ini
[couchdb]
single_node=true

[chttpd]
require_valid_user = true

[chttpd_auth]
require_valid_user = true
authentication_redirect = /_utils/session.html

[httpd]
WWW-Authenticate = Basic realm="couchdb"
enable_cors = true

[cors]
origins = app://obsidian.md,capacitor://localhost,http://localhost
credentials = true
headers = accept, authorization, content-type, origin, referer
methods = GET, PUT, POST, HEAD, DELETE
max_age = 3600
```

### 📁FileBrowser

https://filebrowser.org/

NextCloudが重すぎたり、内部を直接操作したいときに使うファイルブラウザ。
SMBがいろいろ面倒くさかったので、素直にWebサーバにしてしまいました。

:::details composeファイル

```yaml:compose.yaml
services:
  filebrowser:
    image: filebrowser/filebrowser:v2.31.2
    volumes:
      - type: bind
        source: /mnt/hdd/smb/ownserver
        target: /srv/ownserver
      - type: bind
        source: ./filebrowser.db
        target: /database.db
      - type: bind
        source: ./filebrowser.json
        target: /.filebrowser.json
    hostname: filebrowser
    networks:
      public:
    user: "1000:1000"

networks:
  public:
    external: true
    name: tailscale-external
```

```json:filebrowser.json
{
  "port": 80,
  "baseURL": "",
  "address": "",
  "log": "stdout",
  "database": "/database.db",
  "root": "/srv"
}
```

:::
