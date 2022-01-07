---
title: "DNSの設定"
---

# サブドメインの対応
::: message alert
上手く行かないのでこの項目は無視してください．
:::
まず，サブドメインを受け付けるようにavahiの設定を更新して上げる必要があります．

https://www.officeiko.co.jp/post/tech-raspberry-pi-4-ubuntu-nginx-proxy/

https://naokton.hatenablog.com/entry/2020/10/12/062102

この辺を参考に．
元ではLANケーブルを利用しているようだが，自分はWi-Fiなので`eth0`ではなく`wlan0`を使用した．
何を選択すべきかは，`ip a`などで確認してみましょう．
```
$ vim /usr/local/bin/avahi-add-subdomain.sh
```

``` sh:avahi-add-subdomain.sh
#!/bin/bash
set -e
SUBDOMAIN=$1
HOSTNAME=$(uname -n)
NETWORK_INTERFACE=$2
ADDRS=$(ip a show dev $NETWORK_INTERFACE | awk '$1~/inet/{print $2}' | cut -d/ -f1)

for ADDR in $ADDRS; do
    echo "Add [$SUBDOMAIN.$HOSTNAME.local] $ADDR"
    /usr/bin/avahi-publish -a -R $SUBDOMAIN.$HOSTNAME.local $ADDR&
done
wait
```
`$2`で提供される全てのIPアドレスとサブドメイン`$1`を対応付けするスクリプト．
```
$ sudo chmod +x /usr/local/bin/avahi-add-subdomain.sh
$ sudo vim /etc/systemd/system/avahi-subdomain@.service
```
``` ini:avahi-subdomain@.service
[Unit]
Description=Publish %I.%H.local as alias for %H.local via mdns
Requires=avahi-daemon.service
After=avahi-daemon.service

[Service]
Type=simple
ExecStart=/usr/local/bin/avahi-add-subdomain.sh %I
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

上記サービスを新しく登録する．
`avahi-subdomain@XXX.service`の形で新しいドメイン`XXX`を指定できる仕組み．

ステータスは`systemctl status avahi-subdomain@XXX`で確認できます．

実験として，`www`サブドメインを作ってあげます．
現段階では`www.*domain*`にアクセスできないことは確認しておくと良いでしょう．
以下のコマンドで追加できます．
```
$ sudo systemctl enable --now avahi-subdomain@www.service
```

下のようなアプリ使うとIPアドレスの取得はできるので，mDNS自体は飛んでいるかと思いますが，Windowsが解決できてない感じがありますね．

https://play.google.com/store/apps/details?id=com.dokoden.dotlocalfinder

## なんか上手く行かねぇので消す
起動してしまったサービスを確認．
```
$ systemctl list-units --type=service
```
対象の`avahi-subdomain@www.service`を削除
```
$ systemctl stop avahi-subdomain@www.service
$ systemctl disable avahi-subdomain@www.service
```
サービスファイルの削除
```
$ sudo rm /etc/systemd/system/avahi-subdomain@.service
$ sudo rm /usr/local/bin/avahi-add-subdomain.sh
```

# ローカルDNSサーバを利用
結局，mdnsはAndroidからもアクセスできないし，何かと面倒くさいのでルータのDNS機能に甘えるのが一番早いと確認．
ssh等を考えるとmdnsはそれはそれであって欲しい機能なのだけど．

## IPアドレスの固定化
今まではmDNSのお陰で動的配置されたIPアドレスを返してくれましたが，固定すると何かと捗るでしょう．
なにかのトラブルで固定IPが壊れてしまってもつかめるよう，mDNSは一応残しておこうかと思いますが，それはそれとしてルータの方で固定設定をします．
以後，安心してIPアドレスで端末を指定することができます．

### DNSサーバの設定
この状態になれば，DNSサーバはルータに委ねようが自分でおっ立てようが構わない．
管理が楽そうなローカルで設定すべく，DNSサーバを立てたいと思う．（サーバ固有の設定，めんどくさい）
### アドレス範囲の決定
まず，利用したいアドレスをルータのDHCPプールの中から選びます．
そして，ルータ設定で割り当てられていないことを確認，`ping`で存在を問います
```
$ ping 192.168.1.201
PING 192.168.1.201 (192.168.1.201) 56(84) bytes of data.
From 192.168.1.15 icmp_seq=1 Destination Host Unreachable
From 192.168.1.15 icmp_seq=2 Destination Host Unreachable
From 192.168.1.15 icmp_seq=3 Destination Host Unreachable
From 192.168.1.15 icmp_seq=4 Destination Host Unreachable
From 192.168.1.15 icmp_seq=5 Destination Host Unreachable
From 192.168.1.15 icmp_seq=8 Destination Host Unreachable
From 192.168.1.15 icmp_seq=9 Destination Host Unreachable
From 192.168.1.15 icmp_seq=11 Destination Host Unreachable
From 192.168.1.15 icmp_seq=12 Destination Host Unreachable
From 192.168.1.15 icmp_seq=14 Destination Host Unreachable
From 192.168.1.15 icmp_seq=15 Destination Host Unreachable
From 192.168.1.15 icmp_seq=17 Destination Host Unreachable
From 192.168.1.15 icmp_seq=18 Destination Host Unreachable
^C
--- 192.168.1.201 ping statistics ---
19 packets transmitted, 0 received, +13 errors, 100% packet loss, time 18434ms
pipe 4
```
どこにも届かないようなら未使用でしょう．

### ルータで固定アドレス割当の設定
使用するルータには**DHCP固定アドレス割当**という項目があり，ここにMACアドレスと固定するIPアドレスを指定します．

ルータを再起動します．

:::message
ルータを再起動すると，wifi接続が遮断されて，SSH接続が遮断されます．端末を接続するキーボード・モニターが必要です．
:::

ルータの設定で固定設定したipアドレスが割り振られていること，ラズパイで`ifconfig`を叩いて`wan0`のIPアドレスが指定したものに変更されていることを確認します．

## DNSサーバの設立
### ドメインの取得
適当なフリードメインでも構いませんが，とりあえず被らないドメインを一つ取得しておくといいと思います．
そのドメインについて，DNSサーバで勝手にサブドメインを作ろうというのが趣旨です．
筆者環境では`gunseikpaseri.cf`を取得し，`**.local.gunseikpaseri.cf`を利用しますので適宜読み替えてください．

### Dockerによる導入
こちらもDocker-composeを利用します．
DNSサーバの実装は複数あり，`BIND`とか`PowerDNS`とか`dnsmasq`とか，最近だとKubernetes関係からから`coredns`があります．
Go実装であること，設定が容易であること等の理由でなんか`coredns`がワイワイしてて楽しそうなので、こちらを選びます．

```
$ cd ~
$ mkdir dns
$ cd dns
$ vim docker-compose.yml
```

``` yml:docker-compose.yml
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

```
$ mkdir config
$ vim config/Corefile
$ vim config/hosts
```

``` text:config/Corefile
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
Corefileはcorednsのプラグインを列挙する形で設定する．
ローカルでのIPアドレス以外についてのリクエストはGooglePublicDNSとCloudflareDNSに分散してフォワードしている．
いずれも暗号化プロトコルであるDNS over TLSに対応しているため，それを利用することで外部への問い合わせを暗号化している．

|プラグイン名|概要|
|:----------:|----|
|whoami|全てのクエリに対し自身のIPアドレス情報を返答させる|
|errors|エラーとなったクエリを出力する|
|log|全ての入力クエリを標準出力に吐く<br>[オプション等について](https://coredns.io/plugins/log/)|
|reload|Corefile設定ファイルを自動でリロードする|
|hosts|HOSTファイルを読み込む|
|forward|他のゾーンのクエリを外部へ転送する|

``` text:config/hosts
192.168.1.201 local.gunseikpaseri.cf
192.168.1.201 www.local.gunseikpaseri.cf
192.168.1.201 nextcloud.local.gunseikpaseri.cf
```

ポートに穴を開けて、いざ実行
```
$ sudo ufw allow 53
$ docker-compose up -d
```

#### 53ポートを使用中の時
現在53ポートを利用中の旨が出て，`sudo lsof -i:53`すると`systemed-resolve`が利用していることがある．

どうもこれのDNS stub listenerという機能が自分自身をキャッシュサーバとして振る舞うような挙動をしているらしい．

停止するには以下のコマンドを叩く
StubListerの利用の停止（テンプレをシンボリックリンク）
```
$ sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
StubListener機能を停止する設定の追加
```
$ sudo vim /etc/systemd/resolved.conf
```
``` ini:resolved.conf
[Resolve]
DNSStubListener=no
```
再起動
```
$ sudo systemctl restart systemd-resolved
```

#### 動作確認
`dig`して通るか確認すれば良い
```
dig @localhost local.gunseikpaseri.cf
```

### ルータに設定
最後に，サーバのIPアドレスをDNSサーバとしてルータ等に登録する．
こうすることで，ルータ経由のアクセスを行った時，問い合わせがサーバにも飛んでくる．

使用するルータにはDNSアプリケーションの設定項目に，DNSサーバのIPアドレスを登録する箇所があった．

この段階で，ローカルネットワークであればmDNS非対応のAndroidを含むどんな端末でも名前解決ができるようになった．

ただし，全てのDNSリクエストがここを経由するようになることに注意．

大丈夫だとは思うが，CoreDNSコンテナを終了した後もDNS機能が適切に動作するかは確認する必要がある．

# ローカルリバースプロキシを利用




