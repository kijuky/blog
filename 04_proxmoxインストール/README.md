我が家ではsynologyを飼っているがcpu,メモリが貧弱なので、コンテナ起動用に中古のミニPCを買った。Lenovo M720qで、さらにこいつのcpuをi9 9900Uに、メモリを64GBに、ストレージを2TBに換装した。最初にwindowsが入っていたのでhyper-vで遊んでいたのだが、チャッピーと話していると

「え、なんでそのスペックでwindows hyper-vで遊んでるの？」

「proxmoxつかいなよ」

みたいに煽られたので、proxmoxをインストールすることとなった。この記事はwindowsからproxmoxに移行するまでの記録です。

# 1. windowsのバックアップ

現状のwindows環境をproxmoxに移行した後も使えるようにします。そのためにDisk2vhdを使って、現環境の仮想ハードディスクを作成します。

作成先は外付けHDDにします。私はたまたま外付けHDD 6TBを持っていたので、そこに作成しました。

# 2. proxmoxの準備

proxmoxのインストールイメージをダウンロードして、USBメモリに書き込みます。私はRufusを使いました。

# 3. proxmoxのインストール

誤操作による仮想ハードディスクの消失を防ぐため、6tbの外付けHDDを外します。その後、先ほど作成したUSBメモリを起動ディスクとして起動します。proxmoxのインストーラーが起動するのでいい感じに設定します。途中でホスト名の入力を促されるため、ここでは `lenovo-m720q.home.arpa`（内向きDNS）としました。IPアドレスはルーター側で固定化しておきます。

# 4. proxmox無料版の設定

proxmoxは、初期設定だと有料（サブスクリプション）のリポジトリが登録されています。家庭用のため、無料版のリポジトリに変更しておきます。

https://qiita.com/AkikanPHP/items/c6a5d2dbce3439aa6490

# 5. VHDXをQCOW2に変換

proxmoxではVHDX形式の仮想ハードディスクをそのまま扱いづらいため、QCOW2形式に変換して読み込みます。ここからは楽しいシェルのお時間です。まずはsshで接続します。

```shell
$ ssh root@[IPアドレス]
```

次にVHDXファイルがある外付けHDDをマウントします。ただし、変換作業を少しでも早くするため、SATA接続のHDDにコピーします。SATA接続のHDDは未フォーマットのため、ext4にフォーマットして、そこにVHDXをコピーします。

```shell
# lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
NAME                SIZE FSTYPE      MOUNTPOINT
sda                 1.8T             
└─sda1              1.8T exfat       
sdb                 5.5T             
└─sdb1              5.5T exfat       
nvme0n1             1.8T             
├─nvme0n1p1        1007K             
├─nvme0n1p2           1G vfat        /boot/efi
└─nvme0n1p3         1.8T LVM2_member 
  ├─pve-swap          8G swap        [SWAP]
  ├─pve-root         96G ext4        /
  ├─pve-data_tmeta 15.9G             
  │ └─pve-data      1.7T             
  └─pve-data_tdata  1.7T             
    └─pve-data      1.7T
```

```shell
# apt install exfatprogs
Installing:                      
  exfatprogs

Summary:
  Upgrading: 0, Installing: 1, Removing: 0, Not Upgrading: 0
  Download size: 71.2 kB
  Space needed: 356 kB / 90.0 GB available

Get:1 http://deb.debian.org/debian trixie/main amd64 exfatprogs amd64 1.2.9-1+deb13u1 [71.2 kB]
Fetched 71.2 kB in 0s (1,514 kB/s)  
Selecting previously unselected package exfatprogs.
(Reading database ... 60140 files and directories currently installed.)
Preparing to unpack .../exfatprogs_1.2.9-1+deb13u1_amd64.deb ...
Unpacking exfatprogs (1.2.9-1+deb13u1) ...
Setting up exfatprogs (1.2.9-1+deb13u1) ...
Processing triggers for man-db (2.13.1-1) ...
```

```shell
# ls -al /mnt
total 8
drwxr-xr-x  2 root root 4096 Mar  2 20:45 .
drwxr-xr-x 18 root root 4096 Mar  2 20:45 ..
# mkdir -p /mnt/usb6tb
# mount /dev/sdb1 /mnt/usb6tb
# ls -al /mnt/usb6tb/Exported_vhd/lenovo/
total 123411840
drwxr-xr-x 2 root root       131072 Mar  2 19:54 .
drwxr-xr-x 9 root root       131072 Mar  2 06:44 ..
-rwxr-xr-x 1 root root 126373331456 Mar  2 20:08 lenovo.VHDX
```

マウントできました。次にSATA接続HDDをext4でフォーマットします。

```shell
# umount /dev/sda1
umount: /dev/sda1: not mounted.
# wipefs -a /dev/sda
/dev/sda: 2 bytes were erased at offset 0x000001fe (dos): 55 aa
/dev/sda: calling ioctl to re-read partition table: Success
# parted /dev/sda
-bash: parted: command not found
```

partedがないのでインストールします。

```shell
# apt update
Hit:1 http://deb.debian.org/debian trixie InRelease
Hit:2 http://deb.debian.org/debian trixie-updates InRelease
Hit:3 http://security.debian.org/debian-security trixie-security InRelease
Hit:4 http://download.proxmox.com/debian/pve trixie InRelease
All packages are up to date.     
# apt install parted
Installing:                      
  parted

Installing dependencies:
  libparted2t64

Suggested packages:
  libparted-dev  libparted-i18n  parted-doc

Summary:
  Upgrading: 0, Installing: 2, Removing: 0, Not Upgrading: 0
  Download size: 340 kB
  Space needed: 683 kB / 90.0 GB available

Continue? [Y/n] Y
Get:1 http://deb.debian.org/debian trixie/main amd64 libparted2t64 amd64 3.6-5 [300 kB]
Get:2 http://deb.debian.org/debian trixie/main amd64 parted amd64 3.6-5 [40.2 kB]
Fetched 340 kB in 0s (4,138 kB/s) 
Selecting previously unselected package libparted2t64:amd64.
(Reading database ... 60156 files and directories currently installed.)
Preparing to unpack .../libparted2t64_3.6-5_amd64.deb ...
Adding 'diversion of /lib/x86_64-linux-gnu/libparted.so.2 to /lib/x86_64-linux-gnu/libparted.so.2.usr-is-merged by libparted2t64'
Adding 'diversion of /lib/x86_64-linux-gnu/libparted.so.2.0.5 to /lib/x86_64-linux-gnu/libparted.so.2.0.5.usr-is-merged by libparted2t64'
Unpacking libparted2t64:amd64 (3.6-5) ...
Selecting previously unselected package parted.
Preparing to unpack .../parted_3.6-5_amd64.deb ...
Unpacking parted (3.6-5) ...
Setting up libparted2t64:amd64 (3.6-5) ...
Removing 'diversion of /lib/x86_64-linux-gnu/libparted.so.2 to /lib/x86_64-linux-gnu/libparted.so.2.usr-is-merged by libparted2t64'
Removing 'diversion of /lib/x86_64-linux-gnu/libparted.so.2.0.5 to /lib/x86_64-linux-gnu/libparted.so.2.0.5.usr-is-merged by libparted2t64'
Setting up parted (3.6-5) ...
Processing triggers for libc-bin (2.41-12+deb13u1) ...
Processing triggers for man-db (2.13.1-1) ...
```

```shell
# parted /dev/sda
GNU Parted 3.6
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt                                                      
(parted) mkpart primary ext4 0% 100%                                      
(parted) quit                                                             
Information: You may need to update /etc/fstab.

# mkfs.ext4 /dev/sda1                                  
mke2fs 1.47.2 (1-Jan-2025)
/dev/sda1 contains a exfat file system labelled 'ボリューム'
Proceed anyway? (y,N) y
Creating filesystem with 488378368 4k blocks and 122101760 inodes
Filesystem UUID: ########-####-####-####-############
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
	102400000, 214990848

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

SATA接続HDDがext4でフォーマットされたので、そこにVHDXをコピーします。

```shell
# mkdir -p /mnt/hdd2tb
# mount /dev/sda1 /mnt/hdd2tb
# df -h
Filesystem            Size  Used Avail Use% Mounted on
udev                   32G     0   32G   0% /dev
tmpfs                 6.3G  1.4M  6.3G   1% /run
/dev/mapper/pve-root   94G  5.4G   84G   6% /
tmpfs                  32G   46M   32G   1% /dev/shm
efivarfs              320K   71K  245K  23% /sys/firmware/efi/efivars
tmpfs                 5.0M     0  5.0M   0% /run/lock
tmpfs                 1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
tmpfs                  32G     0   32G   0% /tmp
/dev/nvme0n1p2       1022M  8.8M 1014M   1% /boot/efi
/dev/fuse             128M   16K  128M   1% /etc/pve
tmpfs                 1.0M     0  1.0M   0% /run/credentials/getty@tty1.service
tmpfs                 6.3G  4.0K  6.3G   1% /run/user/0
/dev/sdb1             5.5T  1.4T  4.1T  25% /mnt/usb6tb
/dev/sda1             1.8T  2.1M  1.7T   1% /mnt/hdd2tb
```

```shell
# ls -lh /mnt/usb6tb/Exported_vhd/lenovo/
total 118G
-rwxr-xr-x 1 root root 118G Mar  2 20:08 lenovo.VHDX
```

```shell
# cp -v /mnt/usb6tb/Exported_vhd/lenovo/lenovo.VHDX /mnt/hdd2tb/
'/mnt/usb6tb/Exported_vhd/lenovo/lenovo.VHDX' -> '/mnt/hdd2tb/lenovo.VHDX'
```

コピーは結構時間かかります。お風呂に入っていたら完了していました。

次に、VHDXファイルをQCOW2形式に変換します。

```shell
# qemu-img convert -p -f vhdx -O qcow2 \
> /mnt/hdd2tb/lenovo.VHDX \
> /mnt/hdd2tb/lenovo.qcow2
    (0.00/100%)
```

これも時間がかかります。寝て、朝起きたら完了していました。

```shell
# qemu-img convert -p -f vhdx -O qcow2 \
> /mnt/hdd2tb/lenovo.VHDX \
> /mnt/hdd2tb/lenovo.qcow2
    (100.00/100%)
```

# 6. 仮想マシンの作成

```shell
# qm create 100 --name lenovo --memory 8192 --cores 4
# qm set 100 --scsihw virtio-scsi-pci
update VM 100: -scsihw virtio-scsi-pci
# qm set 100 --machine q35
update VM 100: -machine q35
# qm set 100 --bios seabios
update VM 100: -bios seabios
# qm importdisk 100 /mnt/hdd2tb/lenovo.qcow2 local-lvm
importing disk '/mnt/hdd2tb/lenovo.qcow2' to VM 100 ...
  Rounding up size to full physical extent <1.82 TiB
  WARNING: You have not turned on protection against thin pools running out of space.
  WARNING: Set activation/thin_pool_autoextend_threshold below 100 to trigger automatic extension of thin pools before they get full.
  Logical volume "vm-100-disk-0" created.
  WARNING: Sum of all thin volume sizes (<1.82 TiB) exceeds the size of thin pool pve/data and the size of whole volume group (<1.82 TiB).
  Logical volume pve/vm-100-disk-0 changed.
transferred 0.0 B of 1.8 TiB (0.00%)
transferred 18.6 GiB of 1.8 TiB (1.00%)
transferred 37.3 GiB of 1.8 TiB (2.00%)
transferred 55.9 GiB of 1.8 TiB (3.00%)
transferred 74.7 GiB of 1.8 TiB (4.01%)
transferred 93.3 GiB of 1.8 TiB (5.01%)
transferred 112.0 GiB of 1.8 TiB (6.01%)
transferred 130.6 GiB of 1.8 TiB (7.01%)
transferred 149.2 GiB of 1.8 TiB (8.01%)
transferred 167.9 GiB of 1.8 TiB (9.01%)
transferred 186.5 GiB of 1.8 TiB (10.01%)
transferred 205.3 GiB of 1.8 TiB (11.02%)
transferred 223.9 GiB of 1.8 TiB (12.02%)
transferred 242.6 GiB of 1.8 TiB (13.02%)
transferred 261.2 GiB of 1.8 TiB (14.02%)
transferred 279.8 GiB of 1.8 TiB (15.02%)
transferred 298.5 GiB of 1.8 TiB (16.02%)
transferred 317.1 GiB of 1.8 TiB (17.02%)
transferred 335.7 GiB of 1.8 TiB (18.02%)
transferred 354.5 GiB of 1.8 TiB (19.03%)
transferred 373.2 GiB of 1.8 TiB (20.03%)
transferred 391.8 GiB of 1.8 TiB (21.03%)
transferred 410.4 GiB of 1.8 TiB (22.03%)
transferred 429.1 GiB of 1.8 TiB (23.03%)
transferred 447.7 GiB of 1.8 TiB (24.03%)
transferred 466.3 GiB of 1.8 TiB (25.03%)
transferred 484.9 GiB of 1.8 TiB (26.03%)
transferred 503.6 GiB of 1.8 TiB (27.03%)
transferred 522.2 GiB of 1.8 TiB (28.03%)
transferred 540.8 GiB of 1.8 TiB (29.03%)
transferred 559.7 GiB of 1.8 TiB (30.04%)
transferred 578.3 GiB of 1.8 TiB (31.04%)
transferred 596.9 GiB of 1.8 TiB (32.04%)
transferred 615.5 GiB of 1.8 TiB (33.04%)
transferred 634.2 GiB of 1.8 TiB (34.04%)
transferred 652.8 GiB of 1.8 TiB (35.04%)
transferred 671.4 GiB of 1.8 TiB (36.04%)
transferred 690.1 GiB of 1.8 TiB (37.04%)
transferred 708.7 GiB of 1.8 TiB (38.04%)
transferred 727.5 GiB of 1.8 TiB (39.05%)
transferred 746.1 GiB of 1.8 TiB (40.05%)
transferred 764.8 GiB of 1.8 TiB (41.05%)
transferred 783.4 GiB of 1.8 TiB (42.05%)
transferred 802.0 GiB of 1.8 TiB (43.05%)
transferred 820.7 GiB of 1.8 TiB (44.05%)
transferred 839.3 GiB of 1.8 TiB (45.05%)
transferred 857.9 GiB of 1.8 TiB (46.05%)
transferred 876.5 GiB of 1.8 TiB (47.05%)
transferred 895.2 GiB of 1.8 TiB (48.05%)
transferred 913.8 GiB of 1.8 TiB (49.05%)
transferred 932.4 GiB of 1.8 TiB (50.05%)
transferred 951.1 GiB of 1.8 TiB (51.05%)
transferred 969.7 GiB of 1.8 TiB (52.05%)
transferred 988.3 GiB of 1.8 TiB (53.05%)
transferred 1007.0 GiB of 1.8 TiB (54.05%)
transferred 1.0 TiB of 1.8 TiB (55.05%)
transferred 1.0 TiB of 1.8 TiB (56.05%)
transferred 1.0 TiB of 1.8 TiB (57.05%)
transferred 1.1 TiB of 1.8 TiB (58.05%)
transferred 1.1 TiB of 1.8 TiB (59.06%)
transferred 1.1 TiB of 1.8 TiB (60.06%)
transferred 1.1 TiB of 1.8 TiB (61.06%)
transferred 1.1 TiB of 1.8 TiB (62.06%)
transferred 1.1 TiB of 1.8 TiB (63.06%)
transferred 1.2 TiB of 1.8 TiB (64.06%)
transferred 1.2 TiB of 1.8 TiB (65.06%)
transferred 1.2 TiB of 1.8 TiB (66.06%)
transferred 1.2 TiB of 1.8 TiB (67.06%)
transferred 1.2 TiB of 1.8 TiB (68.06%)
transferred 1.3 TiB of 1.8 TiB (69.06%)
transferred 1.3 TiB of 1.8 TiB (70.06%)
transferred 1.3 TiB of 1.8 TiB (71.06%)
transferred 1.3 TiB of 1.8 TiB (72.07%)
transferred 1.3 TiB of 1.8 TiB (73.07%)
transferred 1.3 TiB of 1.8 TiB (74.07%)
transferred 1.4 TiB of 1.8 TiB (75.07%)
transferred 1.4 TiB of 1.8 TiB (76.07%)
transferred 1.4 TiB of 1.8 TiB (77.07%)
transferred 1.4 TiB of 1.8 TiB (78.07%)
transferred 1.4 TiB of 1.8 TiB (79.07%)
transferred 1.5 TiB of 1.8 TiB (80.07%)
transferred 1.5 TiB of 1.8 TiB (81.07%)
transferred 1.5 TiB of 1.8 TiB (82.07%)
transferred 1.5 TiB of 1.8 TiB (83.08%)
transferred 1.5 TiB of 1.8 TiB (84.08%)
transferred 1.5 TiB of 1.8 TiB (85.08%)
transferred 1.6 TiB of 1.8 TiB (86.08%)
transferred 1.6 TiB of 1.8 TiB (87.08%)
transferred 1.6 TiB of 1.8 TiB (88.08%)
transferred 1.6 TiB of 1.8 TiB (89.08%)
transferred 1.6 TiB of 1.8 TiB (90.08%)
transferred 1.7 TiB of 1.8 TiB (91.09%)
transferred 1.7 TiB of 1.8 TiB (92.09%)
transferred 1.7 TiB of 1.8 TiB (93.09%)
transferred 1.7 TiB of 1.8 TiB (94.09%)
transferred 1.7 TiB of 1.8 TiB (95.09%)
transferred 1.7 TiB of 1.8 TiB (96.09%)
transferred 1.8 TiB of 1.8 TiB (97.10%)
transferred 1.8 TiB of 1.8 TiB (98.10%)
transferred 1.8 TiB of 1.8 TiB (99.10%)
transferred 1.8 TiB of 1.8 TiB (100.00%)
transferred 1.8 TiB of 1.8 TiB (100.00%)
unused0: successfully imported disk 'local-lvm:vm-100-disk-0'
# qm set 100 --scsi0 local-lvm:vm-100-disk-0
update VM 100: -scsi0 local-lvm:vm-100-disk-0
# qm set 100 --boot order=scsi0
update VM 100: -boot order=scsi0
```

これで仮想マシンの準備ができました。

# 7. 仮想マシンの起動

Web UIから仮想マシンを起動させます。仮想マシンの状態はコンソールから確認できます。起動できなかった場合は次のことを確認してみてください。

- 仮想マシンのCPUがhostになっていること。
  - 限定されたエミュレーションだと動かないことがあります。
- 特にWinXPなどの古いOSの場合、仮想マシンのHDDをIDE接続に変更する。
  - Hyper-Vでも同様のブルスク事例があった。
- ネットワークカードをE1000Eにする。
  - 仮想マシン起動後にPINコードを再設定する必要があったのですが、そのためにMicrosoftアカウントでログインする必要があり、それにはネットワーク接続が必要になりました。E1000Eだと繋がりました。
  - WinXPなどの古いOSの場合は、Realtekでつながりました。

## 例: Win11

```shell
# qm config 100
agent: 1
bios: ovmf
boot: order=sata0
cores: 6
cpu: host
efidisk0: local-lvm:vm-100-disk-3,pre-enrolled-keys=1,size=4M
machine: pc-q35-10.1
memory: 8192
meta: creation-qemu=10.1.2,ctime=1772494934
name: win11
net0: e1000e=BC:24:11:C8:B8:7D,bridge=vmbr0,firewall=1
numa: 0
ostype: win11
sata0: local-lvm:vm-100-disk-0,size=1907732M
sata1: local:iso/virtio-win-0.1.285.iso,media=cdrom,size=771138K
scsihw: virtio-scsi-pci
smbios1: uuid=########-####-####-####-############
sockets: 1
tpmstate0: local-lvm:vm-100-disk-2,size=4M,version=v2.0
vmgenid: ########-####-####-####-############
```

## 例: WinXP(32bit)

```shell
# qm config 101
agent: 1
audio0: device=AC97,driver=none
boot: order=ide0;ide2
cores: 1
cpu: host
ide0: local-lvm:vm-101-disk-0,size=76320M
ide2: local:iso/virtio-win-0.1.130.iso,media=cdrom,size=163080K
machine: pc-i440fx-10.1
memory: 2048
meta: creation-qemu=10.1.2,ctime=1772517622
name: winxp
net0: rtl8139=BC:24:11:43:5B:7A,bridge=vmbr0,firewall=1,link_down=1
numa: 0
ostype: wxp
smbios1: uuid=########-####-####-####-############
sockets: 1
vmgenid: ########-####-####-####-############
```

# (オプション) クライアントエージェントのインストール

https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers

クラウアイとエージェントをインストールすることで、安全なシャットダウンやIPアドレスの検知ができるようになります。ただし、WinXPなどの古いOSの場合は対応していないことがあります。

Linuxの場合はaptでインストールします。TrueNASとかだとすでに入っていることもあります。

```shell
# apt install qemu-guest-agent
```

インストールできたら有効化します。

```shell
# systemctl enable qemu-guest-agent
# systemctl start qemu-guest-agent
```

# (オプション) ファイアウォールの設定

古いOSの場合はネットに繋がないようにファイアウォールの設定をしておきましょう。

# (オプション) 遠隔操作の方法

ProxmoxはNoVNCを使って仮想マシンを遠隔操作できますが、これだと音が出ません。RDP接続するとマルチメディア機能も使えるのですが、Windows特にHome版だとRDPが使えないので、昔のUltraVNCとかを入れておくと、便利かもしれません。

# まとめ

- Windows + Hyper-Vから、Proxmoxに移行しました。
- CPUやメモリを換装してスペックを余らせていたので、それが活かせそうでワクワクしています。
