---
title: "Ubuntu・ラズパイのセットアップ"
---

# OS起動までのセットアップ
今回買ったセットに含まれたSDカードは既にNOOBSでOSがセットアップされてましたが，自分はUbuntuを使いたかったのでUbuntuを入れます．

まず，以下から**Raspberry Pi Imager**をダウンロード・インストールします

https://www.raspberrypi.com/software

CHOOSE OS > Other general purpose OS > Ubuntu > UBUNTU SERVER 21.10(RPI 3/4/400)を選択

利用するラズパイが64-bit対応なので．

状況によってはバージョンが異なる可能性がありますが，必要なものを選択してください．

そして，Storageで該当するmicroSDカードを選択し，WRITEをクリックします．

ここから非常に時間がかかるので，その間にラズパイにヒートシンク付けたりファンつけたりしていればいいと思います

ケーブル類を指して、USBTypeC端子に電源を通すと、自動でUbuntuServerが起動する．
:::message
ラズパイの操作には**外付けUSBキーボード**・**HDMI接続モニター**が必要です．また，後述の処理のため一時的に**有線LAN**が必要です．
:::

# UbuntuServer起動後のセットアップ
## ログイン
:::message
初期状態ではUSキーボードになっているため，日本語キーボードの記号入力とはキータッチが異なります．
:::
コンソールが荒れることもあるが，最初に求められるのはログインである．初期ユーザ名`ubuntu`・パスワード`ubuntu`を入力することでログインできる．初回ログイン時はパスワードの変更を求められるので，新しい安全なパスワードを設定しましょう．
これを入力することで，シェルに遷移します．
## ホスト名の変更
初期ホスト名は`ubuntu`でわかりにくいので，以下のようなコマンドで`NEWNAME`に書き換えます
```
$ sudo hostnamectl set-hostname NEWNAME
```

現在のホスト名は
```
$ hostname
```
で確認できます．

## インターネットへの接続
有線ならばケーブルを接続すればよいですが，無線は少しめんどくさいです．
とりあえず，有線で接続して`sudo apt install network-tools`を実行しておいてください．


`*YOUR_WIFI_SSID*`には利用するwi-fiのSSID（接続先名）を，`*YOUR_PASSCODE*`に接続先のパスワードを入力
```
$ sudo nmcli connection add \
type wifi \
ifname '*' \
con-name *YOUR_WIFI_SSID* \
ssid *YOUR_WIFI_SSID*
$ sudo nmcli connection edit *SSID*
nmcli> set connection.autoconnect yes
nmcli> goto 802-11-wireless-security
nmcli 802-11-wireless-security> set key-mgmt
value: WPA-PSK
nmcli 802-11-wireless-security> set psk-flags
value: 0
nmcli 802-11-wireless-security> set psk
value: *YOUR_PASSCODE*
nmcli 802-11-wireless-security> psk-flags
value: 0
nmcli 802-11-wireless-security> save
```
WPA-PSKという設定項目は使っているのがwpa2-pskだとかそういうのは考えなくてよいです．

参考：
http://vega.pgw.jp/~kabe/linux/c7-nm-wifi.html


再起動などして無線で自動接続されることなどを確認しておきましょう．
```
$ shutdown -r now
```

## Ubuntuの日本語化

### タイムゾーンの設定
```
$ timedatectl
```
この出力の*LocalTime*を*JST*にします．
ログ出力などで時間が日本標準時じゃないと苦しいでしょう．
```
$ sudo timedatectl set-timezone Asia/Tokyo
```
再度確認してJSTになっていることを確認しましょう
### キーボードを日本語配列に
初期設定では，おそらく記号などが思うように入力できないかと思います．
```
$ localectl
```
この出力の*x11 Layout*が*us*等になっているでしょう．これを日本語配列にします．
```
$ sudo localectl set-x11-keymap jp jp106
```

### 日本語化
```
$ localectl
```
この出力の`System locale`が`LANG=C.UTF-8`になっているかと思います．
```
$ localectl liist-locales
```
で選択可能な言語環境が出力されます．`ja_JP.UTF-8`がない場合は日本語言語パッケージを追加します．
```
$ sudo apt install -y language-pack-ja
```
追加された場合以下のコマンドでロケールを変更します．
```
$ sudo localectl set-locale LANG=ja_JP.UTF-8
```
:::message
出力が文字化けするようなら同様のコマンドで`C.UTF-8`に戻しましょう．
:::

## 名前解決
### localドメインがウンタラカンタラ
まぁ調べるとActiveDirectoryで.localを使うなという話があったので，変えようとしたのですがこれは不要みたいです…？むしろmdnsが解決しなくなる．（別空間になってる？）
つまり，`/etc/mdns.allow`にドメイン名を書き込むとか，`/etc/avahi/avahi-daemon.conf`のドメイン名の項目をいじるとか，`/etc/nsswitch.conf`を書き換えるとか，そういうのはいらないみたいです？

### avahi-daemonの導入
mDNS（動的なDNSの設定）にはavahiというZeroconf準拠ツールを利用します．
これにより，ローカルネットではDNSサーバを立てること無く名前解決することができるようになります．
```
$ sudo apt install avahi-daemon
$ sudo systemctl start avahi-daemon
$ sudo systemctl enable avahi-daemon
```
### ファイヤウォールでmDNSの許可
ファイヤウォールは，各ポートに対し外部からの不正な操作を遮断するアプリケーションです．
ポートは迂闊に空けておくと，不正アクセスの元なので，できる限り塞ぐようにします．

しかしmDNSはポート`5353`のUDPを利用するのでファイアウォールの穴を開けておく必要があるため（逆にそれ以外の不要なポートは塞ぐ必要がある）ため，その設定を行います．
Ubuntuにはデフォルト`ufw`があるはずなので，それを利用する．ない場合は入れるなりfirewalldなどを使うなりする．以下は`ufw`を利用する．
```
$ ufw --version
Copyright 2008-2021 Canonical Ltd.
```
自動起動の有効化設定を行う．
```
$ systemctl enable ufw
$ systemctl start ufw
```
本体有効化（この時他の通信ができなくなる）・一度全てのポートを拒否
```
$ sudo ufw enable
$ sudo ufw default deny
```
OpenSSH(ついでに)，mdnsの有効化
```
$ sudo ufw allow OpenSSH
$ sudo ufw allow mdns
```
ファイヤウォールの状態を確認するのは以下のコマンドで．
```
$ sudo ufw status verbose
```
### mdnsの有効化

`/etc/avahi/services`には利用するサービスをxml形式で追加する．
sshサービスに関する設定情報を予め用意された例からコピペします．
```
$ sudo cp /usr/share/doc/avahi-daemon/examples/ssh.service /etc/avahi/services/
```

### mdnsの動作の確認
#### 正引き
```
$ dig *DOMAIN*. @224.0.0.251 -p 5353
$ avahi-resolve -n *DOMAIN*
```
`224.0.0.251`はmDNS用に予約されたリンクアドレスなので環境によらず共通（なはず），ポートも同様．
ここでちゃんと当該端末のipアドレスが出力されているだろうか．

#### 逆引き
`*IP*`をあなたの環境に合わせて置き換えてください．
```
$ dig -x *IP* @224.0.0.251 -p 5353
$ avahi-resolve -a *IP*
```
ここで設定したドメインが出力されているかどうかを確認してみよう．

## ラズパイ用コマンドの追加
ラズパイハード向けの諸々を含むパッケージを追加する．USB用kernel moduleやらdocker実行周辺の何か？とかとにかくないと変なバグに陥るようである．
```
$ sudo apt install linux-modules-extra-raspi
```

これを導入することで`vcgencmd`なども入る様子．
```
$ vcgencmd measure_temp
temp=45.2'C
```
CPU温度の取得が簡単にできるようになる．
ちなみに，上記コマンド以外にも，以下のような方法で温度を取得できる（ミリ単位だが）
```
$ cat /sys/class/thermal/thermal_zone0/temp
45277
```
