---
title: "HDDのセットアップ"
---

# HDDのraidを組む
今回組むのは3台によるRAID 5です．

まず，HDD（ケースのUSB）をラズパイに接続します．

接続したHDDのパーティション情報は`fdisk`コマンドで取得できます．
```
$ sudo fdisk -l
...
Disk /dev/sda: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: 2115
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes


Disk /dev/sdb: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: 2115
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes


Disk /dev/sdc: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: 2115
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 byte
...
```


RAID構築には`mdadm`というコマンドを利用します．
```
$ mdadm --version
mdadm - v4.2-rc2 - 2021-08-02
```
RAID5は以下のようなコマンドで取得できます．
```
$ sudo mdadm --create /dev/md1 --level=5 --raid-devices=3 /dev/sda /dev/sdb /dev/sdc
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md1 started.
```
一瞬で終わる．RAID情報は以下のような情報が得られます．
```
$ sudo mdadm --detail /dev/md1
/dev/md1:
           Version : 1.2
     Creation Time : Sun Nov  7 00:06:14 2021
        Raid Level : raid5
        Array Size : 3906764800 (3.64 TiB 4.00 TB)
     Used Dev Size : 1953382400 (1862.89 GiB 2000.26 GB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Sun Nov  7 00:06:35 2021
             State : clean, degraded, recovering
    Active Devices : 2
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : bitmap

    Rebuild Status : 0% complete

              Name : ownserver:1  (local to host ownserver)
              UUID : cc4c859b:a40c6c06:5e20b0e2:ed78e905
            Events : 5

    Number   Major   Minor   RaidDevice State
       0       8        0        0      active sync   /dev/sda
       1       8       16        1      active sync   /dev/sdb
       3       8       32        2      spare rebuilding   /dev/sdc
```

最後に組んだRAIDに対してフォーマット．
ubuntuでのみの利用を想定しているので．安定して高速な`ext4`を利用します．
Windowsと接続する想定がある場合，リムーバブルにする場合ではそれぞれ形式が変わるでしょう．
```
$ sudo mkfs.ext4 /dev/md1
mke2fs 1.46.3 (27-Jul-2021)
Creating filesystem with 976691200 4k blocks and 244178944 inodes
Filesystem UUID: 31c4c7b3-f300-4181-9d60-8cdf0456d208
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```
30秒～1分程度かかりました．
```
sudo mkdir /mnt/hdd
sudo mount /dev/md1 /mnt/hdd
```
このようにすることで，`/mnt/hdd`ディレクトリ以下はハードディスクになりました．
マウント状況は，以下のようなコマンドで確認できます．
```
$ mount
```

# 設定の保存
上記状態だと設定が保存されていないので再起動すると壊れます．
## 再起動しちゃった人
俺です．

`mdadm`の設定は以下のように取得できます．
```
$ cat /proc/mdstat
md127 : active raid5 sdc[3] sdb[1] sda[0]
      3906764800 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      bitmap: 0/15 pages [0KB], 65536KB chunk
```
RAID情報まで壊れているわけではない様子．番号が変わってますね．
```
$ sudo mount /dev/md127 /mnt/hdd
```

## RAID情報の保存
```
$ su
# mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```
## 自動マウント設定
```$ sudo tune2fs -l /dev/md127 | grep UUID
Filesystem UUID:          *FILESYSUUID*
$ sudo vi /etc/fstab
```
以下の行を追加
```
UUID=*FILESYSUUID* /hoge ext4 defaults 0  0 
```
空白はTabかな
