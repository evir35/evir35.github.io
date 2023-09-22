---
title: Podmanを始める
date: 2022-02-20T17:05:53+09:00
tags: [container,docker,podman]
---

## Docker desktopの有料化

ほとんどのユーザーがContainerに接するのはDockerを通して接しているはずです。 Linux distribution(e.g. Ubuntu, CentOS, ...)を使うUserは問題ないですが、WindowsやMacOSではDockerをそのまま使うことができず、仮想化を通してしか使うことができません。 このようなユーザーの不便さを減らすために登場したのがDocker desktopです。 Docker desktopは[LinuxKit](https://github.com/linuxkit/linuxkit)を利用してDockerを使えるVirtual MachineをWindowsやMacOSで自動で構築してDockerを使える環境を作ってくれます。 このようなDocker desktopがBusiness目的で使われる場合、有料化されました。 2022.01.31以降、Business目的で使用する場合、Dockerにサブスクリプション形式で使用料を支払う必要があります。 このような変更は、既存のDocker desktopを無料で使っていたユーザーには負担になることでしょう。 Docker desktopの代替を探してた時、[podman](https://github.com/containers/podman)を発見しました。

## Podmanとは？

[Podman](https://github.com/containers/podman)はDocker CLIで使うほとんどのcommandが互換性があるconatiner/podを管理することができるツールです。 さらに[Podmanのドキュメント](https://docs.podman.io/en/latest/index.html)では `alias docker=podman`でaliasを付けて使っても大丈夫だと言ってるほどDocker CLIとほとんど互換性があります。 それでは実際podmanをインストールして使ってみます。

## 環境設定

今回のpostで使ったlocal環境は下記のような環境です。

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.1
```

## Install Podman

Macでは[homebrew](https://brew.sh/index_ko)を使って簡単にインストールできます。

```shell
brew install podman
```

上のコマンドでインストールした後、`version`commandを使ってインストールされたPodmanのバージョンを確認することができます。

```shell
$ podman version
Client:
Version:      3.4.4
API Version:  3.4.4
Go Version:   go1.17.6
Built:        Thu Dec  9 03:41:11 2021
OS/Arch:      darwin/arm64

Server:
Version:      3.4.4
API Version:  3.4.4
Go Version:   go1.16.8
Built:        Thu Dec  9 06:48:10 2021
OS/Arch:      linux/arm64
```

私がインストールする時は`3.4.4`が最新バージョンなので`3.4.4`でインストールしました。

## Podman machine

WindowsやMacOSではcontainerをすぐ動かすことができないので、containerを動かすことができるVirtaul Machineを作ってそのVirtual Machineを作ってこの上にcontainerを上げる方法を主に使います。 PodmanではWindowsやMacOSでも簡単にcontainerを使えるように[machine](https://docs.podman.io/en/latest/markdown/podman-machine.1.html)というコマンドを使ってVirtual Machineを簡単に作って、管理できるように提供しています。

### Virtual Machine生成する

[machine init](https://docs.podman.io/en/latest/markdown/podman-machine-init.1.html)を使ってVirtual Machineの生成が可能です。

```shell
podman machine init \
  --cpus 2 \
  --memory 4096 \
  --disk-size 32 \
  --image-path stable \
  default-vm
```

上のようにcpu 2core, memory 4GB, disk 32GBの`default-vm`という名前のPodman machineを生成しました。

```shell
$ podman machine list
NAME VM TYPE CREATED LAST UP CPUS MEMORY DISK SIZE
default-vm* qemu 12 seconds ago 12 seconds ago 2 4.295GB 34.36GB
```

[machine list](https://docs.podman.io/en/latest/markdown/podman-machine-list.1.html)コマンドで生成されたmachineを確認することができます。

### Virtual Machineを駆動させる

[machine start](https://docs.podman.io/en/latest/markdown/podman-machine-start.1.html)コマンドを使って生成されたmachineの駆動が可能です。

```shell
$ podman machine start default-vm
INFO[0000] waiting for clients...
INFO[0000] listening tcp://127.0.0.1:7777
INFO[0000] new connection from  to /var/folders/v5/4w4brkd5593764fjm96sjzpc0000gn/T/podman/qemu_default-vm.sock
Waiting for VM ...
Machine "default-vm" started successfully
```

そして駆動されてるか`machine list`で確認してみましょう。

```shell
$ podman machine list
NAME         VM TYPE     CREATED        LAST UP            CPUS        MEMORY      DISK SIZE
default-vm*  qemu        6 minutes ago  Currently running  2           4.295GB     34.36GB
```

`LAST UP`の部分を見ると、`Currently running`が表示されることが確認できます。

## Contianer生成する

今、containerが上がるVirtual Machineも生成されたので、そのmachineにcontainerを生成してみます。 dockerで提供してる[getting started](https://docs.docker.com/get-started/)で提供してるcontainer imageを使って生成してみます。 上の資料で提供してるコマンドをdockerからpodmanへだけ変更して進めてください。

```shell
# Create container using docker/getting-started image
$ podman run -d -p 8080:80 docker/getting-started
798b8a5a02819ff3a0c942879ae21348a1343bd14741b27edb6422d5db3216b7

# Listing containers
$ podman ps
CONTAINER ID  IMAGE                                    COMMAND               CREATED        STATUS             PORTS                 NAMES
798b8a5a0281  docker.io/docker/getting-started:latest  nginx -g daemon o...  9 seconds ago  Up 10 seconds ago  0.0.0.0:8080->80/tcp  beautiful_northcutt
```

localの8080 portでアクセスしたらcontainerの80 portでアクセスできるように設定しました。 その後、chromeでhttp://localhost:8080にアクセスすると下記のようにdockerのgetting-started pageを確認することができます。

![docker getting-started page](images/docker-getting-started-page.webp)

## 結論

Podmanを使ってdocker desktopを代替できるし、machineというコマンドを提供してVirtual Machineの制御ができる点が良かったです。 Docker desktopが必要なら、podmanの代わりに使うのもいいと思います。
