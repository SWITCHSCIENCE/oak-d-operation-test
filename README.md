# Luxonis OAK-D S2 を試してみた

## 製品リンク

https://www.switch-science.com/products/8268

この製品の利点は以下の３つのデバイスが内包されていること。

- 4Kカメラ
- ステレオ深度カメラ
- ニューラルエンジン

これらを別々に買うよりは安く済むというところと4Kカメラと深度カメラのキャリブレーションが出荷時に済んでいるところ。

## 環境づくりについて

- depthai_sdkコアやルクソニス自体が管理しているライブラリは頑張っていろんな環境用のバイナリを用意してくれている
- 実際に使うとなるとかなり多様なライブラリを利用することになるがこちらのバイナリは主要環境向けだけだったりする
- ラズパイで動く環境を作ろうとするといろんな依存をソースからビルドしようとし始めちゃう
- Open3DやOpenGL、OpenCVなど巨大なものはリンクで大量にメモリを要求するので状況によってはメモリ不足で失敗する
- ラズパイのストレージは非力なことが多く、これらのビルドには日数レベルが必要になる
- どうしてもラズパイ環境を構築するならいったんメモリの潤沢なPC上でDockerのエミュレーションモードでビルドする

## Dockerのエミュレーションモードについて

Dockerにはマルチアーキテクチャ対応の機能が備わっています。

```
docker run --rm --privileged tonistiigi/binfmt:latest --install linux/amd64,linux/arm64,linux/ppc64le,linux/s390x,linux/386,linux/arm/v7,linux/arm/v6
```
この呪文めいたコマンドを実行するとdockerホスト側に`qemu-system-*`がインストールされ、コンテナ実行時のアーキテクチャ別に所定の場所にマウントされるようになります。

qemuの実行バイナリが存在し、ホストアーキテクチャと異なるイメージを実行する時に実行ファイルはqemuを通して実行されます。

そのDocker環境の場合、以下のコマンドでRaspberryPi4などと同じアーキテクチャのイメージを異なるホストマシンアーキテクチャ上でも実行することができます。
```
docker run -it --rm --platform linux/arm64 arm64v8/alpine:latest
```

ただし、DockerはUSB接続サポートがないのであくまで依存ライブラリのビルドまで。それをラズパイ側に転送して使うという手法になります。

## demo実装について

ここが「ルクソニスが提供する基本のデモ」が一通りあるところ。

https://github.com/luxonis/depthai-experiments

ここではIMU利用のサンプル、デプスカメラのサンプル各種、ニューラルエンジンのサンプルなど分離した形のデモが一通りあります。

おすすめのPythonバージョンは3.8～3.10くらい。
なぜなら一部の依存ライブラリのバイナリ版がまだ最新Python版がリリースされていないのです。

ここではPython3.8を前提に解説していきます。
また、可能な限り、venv(もしくはvirtualenv)にてパッケージインストール先を本体とは別にしたほうがいいです。

上記のデモだけならだいぶ調整済みだから問題は起きにくいですが、外部デモのrequirements.txtはバージョン固定しているものが多く、この外部デモを動かしたら一部の依存ライブラリが古いバージョンに差し替えられてさっき動いていたデモが動かなくなったりします。

## 動作環境について

1. Webカメラ＋CPUパワー
2. Webカメラ＋GPUパワー
3. OAK-D＋CPUパワー
4. OAK-D＋内蔵ニューラルエンジン

1.と2.が一般的によく使われる環境ですが、OAK-Dはニューラルエンジンも内包しているので
4.の「OAK-D＋内蔵ニューラルエンジン」が上手く使ったサンプルの場合、かなり省エネでフレームレートもしっかり出せてる。

## ハンドジェスチャー認識デモ

デプスカメラ＋ニューラルエンジン

デプスカメラからの映像をもとに手の姿勢トラッキングを行いますが、OAK-D内蔵のニューラルエンジンを利用するオプションがあります。

https://github.com/geaxgx/depthai_hand_tracker

```
git clone https://github.com/geaxgx/depthai_hand_tracker
cd depthai_hand_tracker
python38 -m venv .venv
.venv/Scripts/activate.bat // for cmd.exe
source .venv/Scripts/activate // for git-bash
source .venv/bin/activate // for macOS or Linux
pip install -r requirements.txt
python demo.py <- CPUでニューラル処理
python demo.py -e <- 内蔵ニューラルエンジンを利用
```

https://github.com/SWITCHSCIENCE/oak-d-operation-test/assets/616462/8da53e37-61d3-4fa9-953f-66534471c852


## ビジュアルマッピングデモ

デプスカメラ＋IMU

IMUで本体の位置トラッキングを行いつつ3D空間のマッピングを行うデモ

```
git clone https://github.com/SpectacularAI/sdk-examples
cd sdk-examples/python/oak
python38 -m venv .venv
.venv/Scripts/activate.bat // for cmd.exe
source .venv/Scripts/activate // for git-bash
source .venv/bin/activate // for macOS or Linux
pip install -r requirements.txt
pip install open3d
python mapping_visu.py
```

![IMG_6183](https://github.com/SWITCHSCIENCE/oak-d-operation-test/assets/616462/21919980-3ddb-4f71-983a-39ba7c6d7723)

以上の環境をOAK-Dを動かしてマッピングした結果が以下です。
赤い４角錐がOAK-Dの位置と見ている視野です。

![image](https://github.com/SWITCHSCIENCE/oak-d-operation-test/assets/616462/946dee70-5152-4293-be42-3427e159fc2d)

## まとめ

- 現状あるサンプルコードは基礎的な機能利用のものがほとんど
- ニューラル系サンプルは確かにフレームレートが高い
- ラズパイなどで動かそうとすると環境構築にかなり気合が必要
- 逆に自分でやりたいことを組み立てる場合は基礎サンプルをつなぎ合わせる方が助かる
