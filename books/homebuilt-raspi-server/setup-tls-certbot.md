---
title: "SSL/TSLのセットアップ(Certbot編)"
---
## 🤔オレオレ認証の問題点

問題点：嫌。

## 🔐Let's Encrypt

Let's Encrypt はフリーで自動化されたオープンな認証局です。

https://letsencrypt.org/ja/

OV(実在証明型)やEV(実在証明拡張型)といった高レベルの証明書を求めず、DV(ドメイン認証)さえ実現できればよいならLet's Encryptでも十分です。

## 🤖certbot

Let's Encrypt で証明書を発行するコマンドです。apacheの設定を利用する等の機能を持ちますが、今回はマニュアルモードで発行します。

```sh
cd ~/ssl/certbot/
```

まずdockerでcertbotを利用してコマンドを作成するファイルを作成。

```sh:certonly.sh
#!/bin/bash

docker run \
  --rm \
  -it \
  --name letsencrypt \
  -v /etc/group:/etc/group:ro \
  -v /etc/passwd:/etc/passwd:ro \
  -u $(id -u $USER):$(id -g $USER) \
  -v "$(pwd)"/letsencrypt:/etc/letsencrypt \
  -v "$(pwd)"/lib/letsencrypt:/var/lib/letsencrypt \
  certbot/certbot:arm64v8-v1.28.0 \
  certonly \
    --manual \
    --preferred-challenges dns \
    -d local.gunseikpaseri.cf \
    -d *.local.gunseikpaseri.cf \
    -m gunseikpaseri@gmail.com \
    --agree-tos
```

RasPi環境なのでarm64v8を指定する必要がある。必要に応じて変更してあげる。

実行する。処理中に他のコマンドを叩いたり、待ってる間にSSHが切れる可能性もあるため、tmux上での処理を推奨する。

```sh
chmod +x certonly.sh
tmux a
./certonly.sh
```

正常に起動すると、作成のコンソールが出てくる。

> Would you be willing, ...
> Let'sEncryptに関するお知らせを購読するためにメルアドを登録するか？`N`でよい。
> Please deploy a DNS TXT record under the name:
>
> _acme-challenge.gunseikpaseri.cf.
>
> with the following value:
>
> XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

ここで出力されたコードをDNSのTXTレコードに登録する。

自分はfreenomのDNSサーバを利用しているため、freenomのサイトに移動し、Service>MyDomainsを選択
自分のドメインについてManageDomainを開く。
ManageFreenomDNSタブを開く。

Add Recodesに以下のように指定。

|      Name      | Type | TTL |    Target    |
| :-------------: | :--: | :--: | :----------: |
| _acme-challenge | TXT | 3600 | (例のコード) |

tmux等を利用して別ウィンドウで以下のコマンドで先程指定した値が取得できるようになるのを待つ。

```sh
nslookup -type=TXT _acme-challenge.local.gunseikpaseri.cf 8.8.8.8
```

ここでEnterを入力するともう1つ入力が求められる。（求めるドメインの数だけ入力する必要がある様子）
新しいレコードをまた追加し、同様に取得できるようになるまで待ち、Enterを入力。

成功するとディレクトリ下に作成される。
このとき所有権がrootになっていることに注意する。

```sh
$ sudo ls letsencrypt/live/gunseikpaseri.cf
README  cert.pem  chain.pem  fullchain.pem  privkey.pem
```

## 🔐反映

リバースプロキシを停止。

tmuxを利用しているときは `Ctrl+b`+`s`で移動、docker-composeで稼働中のコンテナを `ctrl+c`で停止する。

既存のものを除去。

```sh
cd ~/reverseproxy/certs
mkdir old
mv ownserver-local.crt old/
mv ownserver-local.key old/
```

コピーして所有権を自分のものにして前のものに置き換える。

```sh:copycert.sh
#!/bin/bash
cp /home/"$SUDO_USER"/ssl/certbot/letsencrypt/live/local.gunseikpaseri.cf/fullchain.pem ./ownserver-local.crt
cp /home/"$SUDO_USER"/ssl/certbot/letsencrypt/live/local.gunseikpaseri.cf/privkey.pem ./ownserver-local.key
chown $SUDO_USER:$SUDO_USER ./ownserver-local.crt
chown $SUDO_USER:$SUDO_USER ./ownserver-local.key
```

```sh
sudo ./copycert.sh
```

最後にリバースプロキシを起動する。

各サブドメインに正常にアクセスできるか確認する。

最後に、WebサーバからCA証明書のDLリンクを削除する。

### 🆙更新

```sh:renew.sh
#!/bin/bash
./certonly.sh
docker run \
  --rm \
  -it \
  --name letsencrypt \
  -v /etc/group:/etc/group:ro \
  -v /etc/passwd:/etc/passwd:ro \
  -u $(id -u $USER):$(id -g $USER) \
  -v "$(pwd)"/letsencrypt:/etc/letsencrypt \
  -v "$(pwd)"/lib/letsencrypt:/var/lib/letsencrypt \
  -v "$(pwd)"/auth:/auth
  certbot/certbot:arm64v8-v1.28.0 \
  renew \
    --manual \
    --preferred-challenges dns \
    --manual-auth-hook /auth/hook.sh
```

直下に `auth`ディレクトリを作成。

root権限で以下のファイルを作成。

```sh:auth/hook.sh
#!/bin/bash 
echo "$CERTBOT_VALIDATION" > /auth/validation_token
sleep 300
```

以下のコマンドを実行すると、authディレクトリに認証トークンが出力される。
5分以内にこの内容をDNSのTXTフィールドに反映させる。

```sh
chmod +x renew.sh
tmux a
sudo ./renew.sh
```

んーでもあんまりうまく動かないな。
