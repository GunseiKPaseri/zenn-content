---
title: "tmuxの導入"
---

# tmuxとは

コマンドラインでマルチウィンドウを実現するツール．
操作がやや複雑だが，複数のデバイスを確認できる．


# tmuxの導入

日本語版のReadmeが存在する．

https://github.com/tmux/tmux/blob/master/README.ja

以下をインストールしておくと良い
```
$ sudo apt install git automake bison build-essential pkg-config libevent-dev libncurses5-dev
```

workspaceを作り，以下でビルド作業をする
```
$ mkdir workspace
$ cd workspace
$ git clone --depth 1 https://github.com/tmux/tmux.git
$ cd tmux
$ sh autogen.sh
$ ./configure && make
...
$ tmux -V
tmux 3.1c
```

# tmuxの利用方法

## セッション
GUIのウィンドウに相当するものがセッションである．
セッションが表示されていない状態であっても内部で動作していることが有る．
セッションの一覧は以下のコマンドで確認できる．
```
$ tmux ls
no server running on /tmp/tmux-1001/default
```

## prefix
tmuxを操作する際のショートカットは，prefixと呼ばれるキーバインドを入力した後に入力される．
prefixはデフォルトで`Ctrl+b`に設定されていて，自由に変更することができる．

### 基本操作
|キーバインド|説明|
|:----:|:----:|
|`prefix`+`s`|セッション一覧・選択|
|`prefix`+`d`|セッションから離脱（デタッチ）|

### ペイン操作
|:----:|:----:|
|`prefix`+`%`|左右にペイン分割|
|`prefix`+`"`|上下にペイン分割|
|`prefix`+`o`|ペインを順に移動|
|`prefix`+`x`|ペインを破棄|


## セッションの起動
セッション名を指定して起動することで管理がしやすくなる

```
$ tmux new -s <セッション名>
```
このセッションの中でdocker-composeを動かしてログを閲覧しやすくするなどの利用ができる．

なお，`tmux`と入力しても新規セッションが開始される．
普通にtmuxを再開する場合は`tmux a`で起動する．

# 各docker-composeについてセッションを作成

docker-composeを起動した各ディレクトリに移動し，`docker-compose stop`，`tmux new -s <セッション名>`でセッションを起動，その中で`docker-compose up`を行う．
これにより，マルチディスプレイでdockerを管理できるようになる．




