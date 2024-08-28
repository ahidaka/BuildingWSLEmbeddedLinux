AVNET MaaXBoard-8ULP Yocto 環境インストール 報告

株式会社デバイスドライバーズ 日高亜友

## 参考資料

最初に教えて頂いたページ

https://www.avnet.com/wps/portal/us/products/avnet-boards/avnet-board-families/maaxboard/maaxboard-8ulp/

上記では「Refeference Designs」タブのPre-Built Linux Image (Full), BootLoader u-boot Image
がダウンロード出来ませんでした。それで入手先を質問しました。
以下のページを教えて頂きました。

https://github.com/Avnet/MaaXBoard-8ULP-HUB

ここの Software & BSP の直下の「Yocto source files」の手順

https://github.com/Avnet/meta-maaxboard

に従って作業しましたが、途中でエラーが発生します。
調べるとこれは昔の古い情報らしいことがわかりました。

さらに調べると MaaXBoard 8ULP 用の情報はこの MaaXBoard-8ULP-HUBのページの下の方、
Development Guides にある、「Yocto Development Guide v3.1」のPDFらしいことが判明しました。
そしてこのPDFは、最初に教えて頂いたページの「Technical Documents」タブの
MaaXBoard-8ULP-Linux-Yocto-Development-Guide-V3.1 と同じことが判明しました。

なお最初に探していた、Linux Image (Full)と BootLoader u-boot Image については、ここのページ 
(https://github.com/Avnet/MaaXBoard-8ULP-HUB)
のページ中ほど記載のリンクからダウンロードして、動作確認が出来ています。


## Yocto Development Guide v3.1

MaaXBoard-8ULP-Linux-Yocto-Development-Guide-V3.1.pdf
に従って作業しましたが、いくつか不備と注意点があったので記載します。

- 実行環境

  実行環境は最低16GBのメモリーと400GB程度の空きストレージが必要と推測します。 後述の通り、全てのビルドを完了出来ていないため、u-bootとCotex-M開発環境等を全てインストールした場合は、もう少しストレージ容量が必要です。

- local.conf

  デフォルト設定のまま、bitbake avnet-image-full などを実行すると、論理 10スレッド以上のマルチCPU環境では、メモリー不足でエラーが発生し易くなるため、imx-yocto-bsp/maaxboard-8ulp/build/conf/local.conf に以下の設定を追加する必要があります。
```
  BB_NUMBER_THREADS = "8"
  PARALLEL_MAKE = "-j 8"
```

- 操作コマンドのコピー

  コマンド入力内容はPDFから転記をしますが、数か所で余計な改行があるため、コマンド転記時は注意が必要です。

- 1.1 Setup Build Environment

  次のコマンド実行が必要です。
  ```
  sudo apt-get install make build-essential libncurses-dev bison flex libssl-dev libef-dev
  ```

- 2.1.1 ARM GCC

  下記コマンドは、コピー時にaarch64-none-linuxgnuと文字化けするため、修正が必要です。
TOOLCHAIN_PATH=$HOME/toolchain/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin


- 2.1.2 Yocto SDK

  The generated file is: ~/imx-yocto-bsp/maaxboard-8ulp/build/tmp/deploy/sdk/
fsl-imx-wayland-lite-glibc-x86_64-avnet-image-full-armv8a-maaxboard-8ulp-toolchain-6.1-mickledore..sh
とありますが、正しくは
~/imx-yocto-bsp/maaxboard-8ulp/build/tmp/deploy/sdk/
fsl-imx-wayland-lite-glibc-x86_64-avnet-image-full-armv8a-maaxboard-8ulp-toolchain-6.1-mickledore.sh
です。

- 2.2.2 Compile script

  PDF内記述のmake_mx8ulp_uboot.shスクリプトにエラーがあるため、正常実行出来ません。
  下記のmcore_sdk_8ulp.git のcloneが必要と推測します。
  ```
  git clone https://github.com/Avnet/mcore_sdk_8ulp.git -b maaxboard_lf-6.1.22-2.0.0
  ```

  これらのエラーの結果 fsl-imx-wayland-lite-glibc-x86_64-avnet-image-full-armv8a-maaxboard-8ulp-toolchain-6.1-
mickledore.sh が途中で終了するため、u-bootとcortex-mはビルド出来ないままとなっています。

- セミナー利用

  しかし、2.1.2 Yocto SDK と 2.3 Build Kernel in a standalone environment は正常終了します。
    
  そして MaaXBoard-8ULP-Linux-Yocto-UserManual(MaaXBoard-8ULP-Linux-Yocto-UserManual-V3.1.pdf) の手順に従って、ダウンロードした Linux Image (Full)と BootLoader u-boot Image で、MaaXBoard-8ULP ボードの起動を確認し、SDKおよびビルドを確認したカーネルイメージを使用してドライバーとアプリケーションのビルドが出来ることを確認したので、私の担当するドライバー開発のセミナーには問題が無いと判断しました。

  なお 今回のYoctoの動作環境をWSL上で構築して、bitbake avnet-image-full の代わりに、bitbake core-image-minimal などを実行することで、使用するストレージ容量を削減出来ることを確認しています。

- レシピファイル

  関連して、imx-yocto-bsp/sources/meta-maaxboard/images ディレクトリには説明が無い
avnet-image-chromium.bb avnet-image-lite.bb avnet-image-minimal.bb avnet-image-oob.bb のレシピファイルがありますが、それがどの用に使えるのかについては不明で、今後の課題です。

以上
