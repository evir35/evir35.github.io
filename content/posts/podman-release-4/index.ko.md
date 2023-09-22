---
title: Podman 4.0.0 release
date: 2022-03-21T11:44:53+09:00
tags: [container,docker,podman]
---

## Podman 4.0.0 Release

Podman 4.0.0이 2022/02/18에 릴리즈 되었습니다.

> Podman 4.0 is one of our most significant releases ever, featuring over 60 new features.

공식 사이트에서는 무려 60개 이상의 새로운 feature가 추가되었다고 소개하고 있습니다.
이 중에서 제가 소개해드리고 싶은 새로운 feature에 대해서 간단한 테스트와 소개를 해보고자 합니다.
모든 새로운 feature를 확인하고 싶으신 분들은 [Podman v4.0.0 release notes](https://github.com/containers/podman/releases/tag/v4.0.0)를 확인해주세요.

## Environment

이번 post에서 사용된 local 환경은 아래외 같습니다.
```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.1
```

## Podman upgrade

[이전 post](/posts/docker-alternatives-podman/index.ko.md)에서도 보셨겠지만 Mac에서는 [Homebrew](https://brew.sh/index_ko)를 이용해 간단하게 upgrade가 가능합니다.

```sh
brew upgrade podman
```

이전에 설치했던 podman의 버전은 `3.4.4` 였지만 Hombrew를 통해 upgrade를 진행한 후에는 upgrade된 버전이 나옵니다.

```sh
$ podman --version
podman version 4.0.2
```

제가 설치할 때에는 최신버전이 `4.0.2`라서 version이 `4.0.2`로 나옵니다.
그럼 podman이 upgrade 되었으니 새로운 기능들에 대해서 알아봅시다.

## Podman machine

Podman에서 제공하는 virtual machine 기능이 편리했지만 불편했던 점들도 다수 존재했었는데요.
이번의 v4.0.0으로 업데이트가 되면서 불편했던 부분들이 상당히 해소가 되었습니다.

### 공식적인 WSL 지원

Windows에서는 [WSL](https://docs.microsoft.com/ko-kr/windows/wsl/)이라고 하는 Windows에서 linux 시스템을 사용할 수 있도록 제공하고 있습니다.
Podman에서는 WSL을 virtual machine type으로 제공되지 않았었지만 이번 릴리즈부터 WSL을 공식적으로 지원하기 시작했습니다.

Windows OS를 사용하시는 분들에게는 좋은 소식입니다.

### Podman machine volume

이전 Podman machine에서는 virtual machine이기 때문에 local storage와는 격리된 환경입니다.
즉, Mac 상에서 존재하는 file들을 container에서 mount하고 싶다면 먼저 virtual machine에 file을 옮기고 container에 mount를 해야했습니다.
하지만 이번에 새로 추가된 volume 기능을 이용하면 파일을 따로 옮기지 않고도 local의 파일을 사용 가능합니다.
다음 예제를 통해서 환경을 구축해봅시다.

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

Mac local storage의 `/Users/user`란 폴더를 그대로 virtual machine의  `/var/home/mac`에 mount 시키도록 virtual machine을 생성해 주었습니다.

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

volume mount 기능을 테스트해보기 위해서 Mac `/Users/user`란 폴더에서 `hello-podman.txt`란 임시 파일을 생성해 보았습니다.
`podman machine ssh`로 생성한 virtual machine에 접속한 뒤, `/var/home/mac/hello-podman.txt` 파일 내용을 확인해보면 저희가 임시로 생성한 파일 내용인 `hello podman 4.0`이 출력되는 것을 보실 수 있습니다.

해당 기능을 사용해 자주 사용하는 폴더를 미리 mount해둔 뒤에 작업하실 때 시용하시면 좋을 것 같습니다.

## 결론

이전에도 podman을 사용하면서 machine에 volume mount가 되지 않아서 local에 있는 file을 pod에 mount하는 것이 불가능했기 때문입니다.
하지만 이번 업데이트에서 추가되어서 편해진 것 같습니다. podman이 유용한 기능들을 빠르게 업데이트하고 있어서 기대가 큽니다.
docker를 대신하는 container/pod management tool의 defacto가 되기를 기원해봅니다.