---
title: "SSHの設定"
---
## 🔐SSHとは

外部のターミナル等から端末を操作できる機能です。これによってラズパイは有線キーボード・モニターから開放されます。
同様のことができる機能として一応**telnet**が知られますが、すべての通信が平文であり、パスワードの窃取に脆弱であるため現在では非推奨でしょう。俺も使ったことないです。

## 🔐SSH接続の実現

### #️⃣ポート番号の変更

せっかくなのでセキュリティも勘案してウェルノウンポートである22以外を利用することにします。
ローカルでそのセキュリティいる？と思うかもしれませんが、まぁお遊びなので要ります（意思）
お遊びなんだから本気でやります。
さっき俺のマネをしてOpenSSH向けに開いてしまった人は閉じてもらいます。

```sh
sudo ufw status numbered
```

と打つと、番号付きで表示されるので、その番号をメモっておき、以下のコマンドで削除します。`*NUMBER*`を数字に置き換えてください。

```sh
sudo ufw delete *NUMBER*
```

ちなみに、削除すると番号がずれるので、降順に消すとか都度確認するとかするといいと思います。

次に、使う番号を決めます。49152～65535の間がユーザ自由なのでそのへんから選びましょう。
以下のコマンドで `*PORT*`に好きな番号を入れて、利用状況を確認してみましょう。

```sh
sudo lsof -i:*PORT*
```

利用プログラムが出力されれば利用中、何も出力されなければ未使用です。
未使用領域を見つけたら、ファイヤウォールに許可をしましょう。

```sh
sudo ufw allow *PORT*/tcp
```

avahiの設定もコピペしたものから書き換えます。

```sh
sudo vim /etc/avahi/services/ssh.service
```

`<port>22</port>`となっているところを決めた番号に変えます。

そしていよいよ、sshd_configの設定を書き換えましょう。

```sh
sudo vim /etc/ssh/sshd_config
```

`#Port 22`となっているところのコメントを外し、好きな番号を入れてやりましょう。
以下のコマンドで再起動し、利用するポートが変わっていることを確認します。

```sh
systemctl restart sshd.service
```

### ☑️接続確認

いよいよ、Windows側から接続確認を行います。
以下のようなコマンドを入力すると、端末のパスワード入力を求められるので、入力。侵入できれば接続完了です。

```console
> ssh *USER*@*DOMAIN* -p *PORT*
```

ここまでお疲れさまでした、俺。（ドメイン周りの勘違いで無限に時間を消耗した気がする）
ここから出力コピペできるようになる。

## ⚙公開鍵認証の設定

SSH設定はこれで終わりじゃないです。
公開鍵認証の設定をすることで、ログインパスワードの入力を省略し、即・接続できるようにします。

:::message
SSHの設定を誤っていじって接続不能になるとしんどいので、別のタブで常にssh接続しておくとよいです。一度確立した通信はsshの設定をいじっても維持されます。
:::

参考：
https://qiita.com/c60evaporator/items/ebe9c6e8a445fed859dc
https://qiita.com/c60evaporator/items/2384416f1122ae124f50

:::message
公開鍵の登録はssh-copy-idが便利です。Windowsでは通常使えませんが、GitBashに付いてきます。
:::

### 🪪rootパスワード設定

:::message
セキュリティ的には非推奨ではあります
:::

いざというときの回復用です。

```console
$ sudo passwd root
New password:
Retype new password:
passwd: password updated successfully
```

パスワードマネージャ等でガッチガチの設定&忘れないよう安全に保管しておきましょう。

ただし、`>>`を利用してファイル末尾に書き込むようなときは、`su`コマンドでrootユーザにログインすると便利です。そういうときにrootパスワードが必要になる場合があります。

### 👤ユーザ名の変更

忘れてた。
変更というよりも、新設・移行というのが適切ですね。
`*NEWUSERNAME*`を自分の入れたいユーザ名に置き換えてください。

```console
$ sudo adduser *NEWUSERNAME*
Adding user `*NEWUSERNAME*' ...
Adding new group `*NEWUSERNAME*' (1001) ...
Adding new user `*NEWUSERNAME*' (1001) with group `*NEWUSERNAME*' ...
Creating home directory `/home/*NEWUSERNAME*' ...
Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for *NEWUSERNAME*
Enter the new value, or press ENTER for the default
        Full Name []:
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] y
```

フルネームだのに関して入力が求められますが、無視してよいです。

```sh
$ groups ubuntu
ubuntu : ubuntu adm dialout cdrom floppy sudo audio dip video plugdev netdev lxd
```

既存ユーザが持っていた権限を新アカウントに持っていきます。

```sh
sudo usermod -G ubuntu,adm,dialout,cdrom,floppy,sudo,audio,dip,video,plugdev,netdev,lxd *NEWUSERNAME*
```

最後に、`/home/ubuntu/`のファイルを `/home/*NEWUSERNAME*/`にコピーしましょう。

以上が済んで、新しいユーザでログインし直したりして問題がなさそうであれば、古いユーザは安全のため削除しましょう。

```sh
$ sudo userdel -r ubuntu
userdel: group ubuntu not removed because it has other members.
userdel: ubuntu mail spool (/var/mail/ubuntu) not found
$ sudo ls -a /home/
. .. *NEWUSERNAME*
```

こんな感じになればおけです。

### 👤公開鍵の登録

#### 🆙OpenSSHのバージョン上げ

クライアント側で鍵を生成します。

`C:\Windows\System32\OpenSSH`に存在する `OpenSSH`はやや古く、あまり更新される気配が有りませんので、必要に応じてバージョンアップ等します。

```sh
> ssh -V
OpenSSH_for_Windows_8.1p1, LibreSSL 3.0.2
```

設定>アプリ>アプリと機能>オプション機能 のところでsshを表示し、アンインストールしちゃいます。

https://github.com/PowerShell/Win32-OpenSSH/releases

Portable版のインストール、サービス・パスの設定等。

```sh
> ssh -V
OpenSSH_for_Windows_8.6p1, LibreSSL 3.3.3
```

#### 🔑キーの作成・登録

`ssh-keygen`を利用して鍵を作成します。
これまではRSAを選ぶことが多かったですが、現在はed25519というEdDSA形式も選べます。サイズは256bit固定(意味があるのは内255bit)でパフォーマンスがいいです。
安全性はRSAで2048bit以上4096bit以下という具合なので、安全を取るならRSA4096bitもよいかもしれませんが、ed25519でも十分かと思います。
`-C`で鍵の識別子を入れておくことをおすすめします。

```sh
> ssh-keygen -t ed25519 -C "*COMPUTER_NAME*"
```

出来上がったファイル `id_ed25519`と `id_ed25519.pub`のうち、`id_ed25519.pub`の方をラズパイに登録します。

まず、ラズパイに `scp`でファイルをコピーします。

```sh
>  scp -P *PORT* .\id_ed25519.pub *USER*@*DOMAIN*:
```

最後のコロンは誤字じゃないよ
続いて、`.ssh/authorized_keys`にこのファイルの中身を追記します。

```sh
$ sudo -s
# cat id_ed25519.pub >> .ssh/authorized_keys
```

```sh
sudo vi /etc/ssh/sshd_config
```

`PubkeyAuthentication`や `AuthorizedKeysFile`という記述を見つけてコメントアウトを除去します。

```text:sshd_config
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys
```

パーミッションを以下のように設定します。

```sh
sudo chmod 755 /home/*USERNAME*/
sudo chmod 700 /home/*USERNAME*/.ssh/
sudo chmod 600 /home/*USERNAME*/.ssh/authorized_keys
```

`.ssh`と `.ssh/authorized_keys`の所有者が `*USERNAME*`になっているか確認する。

```sh
$ ls -la ~/.ssh
drwx------ 2 *USERNAME* *USERNAME* 4096 Nov  6 22:29 .ssh
```

`root`とかになっていたら変更しておく。

```sh
chown *username*:*username* .ssh
```

この段階で、以下のコマンドで接続できるようになっているかと思います。

```sh
$ ssh *username*@*domain* -i *path to private key* -p *port*
```

#### ⚙接続設定の記述

端末側の `~\.ssh\config`に以下のよう記述する。

```diff
+     Host *HOSTNAME*
+           User *USERNAME*
+           HostName *DOMAIN*
+           Port *PORT*
+           IdentityFile ~\.ssh\id_ed25519
```

`IdentityFile`には秘密鍵の方を登録します。
以後は以下のコマンドで接続できるようになります。

```sh
ssh *HOSTNAME*
```

ようやく簡単に接続できるようになりました。

## 💬SSH時メッセージの追加

イケてるメッセージを含む適当なファイルを作ります。（例：`/etc/ssh/login_message.txt`）

```txt:login_message.txt
The man who has no imagination has no wings.
      ---- Muhammad Ali (1942-2016)
```

> 想像力がないやつは翼が持てない
>      ---- ムハンマド・アリ（1942-2016）

君の好きな格言を入れてかっこいい感じにしましょう。

`chmod`で `644`に変更したうえで再び設定ファイルを開きます。

```sh
sudo vi /etc/ssh/sshd_config
```

`Banner`という記述を見つけて設定を追加します。

```diff
+     Banner /etc/ssh/login_message.txt
```

最後に `ssh`の再起動します。

```sh
systemctl restart sshd
```

```diff
+     PasswordAuthentication no
```

## ⏱️二段階認証の追加

あの6桁の入力するやつです。あれをssh時必須化します。
ここまでやる必要があるか？　と言われたら**全くない**けど、面白そうなので……
クソデカQRコードが表示されるので、画面を大きくしておくと嬉しいと思います。

```sh
$ sudo apt install libpam-google-authenticator
$ sudo google-authenticator
Do you want authentication tokens to be time-based(y/n)y
（時間ベースのものにするか？→はい）
クソデカQRコード
Enter code from app (-1 to skip): *6NUMBER*
Do you want me to update your "/home/*USERNAME*/.google_authenticator" file?(y/n) y
（設定ファイル書き出すか？→はい）
Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n)
（ワンタイムパスワードは一度しか使えないようにするか？→いいえ　複数ターミナルのときとか考えると）
By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n)
（遅延も考慮して有効期限伸ばす？→いいえ）
If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting?(y/n)
（30秒あたり3回に制限する？→はい）
```

設定を終了するとホームディレクトリに `.google_authenticator`ファイルが出来上がる。設定項目やシークレットキー、緊急コードが記録されている。

続いて、ログイン系に介入するため、PAM(Pluggable Authentication Module)の設定です。

```sh
sudo vi /etc/pam.d/sshd
```

先頭の `common-auth`のインクルードのコメントアウト、末尾にオーセンティケータの設定の追加します。

```diff
+     # @include common-auth

...

+     auth required pam_google_authenticator.so nullok
```

`nullok`は2段階認証が非設定のユーザ向けの項目です。

SSHサーバに設定を加えます。

```sh
sudo vi /etc/ssh/sshd_config
```

以下のように変更する。

```diff
+     ChallengeResponseAuthentication yes
...
+     UsePAM yes
...
+     AuthenticationMethods publickey,keyboard-interactive

```
