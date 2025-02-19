---
title: "DNSの設定"
---
## サブドメインの対応

### 📟セルフホストしたDNSの利用

TailscaleにはカスタムのDNSサーバを指定し、それを参照させる機能がある。そしてこれは、Tailscale内のサーバに向けることができる。

セルフホストしたDNSに設定値を書き込むことで、ドメイン名の隠匿とサービスの手元管理が実現する。

### 📟DNSサーバの設立

#### 🇨🇫ドメインの取得

適当なフリードメインでも構いませんが、とりあえず被らないドメインを1つ取得しておくといいと思います。
そのドメインについて、DNSサーバで勝手にサブドメインを作ろうというのが趣旨です。
筆者環境では `gunseikpaseri.cf`を取得し、`**.local.gunseikpaseri.cf`を利用しますので適宜読み替えてください。

#### 🐋Dockerによる導入

こちらもDocker-composeを利用します。
DNSサーバの実装は複数あり、`BIND`、`PowerDNS`、`dnsmasq`、最近だとKubernetes関係からか `coredns`が盛んみたいです。
Go実装であること、設定が容易であること等の理由でなんか `coredns`がワイワイしていて楽しそうなので、こちらを選びます。

```sh
cd ~
mkdir dns
cd dns
vim docker-compose.yml
```

```yml:docker-compose.yml
version: '3.1'
services:
  coredns:
    image: coredns/coredns
    container_name: coredns
    restart: always
    command: -conf /etc/coredns/Corefile
    expose:
      - '53'
      - '53/udp'
    ports:
      - '53:53'
      - '53:53/udp'
    volumes:
      - './config:/etc/coredns'
```

```sh
mkdir config
vim config/Corefile
vim config/hosts
```

```text:config/Corefile
.:53 {
    whoami
    forward . 127.0.0.1:5301 127.0.0.1:5302
    errors
    log . "{proto} {remote} is Request: {name} {type} {>id}"
    hosts /etc/coredns/hosts {
        fallthrough
    }
    reload
}

.:5301 {
    forward . tls://8.8.8.8 tls://8.8.4.4 {
        tls_servername dns.google
        force_tcp
    }
}
.:5302 {
    forward . tls://1.1.1.1 tls://1.0.0.1 {
        tls_servername cloudflare-dns.com
        force_tcp
    }
}
```

Corefileはcorednsのプラグインを列挙する形で設定する。
ローカルでのIPアドレス以外についてのリクエストはGooglePublicDNSとCloudflareDNSに分散してフォワードしている。
いずれも暗号化プロトコルであるDNS over TLSに対応しているため、それを利用することで外部への問い合わせを暗号化している。

| プラグイン名 | 概要                                                                                            |
| :----------: | ----------------------------------------------------------------------------------------------- |
|    whoami    | すべてのクエリに対し自身のIPアドレス情報を返答させる                                            |
|    errors    | エラーとなったクエリを出力する                                                                  |
|     log     | すべての入力クエリを標準出力に吐く `<br>`[オプション等について](https://coredns.io/plugins/log/) |
|    reload    | Corefile設定ファイルを自動でリロードする                                                        |
|    hosts    | HOSTファイルを読み込む                                                                          |
|   forward   | 他のゾーンのクエリを外部へ転送する                                                              |

```text:config/hosts
100.x.y.z local.gunseikpaseri.cf
100.x.y.z www.local.gunseikpaseri.cf
100.x.y.z nextcloud.local.gunseikpaseri.cf
```

`100.x.y.z`はサーバのIPアドレスを指す。

ポートに穴を開けて、いざ実行。

```sh
sudo ufw allow 53
docker-compose up -d
```

##### 53ポートを使用中のとき

現在53ポートを利用中の旨が出て、`sudo lsof -i:53`すると `systemed-resolve`が利用していることがある。

どうもこれのDNS stub listenerという機能が自分自身をキャッシュサーバとして振る舞うような挙動をしているらしい。

停止するには以下のコマンドを叩く
StubListerの利用の停止（テンプレをシンボリックリンク）

```console
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

StubListener機能を停止する設定の追加。

```sh
sudo vim /etc/systemd/resolved.conf
```

```ini:resolved.conf
[Resolve]
DNSStubListener=no
```

再起動。

```sh
sudo systemctl restart systemd-resolved
```

##### 動作確認

`dig`コマンドで通るか確認すればよい。

```sh
dig @localhost local.gunseikpaseri.cf
```
