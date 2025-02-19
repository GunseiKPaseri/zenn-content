---
title: "付録 - トラブルシューティング"
---
生じたトラブルのうち本文中に書ききれなかったものを都度メモする。

## ❕/mnt/hddが空に！

mountが外れてしまった様子。

```sh
$ sudo mdadm --detail /dev/md1
[sudo] password for gunseikpaseri:
/dev/md1:
           Version : 1.2
     Creation Time : Sun Nov  7 00:06:14 2021
        Raid Level : raid5
        Array Size : 3906764800 (3.64 TiB 4.00 TB)
     Used Dev Size : 1953382400 (1862.89 GiB 2000.26 GB)
      Raid Devices : 3
     Total Devices : 2
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Wed Dec  8 02:43:32 2021
             State : clean, FAILED
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : bitmap

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       16        1      active sync   missing
       3       8       32        2      active sync   missing
```

対応：HDDのUSBケーブルが抜けてた（この間足を引っ掛けた）
対策：再起動したら再び見ることができるようになった。

安心～～～～

## ❕docker-composeしたのにコンテナが無い！

```
$ docker-compose up -d
[+] Running 1/1
 ⠿ Container reverseproxy-reverse-proxy-1  Started
$ docker ps
CONTAINER ID   IMAGE     COMMAND      CREATED         STATUS         PORTS  NAMES
# ここに表示されるはずのreverseproxy-reverse-proxy-1が無い
```

これは、内部エラーが発生して暗黙的にコンテナが終了しています。
ログを確認し、これに対処します。

```sh
docker logs reverseproxy-reverse-proxy-1
```

## ❕DNSがうまくいかない！

ワイルドカードDNSサービスで代用する場合は以下です。

```nginx:reverseproxy/nginx.conf
...
    server {
        listen 443 ssl;
-       server_name nextcloud.local.gunseikpaseri.cf;
+       server_name nextcloud.local.gunseikpaseri.cf nextcloud.local.192.168.1.201.nip.io;
        ssl_certificate /etc/nginx/certs/ownserver-local.crt;
...
```

nextcloudにhttps接続させる場合は以下です。

```nginx:/mnt/hdd/html/config/config.sample.php
...
  'trusted_domains' =>
  array (
    0 => 'nextcloud.local.gunseikpaseri.cf',
+    1 => 'nextcloud.local.192.168.1.201.nip.io',
  ),
  'datadirectory' => '/var/www/html/data',
...
```

nip.ioでのssl接続は諦めてください…

## ❕`sudo: unable to resolve host ownserver: Name or service not known`って出る！

以下のようなコマンドでhostに自分を追加すると解決する。

```sh
echo $(hostname -I | cut -d\  -f1) $(hostname) | sudo tee -a /etc/hosts
```
