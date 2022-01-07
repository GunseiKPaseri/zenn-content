---
title: "Docker・Webサーバのセットアップ"
---

# Dockerとは
コンテナ型の仮想化技術の一つです．
多くの仮想化技術がOSから組み立てるのに対し，Dockerはコンテナと呼ばれるそれよりもスリムで，それでいて独立した環境を作ります．

これをすると何が嬉しいかと言うと，設定ファイル書いて立ち上げれば勝手に独立したアプリが立ち上がってくれるので，環境の衝突だの何だの怪しい問題をほとんど踏まずに済むようになります．
Docker環境が動けばもう勝ちみたいなもんです（本当か？）

もちろんそのアプリ内でのトラブルには遭遇するでしょうし，Docker自体が問題を起こすこともありますが，環境が独立することで考えるべき要因が大幅に減ってくれるのです．
# Dockerの導入
https://docs.docker.com/engine/install/ubuntu/

`sudo apt install`系で入手できた`docker`，`docker.io`，`docker-engine`は古いバージョンなので消せとのことです．
```
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```
指示通りにインストールします．aptは何かと反映が遅くなりがちですが，aptの取得先にdockerのページを設定しているようですね．そういう対処方法もあるのか……
```
$ sudo apt install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd.io
```
Docker-composeもインストールできます．
```
$ sudo curl -L https://github.com/docker/compose/releases/download/v2.1.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```
正式なバージョンは以下参照

https://github.com/docker/compose/releases

実行権限の付与も行います．
```
$ sudo chmod +x /usr/local/bin/docker-compose
```

かくして，以下の通りインストールできました．
```
$ docker -v
Docker version 20.10.10, build b485636
$ docker-compose -v
Docker Compose version v2.1.0
```

試しにコマンドを実行すると以下のようなエラーが出るかもしれません．
```
$ docker ps
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json": dial unix /var/run/docker.sock: connect: permission denie
```
自分をDockerグループに追加，
```
$ sudo gpasswd -a $(whoami) docker
$ id $(whoami)
..... groups=....数字(docker)みたいな記述
```
一度ログアウトして再度入り直すと動きます．

`hello-world`を実行して動作確認してみましょう
```
$ docker pull hello-world
$ docker image ls
REPOSITORY    TAG       IMAGE ID       CREATED       SIZE
hello-world   latest    18e5af790473   6 weeks ago   9.14kB
$ docker run hello-world
docker: Error response from daemon: failed to create endpoint jovial_almeida on network bridge: failed to add the host (veth2d04f4b) <=> sandbox (vethec11f05) pair interfaces: operation not supported.
```

## Apacheサーバの導入
まず，なんか色々な情報が表示できるWebサーバを立てておきましょうか．とりあえずnginxにしとく？
```
$ mkdir webserver
$ mkdir webserver/
$ vim webserver/docker-compose.yml
```
``` yml:docker-compose.yml
version: '3'
services:
  web:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./src:/usr/share/nginx/html
```
```
$ mkdir src
$ vim webserver/src/index.html
```
``` html:index.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>ようこそ</title>
  </head>
  <body>
    <h1>ようこそ</h1>
    <ul>
      <li><a href="http://*domain*">HTTP版トップページ</a></li>
      <li><a href="https://*domain*:8080">HTTPS版トップページ</a></li>
      <li><a href="https://*domain*">NextCloud</a></li>
    </ul>
  </body>
</html>
```
Webサーバにアクセスできるよう，80番ポートは空けておきます．
```
$ sudo ufw allow 80/tcp
```
いざ実行します
```
$ docker-compose up -d
```
手元のブラウザで，`http://*domain*`にアクセスしてみましょう．（80番はウェルノウンポートなのでポート番号不要）
ドメインだけを入力すると，Chromeだと`.local`をドメインと認めてないせいかGoogle検索に飛びそうになるので，プロトコルから入れてあげるといいですね．

以後，自分のサーバの情報はここに記述しておけば誰もが簡単に閲覧できるようになるんじゃないですかね．
