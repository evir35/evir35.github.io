---
title: Podman 4.0.0 release
date: 2022-03-21T11:44:53+09:00
tags: [container,docker,podman]
---
## Podman 4.0.0 Release

Podman 4.0.0 が 2022/02/18 にリリースされました。

> Podman 4.0 is one of our most significant releases ever, featuring over 60 new features.

公式サイトではなんと60以上の新しいfeatureが追加されたと紹介しています。 この中で私が紹介したい新しいfeatureについて簡単なテストと紹介をしてみたいと思います。 全ての新しいfeatureを確認したい方は[Podman v4.0.0 release notes](https://github.com/containers/podman/releases/tag/v4.0.0)を確認してください。

## 環境設定

今回のpostで使ったlocal環境は下記の環境です。

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.1
```

## Podman upgrade

[以前の post](/posts/docker-alternatives-podman/index.ja.md")でも見ましたが、Macでは[Homebrew](https://brew.sh/index_ko)を使って簡単にupgradeすることができます。

```sh
brew upgrade podman
```

以前インストールしたpodmanのバージョンは`3.4.4`でしたが、Hombrewを使ってupgradeをしたらupgradeされたバージョンが出ます。

```sh
$ podman --version
podman version 4.0.2
```

私がインストールする時、最新バージョンが`4.0.2`なのでversionが`4.0.2`と出ます。 じゃ、podmanがupgradeされたので、新しい機能について説明します。

## Podman machine

Podmanで提供してるvirtual machine機能が便利でしたが、不便な点も多く存在しました。 今回のv4.0.0でアップデートされて不便だった部分がかなり解消されました。

### 公式的なWSLサポート

Windowsでは[WSL](https://docs.microsoft.com/ko-kr/windows/wsl/)というWindowsでlinuxシステムを使えるように提供しています。 PodmanではWSLをvirtual machine typeとして提供していませんでしたが、今回のリリースからWSLを公式にサポートするようになりました。

Windows OSを使ってる方には良いニュースです。

### Podman machine volume

以前のPodman machineではvirtual machineなのでlocal storageとは隔離された環境です。 つまり、Mac上で存在するfilesをcontainerでmountしたい場合、最初にvirtual machineへfileを移動してcontainerへmountする必要がありました。 しかし、今回新しく追加されたvolume機能を使うとファイルを別に移動しなくてもlocalのファイルを使うことができます。 次の例を使って環境を構築してみましょう。

```sh
podman machine init \
  --cpus 2 \
  --memory 4096 \
  --disk-size 32 \
  --image-path stable \
  --volume /Users/user:/var/home/mac \
  default-vm

podman machine start default-vm
```

Mac local storageの`/Users/user`というフォルダをそのままvirtual machineの`/var/home/mac`にmountさせるようにvirtual machineを生成しました。

```sh
$ echo "hello podman 4.0" > hello-podman.txt
$ podman machine ssh default-vm
Connecting to vm default-vm. To close connection, use `~.` or `exit`
Warning: Permanently added '[localhost]:56443' (ED25519) to the list of known hosts.
Fedora CoreOS 35.20220305.dev.0
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/tag/coreos

Last login: Tue Mar 22 00:33:06 2022 from 192.168.127.1

$ cat /var/home/mac/hello-podman.txt
hello podman 4.0
```

volume mount機能をテストするためMac`/Users/user`というフォルダで`hello-podman.txt`という一時ファイルを生成してみました。 `podman machine ssh`で生成したvirtual machineに接続した後、`/var/home/mac/hello-podman.txt`ファイルの内容を確認したら、私たちが一時的に生成したファイルの内容である`hello podman 4.0`が出力されることが確認できます。

この機能を使ってよく使うフォルダをあらかじめmountしておいて、作業する時使ってみるといいと思います。

## 結論

以前にもpodmanを使う時、machineにvolume mountができないのでlocalにあるfileをpodにmountすることが不可能でした。 しかし、今回のアップデートで追加されて楽になったようです。 podmanが便利な機能をどんどんアップデートしてるので期待が大きいです。 dockerに代わるcontainer/pod management toolのdefactoになることを祈っています。
