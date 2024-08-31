# BuildingWSLEmbeddedLinux
Building an Embedded Linux Environment Using WSL


[【特別企画】～i.MX8で学ぶ～ 組み込みLinuxハンズオン・セミナ](https://interface.cqpub.co.jp/linux-hands-on/)

Linuxデバイス・ドライバ開発入門 環境構築方法の解説

WSL2 Ubuntu-20.04 を使用する デバイス・ドライバ開発ハンズオンセミナー配布用のWSL2イメージを作成する手順を示します。

これはハンズオンセミナー受講者用の解説ではありません。
受講者用のセミナー環境を準備するための環境構築に必要な手順の説明です。

## 準備

- Windows PC (Windows 11 または Windows 10 x64) 各最新版
- WSL2 Ubuntu-20.04 (Ubuntu 20.04.6 LTS) インストール

コマンドプロンプトで以下を実行して WSL2 をインストールします。
WSL2インストール前に、Cドライブに 200GB 程度の空き容量が必要です。
各ビルド作業に時間がかかるため高速、高性能な環境が必要です。
以下を参考にして環境を選択します。

## ハードウェア環境

環境A：Core i5 6600 / 4コア 4スレッド / 16GB / SSD : bitbake エラー発生
環境B：Ryzen7 5700X / 8コア 16スレッド / 32GB / M.2 Gen3x4  : bitbake 30分程度で終了

### WSL環境のインストールと構築

```cmd
wsl -l -v
wsl --list --online

wsl --update
wsl --instal Ubuntu-20.04
wsl --set-default-version 2
wsl --set-default Ubuntu-20.04
```

- アカウント作成
  - Ubuntu 標準アカウント名
    train/cq

  - 予備のアカウント
    maaxboard/avnet

```sh
# adduser maaxboard
# passwd maaxboard
# usermod -aG sudo maaxboard
```

## Maaxboard Yocto Project環境構築

Yocto Project環境の入手とインストール、環境ビルド

### 環境設定

```sh
$ sudo apt update
$ sudo apt upgrade
```

必然なエディタと作業用ツールのインストール

```sh
$ sudo apt update
$ sudo apt install -y emacs-nox
$ sudo apt install -y net-tools
```

（ほかにある場合はここに追記）

### ツールのインストール

```sh
$ sudo apt install -y wget git-core diffstat unzip texinfo gcc-multilib \
build-essential chrpath socat cpio python python3 python3-pip python3-pexpect \
xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev \
pylint3 xterm rsync curl gawk zstd lz4 locales bash-completion
```

以下は Yocto-Development-Guide-V3.1 に非掲載のため、分けて追記。

```sh
$ sudo apt install -y make build-essential libncurses-dev bison flex libssl-dev libelf-dev
```

### Yocto 基本設定

```sh
$ cd
$ mkdir -p ~/bin
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
$ export PATH=~/bin:$PATH
```

```sh
git config --global user.name username
git config --global user.email username@domain.name
```

### Yocto ソースコード入手

```sh
$ mkdir ~/imx-yocto-bsp
$ cd ~/imx-yocto-bsp
$ repo init -u https://github.com/nxp-imx/imx-manifest -b imx-linux-mickledore -m imx-6.1.22-2.0.0.xml
```

### 同期

```sh
$ repo sync
```

6分ぐらい時間がかかります。

```sh
$ cd ~/imx-yocto-bsp/sources
$ git clone https://github.com/Avnet/meta-maaxboard.git -b mickledore meta-maaxboard
```

### ビルド前

```sh
$ cd ~/imx-yocto-bsp
$ MACHINE=maaxboard-8ulp source sources/meta-maaxboard/tools/maaxboard-setup.sh -b maaxboard-8ulp/build
```

★これの実行でconfが出来て、ディレクトリが変わる。

```sh
＄pwd

＄ls ~/imx-yocto-bsp/maaxboard-8ulp/build
```

このタイミングで local.conf を編集して以下を追加

```sh
$ cat >> conf/local.conf
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"
BB_SERVER_TIMEOUT = "600"
```

### local.conf の確認

```sh
cat ~/imx-yocto-bsp/maaxboard-8ulp/build/conf/local.conf
//emacs conf/local.conf
```

### ビルド

```sh
$ cd ~/imx-yocto-bsp
$ source sources/poky/oe-init-build-env maaxboard-8ulp/build
```


```sh
time bitbake core-image-minimal
```

#### 注意事項

```sh
$ bitbake avnet-image-full
```
の手順は容量が大き過ぎて時間がかかる

```sh
$ bitbake avnet-image-minimal
```
はエラー発生

## カーネルビルド

クロスコンパイラとカーネルソースを入手してカーネルをビルド

### aarch64 クロスコンパイラの入手

https://developer.arm.com/downloads/-/gnu-a

から、**gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz** を入手
ホームディレクトリに保存しておきます。名前を間違えないように十分注意が必要です。

//
$ mkdir ~/toolchain
$ tar -xJf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz -C ~/toolchain


$ cd toolchain/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
$ ./aarch64-none-linux-gnu-gcc -v

gcc version 10.3.1 20210621 (GNU Toolchain for the A-profile Architecture 10.3-2021.07 (arm-10.29))
の表示を確認

ここから
$ TOOLCHAIN_PATH=$HOME/toolchain/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin
$ export PATH=$TOOLCHAIN_PATH:$PATH
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-none-linux-gnu-

$ cd ~/imx-yocto-bsp
$ source sources/poky/oe-init-build-env maaxboard-8ulp/build

//$ bitbake avnet-image-full -c populate_sdk
//ではなくて
bitbake core-image-minimal -c populate_sdk

エラーが出るのでshスクリプトは
飛ばして

★ここから！
2.3 Build Kernel in a standalone environment

から実行すれば、行くはず！？
以下。

cd

git clone https://github.com/Avnet/linux-imx.git -b maaxboard_lf-6.1.22-2.0.0
$ echo $CROSS_COMPILE $ARCH

aarch64-none-linux-gnu- arm64

$ cd linux-imx
$ make distclean
$ make maaxboard-8ulp_defconfig
$ make -j4

Execute the ‘ls’ command to view the Image and dtb files after compilation.

$ ls arch/arm64/boot/Image
$ ls arch/arm64/boot/dts/freescale/maaxboard*dtb
arch/arm64/boot/dts/freescale/maaxboard-8ulp.dtb

train@Rie:~/linux-imx$ ls arch/arm64/boot/dts/freescale/maaxboard*dtb
arch/arm64/boot/dts/freescale/maaxboard-8ulp.dtb  arch/arm64/boot/dts/freescale/maaxboard-mini.dtb  arch/arm64/boot/dts/freescale/maaxboard.dtb
train@Rie:~/linux-imx$

$ make modules
$ make modules_install INSTALL_MOD_PATH=./rootfs

完了！

////

## 環境検証

作成したイメージを実際にインポートして動作検証をします。
受講者が事前準備で行う作業と同じ内容です。

### イメージのインポートの実行

Cドライブに213GBの空きがある状況でどうなるかを確認します。

あらかじめ train01.zip (43.6 GB) の圧縮イメージをダウンロードして入手、展開して train01.tar (120.7 GB)を用意しておきます。

次に、コマンドを実行してハンズオン用 WSLイメージを展開します。

```cmd
> wsl --import Ubuntu-20.04cq "%LOCALAPPDATA%\CQHandsOn-01" "C:\WSL\train01.tar"
```

### デフォルトユーザーの設定

インポート完了後、デフォルトユーザーを設定します。

- WSL デフォルト設定

```cmd
> wsl
```

- WSLを起動

```cmd
> wsl
```

- 


```sh
# cat >> /etc/wsl.conf
```

```sh
[user]
default=train
```

```sh
# cat /wsl.conf
```

```sh
# shutdown -r now
```

再起動コマンド実行後、15秒程度待って、wslを起動してログインして動作確認します。

```cmd
> wsl
```

## 資料

[MaaXBoard 8ULP Starter Kit](https://www.avnet.com/wps/portal/us/products/avnet-boards/avnet-board-families/maaxboard/maaxboard-8ulp/)

[MaaXBoard-8ULP-Linux-Yocto-Development-Guide-V3.1](https://www.avnet.com/wps/wcm/connect/onesite/4fa62a19-239c-40c9-aff6-8a122f993f1e/MaaXBoard-8ULP-Linux-Yocto-Development-Guide-V3.1.pdf?MOD=AJPERES&CACHEID=ROOTWORKSPACE.Z18_NA5A1I41L0ICD0ABNDMDDG0000-4fa62a19-239c-40c9-aff6-8a122f993f1e-oKQ-1ib)

[MaaXBoard-8ULP-Linux-Yocto-UserManual-V3.1](https://www.avnet.com/wps/wcm/connect/onesite/07e9ad99-8969-40c1-b632-db97adf350d0/MaaXBoard-8ULP-Linux-Yocto-UserManual-V3.1.pdf?MOD=AJPERES&CACHEID=ROOTWORKSPACE.Z18_NA5A1I41L0ICD0ABNDMDDG0000-07e9ad99-8969-40c1-b632-db97adf350d0-oKQ-2Mr)


#### GitHub

https://github.com/Avnet/MaaXBoard-8ULP-HUB

#### 独自作成

[MaaXBoard-8ULP-Linux-Yocto-Development-Guide-V3.1
の問題点](IssuesOfDevelopmentGuide.md)

[WSL Video](https://www.youtube.com/watch?v=S78OAxQjPoA)

[WSL Sline](https://www.slideshare.net/slideshow/tips-and-tricks-for-wsl-users-two-easy-and-reliable-ways-to-get-started-with-openssl-server/271345097)


https://github.com/ahidaka/YoctoTrainingRpi

https://github.com/ahidaka/HelloLinuxDriver
