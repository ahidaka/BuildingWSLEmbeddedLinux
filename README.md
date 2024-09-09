# BuildingWSLEmbeddedLinux
Building an Embedded Linux Environment Using WSL


[【特別企画】～i.MX8で学ぶ～ 組み込みLinuxハンズオン・セミナ](https://interface.cqpub.co.jp/linux-hands-on/)

Linuxデバイス・ドライバ開発入門 環境構築方法の解説

WSL2 Ubuntu-20.04 を使用する デバイス・ドライバ開発ハンズオンセミナ配布用のWSL2イメージを作成する手順を示します。

これはハンズオンセミナ受講者用の解説ではありません。
受講者用のセミナ環境を準備するための環境構築に必要な手順の説明です。

## 準備

- Windows PC (Windows 11 または Windows 10 x64) 各最新版
- WSL2 Ubuntu-20.04 (Ubuntu 20.04.6 LTS) インストール

コマンドプロンプトで以下を実行して WSL2 をインストールします。
WSL2インストール前に、Cドライブに 200GB 程度の空き容量が必要です。
各ビルド作業に時間がかかるため高速、高性能な環境が必要です。
以下を参考にして環境を選択します。

## ハードウェア環境

環境A：Core i5 6600 / 4コア 4スレッド / 16GB / SSD : bitbake 開始後直ぐに、エラー発生
環境B：Ryzen7 5700X / 8コア 16スレッド / 32GB / M.2 Gen3x4  : bitbake 30分程度で終了
環境C：Core i3 4160 / 4コア 4スレッド / 16GB / M.2 Gen3x4 : bitbake 282分後 エラー発生
環境B：Ryzen5 5600G / 6コア 12スレッド / 16GB / M.2 Gen3x4  : bitbake 84分程度で終了

### WSL環境のインストールと構築

```cmd
> wsl -l -v
> wsl --list --online

> wsl --update
> wsl --install Ubuntu-20.04
> wsl --set-default-version 2
> wsl --set-default Ubuntu-20.04
```

- アカウント作成
  - Ubuntu 標準アカウント名
    train/cq

  - 予備のアカウント追加
    maaxboard/avnet

```sh
$ sudo -i
# adduser maaxboard
  passwd? avnet
  以下 Enter
  Is the information correct? [Y/n] Y
# usermod -aG sudo maaxboard
```

- パスワード無し設定

  **train  ALL=NOPASSWD: ALL** 行を追加します。
```sh
$ sudo -i
# visudo
...
（一部省略）
%sudo   ALL=(ALL:ALL) ALL
train  ALL=NOPASSWD: ALL

# See sudoers(5) for more information on "#include" directives:
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

（ほかにインストールがある場合はここに追記）

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

GitHub アカウント登録です。ご自身のアカウントで設定してください。

```sh
$ git config --global user.name username
$ git config --global user.email username@domain.name
```

### Yocto Project ソースコード入手

```sh
$ mkdir ~/imx-yocto-bsp
$ cd ~/imx-yocto-bsp
$ repo init -u https://github.com/nxp-imx/imx-manifest -b imx-linux-mickledore -m imx-6.1.22-2.0.0.xml
...
一部省略
Enable color display in this user account (y/N)? Y
```

### 同期

```sh
$ repo sync
```

5～6分ぐらい時間がかかります。

```sh
$ cd ~/imx-yocto-bsp/sources
$ git clone https://github.com/Avnet/meta-maaxboard.git -b mickledore meta-maaxboard
```

### ビルド前環境設定

```sh
$ cd ~/imx-yocto-bsp
$ MACHINE=maaxboard-8ulp source sources/meta-maaxboard/tools/maaxboard-setup.sh -b maaxboard-8ulp/build
```

この「source」コマンド実行で、confが出来てディレクトリが変わります。
念のため現在のディレクトリを確認します。

```sh
＄pwd

＄ls ~/imx-yocto-bsp/maaxboard-8ulp/build
```

このタイミング local.conf を編集して以下のYoctoビルド用パラメータを追加します。

```sh
$ cat >> conf/local.conf
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"
BB_SERVER_TIMEOUT = "600"
```

### local.conf の確認

念のため、パラメータ追加編集したlocal.conf 全体を確認します。
必要があればエディタ（以下の場合はemacs）で編集します

```sh
$ cat ~/imx-yocto-bsp/maaxboard-8ulp/build/conf/local.conf
必要な場合は編集： emacs conf/local.conf
```

### ビルド

bitbake コマンドでYocto Project をビルドします。
時間がかかるので、time コマンドで計測します。

```sh
$ time bitbake core-image-minimal
```

#### 補足

参考情報：
```sh
//$ bitbake avnet-image-full
```

の手順は容量が大き過ぎて時間がかかります。
```sh
//$ bitbake avnet-image-minimal
```

はエラー発生で使えません。

<br/>

## カーネルビルド

クロスコンパイラとカーネルソースを入手してカーネルをビルド

### aarch64 クロスコンパイラの入手

 Yocto-Development-Guide-V3.1 に記載された gcc コンパイラ version 10.3 ではバージョンが古く不具合があるため
 version 12.2 をインストールします。入手場所とパッケージ名（tarとディレクトリ名）が異なるので注意が必要です。

https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads

から、
```
What's new in 12.2.Rel1
This release is based on GCC 12.2

x86_64 Linux hosted cross toolchains
AArch64 GNU/Linux target (aarch64-none-linux-gnu)
```
**arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-linux-gnu.tar.xz** を入手して、
ホームディレクトリに保存しておきます。tar.xz ファイルが多数あるため、名前を間違えないように十分注意が必要です。

以下の手順でダウンロードした tar.xz ファイルを展開します。

```sh
$ cd
$ mkdir ~/toolchain
$ tar -xJf arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-linux-gnu.tar.xz -C ~/toolchain
```

念のためコンパイラのバージョン表示を確認します。

```sh
$ cd ~/toolchain/arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-linux-gnu/bin/
$ ./aarch64-none-linux-gnu-gcc -v
```

```sh
...
一部省略
...
in for the A-profile Architecture 10.3-2021.07 (arm-10.29)'
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 12.2.1 20221205 (Arm GNU Toolchain 12.2.Rel1 (Build arm-12.24))
```

と表示されることを確認します。

#### SDK

SDK ビルド環境の設定
```sh
$ 
$ TOOLCHAIN_PATH=$HOME/toolchain/arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-linux-gnu/bin
$ export PATH=$TOOLCHAIN_PATH:$PATH
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-none-linux-gnu-

$ cd ~/imx-yocto-bsp
$ source sources/poky/oe-init-build-env maaxboard-8ulp/build
```

SDK のビルド。時間がかかります。
```sh
$ bitbake core-image-minimal -c populate_sdk
```

完了後以下の
```
~/imx-yocto-bsp/maaxboard-8ulp/build/tmp/deploy/sdk/
fsl-imx-wayland-lite-glibc-x86_64-core-image-minimal-armv8a-maaxboard-8ulp-toolchain-6.1-mickledore.sh
```

が作成されるので、これを実行します。
```sh
$ sudo ~/imx-yocto-bsp/maaxboard-8ulp/build/tmp/deploy/sdk/fsl-imx-wayland-lite-glibc-x86_64-core-image-minimal-armv8a-maaxboard-8ulp-toolchain-6.1-mickledore.sh
```

少し時間がかかり、途中でPATH設定の確認を求められます。完了後は、
```
SDK has been successfully set up and is ready to be used.

Each time you wish to use the SDK in a new shell session, you need to source the environment setup script e.g.
 $ . /opt/fsl-imx-wayland-lite/6.1-mickledore/environment-setup-armv8a-poky-linux
```

のメッセージを確認してSDKのビルドは完了です。

この場で、実際に実行しておきます。
```sh
$ . /opt/fsl-imx-wayland-lite/6.1-mickledore/environment-setup-armv8a-poky-linux
```

### カーネルビルド

ホームディレクトリに戻ってから作業します。
```sh
$ cd
```

カーネルツリーをcloneします。
```sh
$ git clone https://github.com/Avnet/linux-imx.git -b maaxboard_lf-6.1.22-2.0.0
```

ビルド実行前のパスの確認です。
```sh
$ echo $CROSS_COMPILE $ARCH
```

以下の様に表示されます。
```sh
aarch64-none-linux-gnu- arm64
```

カーネルのビルド実行。時間がかかります。
```sh
$ cd linux-imx
$ make distclean
$ make maaxboard-8ulp_defconfig
$ make -j4
```

次のメッセージを確認して終了します。
```sh
Execute the ‘ls’ command to view the Image and dtb files after compilation.
```

ビルドしたファイルを確認します。
```sh
$ ls arch/arm64/boot/Image
$ ls arch/arm64/boot/dts/freescale/maaxboard*dtb
```

次のファイルが表示されます。
```sh
arch/arm64/boot/dts/freescale/maaxboard-8ulp.dtb
arch/arm64/boot/dts/freescale/maaxboard-mini.dtb
arch/arm64/boot/dts/freescale/maaxboard.dtb
```

モジュールのビルドを実行します。
```sh
$ make modules
$ make modules_install INSTALL_MOD_PATH=./rootfs
```

これでハンズオン用WSL環境の作成作業は完了です。
<br/>

## 配布用ファイルイメージの作成

WSLを停止後、仮想ディスクをエクスポート、さらにそれを圧縮してハンズオンで使用する配布用ファイルを作成します。

### 全てのWSLを停止

まず、全てのWSLからログアウトして、コマンドプロンプトで状況を確認します。次の様に **Running** であれば稼働中です。
```cmd
> C:\Users\Train>wsl -l -v

  NAME            STATE           VERSION
* Ubuntu-20.04    Running         2
```

shutdiwn コマンドで全てのWSLを停止します。
```cmd
> wsl --shutdown
```

再度コマンドプロンプトで状況を確認します。
```cmd
> wsl -l -v

  NAME            STATE           VERSION
* Ubuntu-20.04    Stopped         2
```

**Stopped** で、停止を確認しました。

### WSL仮想ディスクのエクスポート

**export** コマンドで、他の Windows PC に WSL仮想ディスクを移送するための tar ファイルを作成します。
例として、D:\WSL\train01.tar に仮想ディスクイメージを作成する場合は、次の様なコマンドを実行します。
```cmd
> wsl --export Ubuntu-20.04 "D:\WSL\train01.tar"
```

エクスポートする仮想ディスクの tarファイルは、100GB 程度の容量です。少し時間がかかります。

エクスポート完了後は、エクスプローラーをで作成した C:\WSL\train01.tar ファイルを zip ファイルに圧縮します。
ファイル拡張子が変更されて、train01.zip と変わります。37.1GB の容量です。
WSL配布用ファイルの作成は、これで完了です。
<br/>

## 環境検証

作成した配布用 zip ファイルを実際にインポートして、動作検証をします。
以降は、受講者が事前準備で行う作業と同じ内容です。

### イメージのインポートの実行

Cドライブには、別途用意するインストール用ダウンロードイメージとは別に、インストール用の150GB 程度の空き領域が必要です。

あらかじめ train01.zip (37.1 GB) の圧縮イメージをダウンロードして入手、展開して train01.tar (120.7 GB)を用意しておきます。

Cドライブに必要な 150GB の領域はこれらに含まれません。

次に、コマンドを実行してハンズオン用 WSLイメージを展開します。

```cmd
> wsl --import Ubuntu-20.04cq "%LOCALAPPDATA%\CQHandsOn-01" "D:\WSL\train01.tar"
```

### デフォルトユーザーの設定

インポート完了後、デフォルトユーザーを設定します。

#### WSL デフォルト設定

現在の状態を確認します。

```cmd
> wsl -l -v
```
結果の表示例

```cmd
  NAME              STATE           VERSION
  Ubuntu-20.04cq    Running         2
* Ubuntu-20.04      Stopped         2
```

上記の様にUbuntu-20.04cq がインストール済でStopped 状態を確認して、デフォルト設定します。

```cmd
> wsl --set-default Ubuntu-20.04cq
```

#### WSL起動とデフォルトユーザー設定

WSLを起動してログイン

```cmd
> wsl
```
- 
以下の手順で　/etc/wsl.conf にデフォルトユーザーを設定します。

```sh
# cat >> /etc/wsl.conf
```

入力内容

```sh
[user]
default=train
```

wsl.conf 内容確認

```sh
# cat /wsl.conf
```
次の内容になっていることを確認します。

```sh
[boot]
systemd=true
[user]
default=train
```

再起動

```sh
# shutdown -r now
```

再起動コマンド実行後、15秒程度待って、wslを起動してログインして動作確認します。

```cmd
> wsl
```
### 動作確認

インポート環境の動作を確認します。

#### ネットワークと最新状態の確認

次のコマンドで確認します。

```cmd
$ sudo apt update
$ sudo apt upgrade
```

必要に応じて'Y'入力して更新します。

#### クロスコンパイラの状態の確認

次のコマンドで確認します。

```sh
$ cd ~/toolchain/arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-linux-gnu/bin/
$ ./aarch64-none-linux-gnu-gcc -v
```
を実行して次の様に表示されることを確認します。

```sh
...
一部省略
...
in for the A-profile Architecture 10.3-2021.07 (arm-10.29)'
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 12.2.1 20221205 (Arm GNU Toolchain 12.2.Rel1 (Build arm-12.24))
```

## ハンズオン環境の動作確認

ここまで実行して、最後の **aarch64-none-linux-gnu-gcc -v** の表示の確認をしてください。
異常がある場合は作業を見直して、ハンズオン当日までに必ず動作確認をお願いします。

### トラブル対応

イメージインポートにおいて、インポート先のCドライの容量が足りない場合などは、Import操作が終了しません。
コントロールCでコマンド中断後、%LOCALAPPDATA%\CQHandsOn-01 フォルダーを削除、Cドライブの空き領域を拡張してから、次のコマンドでインストール済ディストリビューションを削除します。
```cmd
> wsl --unregister Ubuntu-20.04cq
```
その後念のため再起動してから、WSLの停止を確認して再度インポートを実行します。

### 削除

使用しなくなったWSLイメージの削除は前項のトラブル対応とほぼ同じです。

wsl の停止を確認して、%LOCALAPPDATA%\CQHandsOn-01 フォルダーを削除、次のコマンドでインストール済ディストリビューションを削除します。
```cmd
> wsl --unregister Ubuntu-20.04cq
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
