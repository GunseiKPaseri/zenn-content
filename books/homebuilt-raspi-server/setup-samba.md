---
title: "Sambaの設定"
---

```bash
mkdir /mnt/hdd/smb
mkdir samba
```

``` yml:samba/docker-compose.yml
version: '3'
services:
  samba:
    image: dperson/samba
    container_name: samba
    ports:
      - "137:137/udp"
      - "138:138/udp"
      - "139:139/tcp"
      - "445:445/tcp"
    volumes:
      - /mnt/hdd/smb:/mnt/pub:rw
    environment:
        - TZ=Asia/Tokyo
        - USERID=1000
        - GROUPID=1000
    command:
      '-u "smbreader;hogehogefugafuga" -u "smbwriter;fugafugapiyopiyo" -s "share;/mnt/pub;yes;yes;no;smbreader,smbwriter;none;smbwriter"'
    restart: unless-stopped
```
コマンドでは最低限読み取り権限のみのユーザーと，書き込み権限もあるユーザを作成している．

以下の結果の`uid`と`gid`で見れる
```
$ id
```
関連ポートを許可する
```
$ sudo ufw allow 137/udp
$ sudo ufw allow 138/udp
$ sudo ufw allow 139/tcp
$ sudo ufw allow 445/tcp
```


commandでSMBの一覧やユーザのアクセス制限などを記述する
オプションは以下

```
-s "<name;/path>[;browse;readonly;guest;users;admins;writelist;comment]"
            Configure a share
            required arg: "<name>;</path>"
            <name> is how it's called for clients
            <path> path to share
            NOTE: for the default values, just leave blank
            [browsable] default:'yes' or 'no'
            [readonly] default:'yes' or 'no'
            [guest] allowed default:'yes' or 'no'
            NOTE: for user lists below, usernames are separated by ','
            [users] allowed default:'all' or list of allowed users
            [admins] allowed default:'none' or list of admin users
            [writelist] list of users that can write to a RO share
            [comment] description of share
-u "<username;password>[;ID;group;GID]"       Add a user
            required arg: "<username>;<passwd>"
            <username> for user
            <password> for user
            [ID] for user
            [group] for user
            [GID] for group
```
