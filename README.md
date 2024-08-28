# BuildingWSLEmbeddedLinux
Building an Embedded Linux Environment Using WSL


[【特別企画】～i.MX8で学ぶ～ 組み込みLinuxハンズオン・セミナ](https://interface.cqpub.co.jp/linux-hands-on/)

Linuxデバイス・ドライバ開発入門 環境構築情報

## 準備

- Windows PC (Windows 11 または Windows 10 x64) 各最新版
- WSL2 Ubuntu-20.04 (Ubuntu 20.04.6 LTS) インストール

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

## 手順（その１）

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

### 基本設定

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

### ソースコード入手

```sh
mkdir ~/imx-yocto-bsp
cd ~/imx-yocto-bsp
repo init -u https://github.com/nxp-imx/imx-manifest -b imx-linux-mickledore -m imx-6.1.22-2.0.0.xml
```

### 同期

```sh
$ sync
```

6分ぐらい時間がかかります。

```sh
$ cd ~/imx-yocto-bsp/sources
$ git clone https://github.com/Avnet/meta-maaxboard.git -b mickledore meta-maaxboard
```

### ビルド

```sh
$ cd ~/imx-yocto-bsp
$ MACHINE=maaxboard-8ulp source sources/meta-maaxboard/tools/maaxboard-setup.sh -b maaxboard-8ulp/build
```

★これの実行でconfが出来て、ディレクトリが変わる。

// train@YOKO:~/imx-yocto-bsp/maaxboard-8ulp/build$ 

ここで conf を編集して以下を追加

```sh
cat >> conf/local.conf
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"
```

//~/imx-yocto-bsp/maaxboard-8ulp/build/conf/local.conf

//emacs conf/local.conf

```sh
$ cd ~/imx-yocto-bsp
$ source sources/poky/oe-init-build-env maaxboard-8ulp/build
```

//$ bitbake avnet-image-full

ではなくて！

```sh
$ bitbake avnet-image-minimal
time bitbake avnet-image-minimal
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
