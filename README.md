# ラズパイ用カーネルをQEMUで起動する手順

## 最終更新日

2019/5/6

## 動作環境

- Ubuntu 19.04(on Windows10 1804 Hyper-V)
- qemu-system-arm 3.1.0

## 事前準備

ルートファイルシステム(イメージファイル)は同梱していないので、ラズパイ本家サイトから入手しておきます。

- https://www.raspberrypi.org/downloads/raspbian/
- http://ftp.jaist.ac.jp/pub/raspberrypi/raspbian_lite/images/

イメージファイルは「2019-04-08-raspbian-stretch-lite.img」(1.7GB, kernel 4.14.98+向け)のようなファイルです。

カーネルとイメージファイルのカーネルモジュールは、ビルドされたカーネルバージョンを合わせておく必要がありますが、
本サイトで配布しているカーネルはドライバを静的リンクしているため、実際にはカーネルモジュールとカーネルバージョンが
合っていなくても起動できます。

## 必要なパッケージ

- git
- qemu-system-arm

## QEMUからのLinuxカーネル起動方法

qemu-system-armコマンドはGNOME(X Window System)の端末から実行します。SSH端末上では実行できません。

Linuxカーネル(zImage)、Device Tree(versatile-pb.dtb)、イメージファイルをカレントディレクトリに格納して、下記コマンドを実行します。

```
# qemu-system-arm \
-kernel zImage \
-cpu arm1176 \
-M versatilepb \
-dtb versatile-pb.dtb \
-m 256 \
-no-reboot \
-serial stdio \
-append "root=/dev/sda2 panic=1 rootfstype=ext4 rw" \
-hda 2019-04-08-raspbian-stretch-lite.img \
-net nic -net user,hostfwd=tcp::10022-:22
```

## カーネルのビルド方法

クロスコンパイルでLinuxカーネルをビルドします。
下記手順にしたがってtoolchainを導入します。
- https://www.raspberrypi.org/documentation/linux/kernel/building.md
```
# git clone https://github.com/raspberrypi/tools ~/tools
↓.bashrcでパスを通す
export PATH=$PATH:~/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin
```

5.0カーネルをクローンします。
```
# git clone --depth=1 -b rpi-5.0.y https://github.com/raspberrypi/linux linux5.0
```

Linuxカーネルに「linux5.0.diff」をパッチします。
```
# patch --dry-run -p0 < linux5.0.diff
# patch  -p0 < linux5.0.diff
```

次に下記コマンドを実行します。

```
# linux5.0
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- versatile_defconfig
```
同梱の「config_file_5.0」を「.config」に上書きします。
```
# mv config_file_5.0 .config
```

カーネルとDevic Treeをビルドします。

```
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bzImage dtbs
```

ビルドが成功したら、下記ファイルを取得します。

- arch/arm/boot/zImage
- arch/arm/boot/dts/versatile-pb.dtb


## Appendix: イメージファイルのマウント方法

```
# fdisk -l 2019-04-08-raspbian-stretch-lite.img
Disk 2019-04-08-raspbian-stretch-lite.img: 1.7 GiB, 1803550720 bytes, 3522560 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xc1dc39e5

Device                                Boot Start     End Sectors  Size Id Type
2019-04-08-raspbian-stretch-lite.img1       8192   96042   87851 42.9M  c W95 FAT32 (LBA)
2019-04-08-raspbian-stretch-lite.img2      98304 3522559 3424256  1.6G 83 Linux
```

```
# sudo mount -v -o offset=$((512*98304)) -t ext4 ./2019-04-08-raspbian-stretch-lite.img /mnt
# sudo umount /mnt
```
