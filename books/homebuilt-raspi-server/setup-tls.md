---
title: "SSL/TSLのセットアップ"
---

# SSL/TLSとは
雑に言ってしまえば，HTTPS対応をしようという話です．
このご時世HTTPS化しないとやりにくいことも多いみたいなので，早めに対応してしまいましょう．
https化してたら，嬉しいだろ．
SSLは，HTTPS以外にもSMTP(メール)とかPOP3(メール)等様々なウェブ上の通信を保護できるもっと広い存在です．

ちなみにSSLは**深刻な脆弱性**があったりなんだりで名称が**TLS**に変わってます．
なので，本当はTLSと言うべきなんですがSSLが定着しすぎたため，諦めて両方併記しているという事情です．

# 必要なこと
まぁ原理だ何だは各々勉強してもらって，必要なのは**SSL証明書**です．
証明書自体はコマンドで簡単に作れるのですが，その証明書が正当であること（第三者がすり替えた偽物でないこと）を保証するために，**認証局**に署名してもらいます．
この認証局もより上位の認証局に署名してもらい～～ということをして，その大元となっている**ルート証明書**はブラウザに組み込まれている，とかそういう具合です．

で，じゃあ自分も認証局とやらに署名してもらおうかと思うのですが，ローカルドメインだとできません．所有者が一人であることが前提ですから，ローカルのはダメです．
というわけで，この辺を自分で作ります．

# なんかいろいろな変化
とにかく仕様変更の通知が多く，おそらく準拠してないとキレられるので，頑張りましょう．

https://ssl.sakura.ad.jp/column/ssl-90days/

https://college.globalsign.com/blog/cabrowserforum_210810/

# 認証局の作成
認証局の作成には`openssl`のコマンドを使い専用ディレクトリを構築し～～という手順を踏むことになります．
しかし，コマンド打ったり何だり面倒だし自動化も考えるとなぁ…と思っていたところ，どうも`easy-ca`といういい感じのものがあった……
過去形なのは色んな人がフォークしたり何だりで使ってる感じがあるところですね．

https://qiita.com/softark/items/cb92f1ebed8b4b6c55b4

お借りしながらやっていきましょう．
## 組織情報の設定
```
$ mkdir ~/ssh
$ cd ~/ssh
$ git clone https://github.com/softark/easy-ca.git
$ cd easy-ca
$ vim defaults.conf
```
まず，証明書署名要求のデフォルト値の設定をします．
更新などを考えると，これらの値は無闇に変更されないほうが良いでしょう．
| 属性値 | 意味       |
| ------ | ---------- |
| C      | 国(二文字) |
| ST     | 州・県     |
| L      | 都市・地域 |
| O      | 組織名     |
| OU     | 部署名     |
組織名だのはまぁ，お遊びですね．架空の会社作ったりしても楽しいんじゃないですかね．

``` diff ini:defaults.conf
-   CA_DOMAIN="bogus.com"
+   CA_DOMAIN="ownserver.local"

-   CA_CERT_C="US"
+   CA_CERT_C="JP"
-   CA_CERT_ST="California"
+   CA_CERT_ST="Tokyo"
-   CA_CERT_L="San Francisco"
+   CA_CERT_L="Fuchu"
-   CA_CERT_O="Bogus Inc."
+   CA_CERT_O="OwnServer"
-   CA_CERT_OU="Operations"
+   CA_CERT_OU="GunseiKPaseri"
```
## 有効期限の設定
`templates/*.tpl`にはそれぞれの証明書用の設定が含まれている．有効期限については色々設定しておく．
色々あるが，httpsに関係するのは`root.tpl`・`signing.tpl`・`server.tpl`だろう．
### ルート証明書
デフォルトでは5年になっている．30年ぐらいにしておく．30年後どうなってるかは知らん．
```
$ vim templates/root.tpl
```
``` diff ini:root.tpl
[req]
...
-   default_days            = 1826                     # How long to certify for
+   default_days            = 10958                    # How long to certify for
...
[root_ca]
-   default_days            = 1826                       # How long to certify for
+   default_days            = 10958                      # How long to certify for
```
### 中間証明書
デフォルトでは2年だが，これも10年ぐらいにしておく．意味あるのかしらん．
```
$ vim templates/signing.tpl
```
``` diff ini:signing.conf
[req]
...
-   default_days            = 730                      # How long to certify for
+   default_days            = 10958                    # How long to certify for
...
```
### サーバ証明書
デフォルトでは2年だが，390日とか90日とかゆくゆく縮める予定と話題になってたので，いっそ90日にしてしまった．あとで自動化するだろうから短くても構わんやろの精神．
::: message alert
何故かこれだけ効かんかった
:::
```
$ vim templates/server.tpl
```
``` diff ini:server.tpl
[req]
...
-   default_days            = 730                     # How long to certify for
+   default_days            = 90                      # How long to certify for
...
```
## ルート証明書作成
`-d`以下では識別可能な名前を付与する．
パスワードはちゃんと保存しておこう．
`PKCS11`を利用可能にするかみたいなのが問われるが，YubiKeyを使う系の話になるので，`N`を選択する．
```
$ ./create-root-ca -d own-root-ca
[*] Creating root CA in dir 'own-root-ca'
[*] Initializing CA home
[>] Enable PKCS11 Engine for this CA? [y/N]: N
[>] Short label for new CA [own-root-ca]:
[*] Using default CA_NAME_DEFAULT : own-root-ca
[>] Domain name for new CA [ownserver.local]:
[*] Using default CA_DOMAIN_DEFAULT : ownserver.local

[!] CRL URL will be https://ownserver.local/ca/own-root-ca.crl

[>] Country code for new certificates [JP]:
[*] Using default CA_CERT_C_DEFAULT : JP
[>] State for new certificates [Tokyo]:
[*] Using default CA_CERT_ST_DEFAULT : Tokyo
[>] City for new certificates [Fuchu]:
[*] Using default CA_CERT_L_DEFAULT : Fuchu
[>] Organization for new certificates [OwnServer]:
[*] Using default CA_CERT_O_DEFAULT : OwnServer
[>] Organization unit for new certificates [Local]:
[*] Using default CA_CERT_OU_DEFAULT : Local
[>] Common Name for CA certificate [OwnServer Certificate Authority]:
[*] Using default CA_ROOT_CN_DEFAULT : OwnServer Certificate Authority

[>] Enter passphrase for encrypting root CA key:
[>] Verifying - Enter passphrase for encrypting root CA key:
[*] Creating the key (aes256 with 4096 bits)
Generating RSA private key, 4096 bit long modulus (2 primes)
....................++++
...............................................................................................................................................................................................................................................................................................................................................................................++++
e is 65537 (0x010001)
writing RSA key
[*] Creating SSH pub (ca/ssh/ca.ssh.pub)
[*] Example in sshd_config: TrustedUserCAKeys ca.ssh.pub

略

Certificate is to be certified until Nov 14 14:54:35 2051 GMT (10958 days)

Write out database with 1 new entries
Data Base Updated


[*] Creating the root CA CRL
Using configuration from ca/ca.conf

[*] Copying toolchain
[!] Root CA initialized.
```
先程指定した名前（`own-root-ca`）のディレクトリ内に認証局としての機能として必要な諸々が作成される．
|             ファイル          | 意味             |
| ----------------------------- | ---------------- |
|$ROOT_CA_DIR/ca/ca.crl         | 証明書失効リスト |
|$ROOT_CA_DIR/ca/ca.crt         | SSL証明書        |
|$ROOT_CA_DIR/ca/ca.csr         | 証明書署名要求   |
|$ROOT_CA_DIR/ca/ca.pub         | 証明書公開鍵     |
|$ROOT_CA_DIR/ca/private/ca.key | 証明書秘密鍵     |

### 証明書の保存
**SSL証明書**を所定の位置・自作Webサーバ内にコピーする．名前が変更されていることに注意して．
```
$ sudo mkdir -p /etc/pki/tls/certs/
$ sudo cp own-root-ca/ca/ca.crt /etc/pki/tls/certs/own-root-ca.crt
```
### 証明書失効リストのコピー
**証明書失効リスト**をWebサーバに反映しやすくできるよう`.sh`も作っておきましょう．
```
$ cd own-root-ca
$ vim bin/copycrl.sh
```
``` sh:copycrl.sh
mkdir -p ~/webserver/ca/
cp ca/ca.crl ~/webserver/ca/ca.crl
```
```
chmod +x bin/copycrl.sh
bin/copycrl.sh
```

## 中間CA証明書の作成
作業ディレクトリがルートディレクトリであるところと，ルートCAのパスワードが求められることを除けば大体ルート証明書と同様の流れである．
```
$ cd own-root-ca/
$ ./bin/create-signing-ca -d own-intermediate-ca

[*] Creating new signing sub-CA in 'own-intermediate-ca'

[*] Initializing CA home
[>] Enter passphrase for root CA key:
[>] Enable PKCS11 Engine for this CA? [y/N]: N
[>] Short label for new CA [own-intermediate-ca]:
[*] Using default CA_NAME_DEFAULT : own-intermediate-ca
[>] Domain name for new CA [ownserver.local]:
[*] Using default CA_DOMAIN_DEFAULT : ownserver.local

[!] CRL URL will be https://ownserver.local/ca/own-intermediate-ca.crl

[>] Country code for new certificates [JP]:
[*] Using default CA_CERT_C_DEFAULT : JP
[>] State for new certificates [Tokyo]:
[*] Using default CA_CERT_ST_DEFAULT : Tokyo
[>] City for new certificates [Fuchu]:
[*] Using default CA_CERT_L_DEFAULT : Fuchu
[>] Organization for new certificates [OwnServer]:
[*] Using default CA_CERT_O_DEFAULT : OwnServer
[>] Organization unit for new certificates [Local]:
[*] Using default CA_CERT_OU_DEFAULT : Local
[>] Common Name for CA certificate [OwnServer Certificate own-intermediate-ca]:
[*] Using default CA_SUB_CN_DEFAULT : OwnServer Certificate own-intermediate-ca
[>] Enter passphrase for encrypting signing sub-CA key:
[>] Verifying - Enter passphrase for encrypting signing sub-CA key:
[*] Creating the signing sub-CA key
[*] Creating the key (aes256 with 4096 bits)
Generating RSA private key, 4096 bit long modulus (2 primes)
.............................................................++++
........................++++
e is 65537 (0x010001)
writing RSA key
[*] Creating SSH pub (ca/ssh/ca.ssh.pub)
[*] Example in sshd_config: TrustedUserCAKeys ca.ssh.pub

略

Certificate is to be certified until Nov 14 15:46:11 2051 GMT (10958 days)

Write out database with 1 new entries
Data Base Updated
[*] Creating the signing sub-CA CRL
Using configuration from ca/ca.conf
[*] Creating the chain bundle
[*] Verifying trusted chain
ca/ca.crt: OK
ca/ca.crt: OK
[*] Copying toolchain
[!] Signing sub-CA initialized.
```
中間CAはルートCAの中に作成される．

中間証明書もサーバの所定の場所に置いておく．
```
$ sudo cp ca/ca.crt /etc/pki/tls/certs/own-intermediate-ca.crt
```

中間証明書の証明書無効化リストも配布しやすいように`sh`を作っておく
```
$ cp ../bin/copycrl.sh ./bin/
```

そして，サーバ証明書有効期限の設定が何故かここにうまく反映されてないので
``` diff ini:ca/ca.conf
...
[ sign_ca ]
-   default_days            = 730                        # How long to certify for
+   default_days            = 90                         # How long to certify for
```

# サーバ証明書の作成・登録
## サーバ証明書の作成

中間CAで`bin/create-server`を実行する．
`-s`オプションでサーバの説明のような名称を設定，`-a` で使用されるドメインを指定．複数設定できる．
今回は`ownserver.local`・`www.ownserver.local`・`nextcloud.ownserver.local`の3つを登録している．
この内サブドメインの方はまだアクセスできないが，これらのマルチキャストは後で改めて行う．
```
$ cd own-intermediate-ca/
$ bin/create-server -s ownserver -a ownserver.local -a www.ownserver.local -a nextcloud.ownserver.local
省略
```
### 何故か有効期限設定が効かなかった
```
$ cat bin/templates/server.tpl | grep default_days
default_days            = 90                       # How long to certify for
$ openssl x509 -noout -dates -in certs/server/ownserver/ownserver.crt
notBefore=Nov 13 15:59:20 2021 GMT
notAfter=Nov 13 15:59:20 2023 GMT
```
90日という設定自体継承されるが，`ca/ca.conf`が反映されたのでこうなった．
一度削除
```
$ ./bin/revoke-cert -c ./certs/server/ownserver/own
server.crt
[!] Revoking certificate '/home/gunseikpaseri/ssl/easy-ca/own-root-ca/own-intermediate-ca/certs/server/ownserver/ownserver.crt'
[*] Verifying trusted chain
/home/gunseikpaseri/ssl/easy-ca/own-root-ca/own-intermediate-ca/certs/server/ownserver/ownserver.crt: OK
Reason for revocation:
1. unspecified（詳細不明）
2. keyCompromise（鍵が危殆化した）
3. CACompromise（CAが危殆化した）
4. affiliationChanged（所属の変更）
5. superseded（証明書を取り替えた）
6. cessationOfOperation（証明書が不要になった）
7. certificateHold（保留状態に変更する）

[>] Enter 1-7 [1]: 5

[!] You are about to revoke this certificate with reason 'superseded'.

[>] Are you SURE you wish to continue? [y/N]: y

[>] Enter passphase for signing CA key:
[*] Revoke the certificate
Using configuration from ca/ca.conf
Revoking Certificate 01.
Data Base Updated
[*] Regenerate the CRL
Using configuration from ca/ca.conf
[!] Server certificate '/home/gunseikpaseri/ssl/easy-ca/own-root-ca/own-intermediate-ca/certs/server/ownserver/ownserver.crt' revoked.
$bin/create-server -s ownserver.local -a ownserver
.local -a www.ownserver.local -a nextcloud.ownserver.local
[*] Creating new SSL server certificate for:
[*] commonName       ownserver.local
[*] subjectAltName   DNS:ownserver.local, DNS:www.ownserver.local, DNS:nextcloud.ownserver.local

[>] Enter passphrase for signing CA key:
RSA key ok
[>] State for new certificates [Tokyo]:
[*] Using default CA_SERVER_CERT_ST : Tokyo
[>] City for new certificates [Fuchu]:
[*] Using default CA_SERVER_CERT_L : Fuchu
[>] Organization unit for new certificates [Local]:
[*] Using default CA_SERVER_CERT_OU : Local
[*] Usually a server key will not be on a pkcs11 device.
[>] Create csr on pkcs11 device? (key must be in "PIV AUTH key" or 9a) [y/N]: N
省略
```
消してよかったのかわからないから`ownserver`はとりあえず放置で

完成した証明書と秘密鍵を所定の場所にコピー．
```
$ sudo cp certs/server/ownserver-local/ownserver-local.crt /etc/pki/tls/certs/
$ sudo cp certs/server/ownserver-local/ownserver-local.key /etc/pki/tls/certs/
```


## サーバに証明書を登録
前項で作ったwebserverをSSLに対応させます．
```
$ cd ~/webserver
$ docker-compose stop
$ vim docker-compose.yml
```
```diff yml:docker-compose.yml
version: '3'
services:
  web:
    image: nginx
    ports:
      - "80:80"
+     - "443:443"
    volumes:
      - ./src:/usr/share/nginx/html
+     - ./default.conf:/etc/nginx/conf.d/default.conf:ro
+     - ./certs:/etc/nginx/certs:ro
```
`:ro`は読み込み専用なんだそうな．
```
$ mkdir certs
$ cp ~/ssl/easy-ca/own-root-ca/own-intermediate-ca/certs/server/ownserver-local/ownserver-local.crt ./certs/
$ cp ~/ssl/easy-ca/own-root-ca/own-intermediate-ca/certs/server/ownserver-local/ownserver-local.key ./certs/
$ vim default.conf
```
秘密鍵深い...!
``` nginx:default.conf
server {
    listen       80;
    listen       443 ssl;
    server_name  ownserver.local;
    ssl_certificate     /etc/nginx/certs/ownserver-local.crt;
    ssl_certificate_key /etc/nginx/certs/ownserver-local.key;

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
`80`番と`443`番の両方で受け付けるように設定している．
ファイヤウォールに穴を開けて，実行
```
$ sudo ufw allow 443/tcp
$ docker-compose up -d
```
上手く立ち上がらないときは，`docker logs **コンテナ名**`でログを確認できる．


## ローカル証明書の配布
先程作った証明書はクライアント側で登録してあげることで初めて使えるようになります．
ちゃんとしたところならグループポリシーやメールだ何だで配布するのでしょうが，今回はサーバで配布します．（そのために`80`番ポートでも動作するようにしておいた）
ファイルとしてダウンロードできるように，権限も開放します．
```
$ mkdir -p ./src/certs
$ cp ./certs/server.crt ./src/certs/ownserver-local.crt
$ cp ~/ssl/easy-ca/own-root-ca/own-intermediate-ca/ca/ca.crt ./src/certs/own-intermed
iate-ca.crt
$ cp ~/ssl/easy-ca/own-root-ca/ca/ca.crt ./src/certs/own-root-ca.crt
$ chmod 644 src/certs/own*
$ vim src/index.html
```
``` diff html:index.html
+                <script>
+                       if (location.href.indexOf("https") === 0) {
+                               document.body.insertAdjacentHTML('afterbegin',"<b>SECURE ACCESS!!</b>");
+                       } else {
+                               document.body.insertAdjacentHTML('afterbegin',"<b>UNSECURE ACCESS...</b>");
+                       }
+               </script>
...
+               <ol>
+                       <li><a href="/certs/own-root-ca.crt">ルートCA証明書</a></li>
+                       <li><a href="/certs/own-intermediate-ca.crt">中間CA証明書</a></li>
+               </ol>
...
```
## ローカル証明書
インストールします．
自分のWebサーバを開き，先ほど追加したリンクを上から順にダウンロード・実行してインストールしていきます．
### Windows
「証明書のインストール」をクリック
好きな方にインストールします．
自動的に決定

ブラウザで認識されているか確認する．
#### Chrome
「設定」→「プライバシーとセキュリティ」→「証明書の管理」→
ルートCA証明書と中間CA→証明書中間証明機関タブ
サーバ証明書→他の人タブ

`https://*domain*/`でアクセスしてみる．
セキュリティソフトによっては警告が色々出るが無視して通過すると，無事鍵マークのついたHTTPS通信が実現した．


# サーバ証明書作成の自動化
[TODO]

