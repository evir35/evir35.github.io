---
title: Podman 4.0.0 release
date: 2022-03-21T11:44:53+09:00
tags: [container,docker,podman]
---

## Podman 4.0.0 Release

Podman 4.0.0 was released on February 18, 2022.

> Podman 4.0 is one of our most significant releases ever, featuring over 60 new features.

The official site lists over 60 new features. I'm going to do a quick test and introduction of some of the new features I'd like to share with you. If you want to check out all the new features, check out the [Podman v4.0.0 release notes](https://github.com/containers/podman/releases/tag/v4.0.0).

## Environment

The local environment used in this post is as follows

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: macOS Monterey 12.1
```

## Podman upgrade

As you may have seen in the [previous post](/posts/docker-alternatives-podman/index.ko.md), upgrading is simple on Mac using [Homebrew](https://brew.sh/index_ko).

```sh
brew upgrade podman
```

The previously installed version of podman was `3.4.4`, but after upgrading via Hombrew, the upgraded version will appear.

```sh
podman --version
podman version 4.0.2
```

In my installation, the latest version is `4.0.2, so`the version is `4.0.2`. Now that podman has been upgraded, let's see what's new.

## Podman machine

The virtual machine function provided by Podman was convenient, but there were many inconveniences. With this update to v4.0.0, the inconveniences have been significantly resolved.

### Official WSL support

Windows provides a way to use linux systems on Windows called [WSL](https://docs.microsoft.com/ko-kr/windows/wsl/). Podman hasn't offered WSL as a virtual machine type, but with this release, we've officially started supporting WSL.

This is great news for those of you using Windows OS.

### Podman machine volume

In previous Podman machines, because they are virtual machines, they are isolated from local storage. This means that if you wanted to mount files from a container that existed on your Mac, you had to move them to the virtual machine and mount them to the container. However, with the new volume feature, you can use the files on your local without moving them. Let's create an environment with the following example.

```sh
podman machine init \
  --cpus 2 \
  --memory 4096
  --disk-size 32 \
  --image-path stable \
  --volume /Users/user:/var/home/mac \
  default-vm

podman machine start default-vm
```

We've created the virtual machine so that it mounts the folder `named /Users/user` on the Mac's local storage to `/var/home/mac` in the virtual machine.

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

To test the volume mount feature, I created a temporary file `named hello-podman.txt` in a folder called `/Users/user` on my Mac. If you connect to the virtual machine we created `with podman machine ssh` and check the contents of the `/var/home/mac/hello-podman.txt` file, you can see the contents of our temporary file, `hello podman 4.0`, is displayed.

You may want to try this out by mounting your favorite folders beforehand.

## Conclusion

I've used podman before, and it was impossible to mount a file locally to a pod because it wouldn't volume mount to the machine. However, it seems to have been added in this update, and I'm excited that podman is quickly updating useful features. Here's to hoping it becomes the defacto container/pod management tool to replace docker.
