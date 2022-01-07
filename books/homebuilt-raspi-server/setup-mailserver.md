---
title: "メールサーバの作成"
---

## メールを受け取れる範囲

今回は自宅サーバであり，自宅内にしか存在しない．
名前解決ができるのは自宅内のみである．
そのため，メールアドレス宛にメールを送信して届けることができるのは自宅内でだけ……と思わせて，gmailを始めとする大概のメールサーバは自宅の外にある．
そのため，受け取れない．残念．

一方送信はどこへでも行うことができる．が，DKIMのような偽造防止技術は送信元に関する情報を取得する際，名前解決が必要なので使うことはできない．

今回はnextcloudのパスワードリセットに使うために使おうと思ってます．

## メール関連用語

### SMTP

Simple Mail Transfer Protocol．メールを送信するための通信プロトコル．
受信はIMAPやPOP等があるが，送信はSMTPのみである．

### postfix

メール転送エージェント．高速らしい．コレを使うことで送信オンリーメールサーバを作成できる．

### DKIM

Domainkeys Identified Mail．DNSサーバを利用して公開鍵をやり取りすることでメールの正当性を検証する．非公開自宅サーバの構成では実現できない．

## Dockerでメールサーバの起動

dockerhubにある[postfixadmin](https://hub.docker.com/_/postfixadmin)はpostfixの管理ツールなので後回し．
mailuさんのpostfixを利用する

```
$ mkdir postfix
$ cd postfix
$ vim docker-compose.yml
```

``` yml:docker-compose.yml
version: '3.8'

services:
  mailserver:
    image: juanluisbaptiste/postfix
    ports:
      - 587:587
    environment:
      - SERVER_HOSTNAME=noreply.local.gunseikpaseri.cf
      - SMTP_SERVER=local.gunseikpaseri.cf
      - SMTP_USERNAME=gunseikpaseri
      - SMTP_PASSWORD=MAwSM413P9BJdWDZjZ7o
    networks:
      public:
        aliases:
          - smtp.local.gunseikpaseri.cf
networks:
  public:
    external: true
    name: public_networks
```

firewallに穴をあける
```
$ sudo ufw allow 587
```


EHLO gunseikpaseri.cf
MAIL FROM: hoge@gunseikpaseri.cf
RCPT TO: yamanoue.paseri@gmail.com
DATA
Subject: test
Hello,
.
QUIT