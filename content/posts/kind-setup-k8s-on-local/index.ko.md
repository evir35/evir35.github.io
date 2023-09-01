---
title: Kind로 kubernetes cluster를 local에서 구동하기
date: 2022-08-16T22:34:50+09:00
tags: [container,kubernetes,kind]
---

## Kind로 Kubernetes cluster를 local에서 구동하기

> kind is a tool for running local Kubernetes clusters using Docker container “nodes”.
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

[kind](https://kind.sigs.k8s.io/)는 위 설명처럼 local에서 kubernetes를 test용도로 사용하고 싶을 때, docker를 이용하여 kubernetes cluster를 쉽게 구축할 수 있도록 도와주는 opensource tool이다. 이름의 유래도 재밌는데 바로 `Kubernetes IN Docker`에서 각 단어에서 `K`, `IN`, `D`를 따와서 `kind`가 되었다. [kind github repository](https://github.com/kubernetes-sigs/kind/)의 description을 확인해보면 알 수 있다.

오늘은 kind cluster를 local에서 생성하고, nginx를 nodeport로 노출해보고 삭제하는 식으로 kind를 테스트해보려고 한다.

## Environment

이번 post에서 사용된 local 환경은 아래외 같습니다.

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.6.3
```

## Install kind

Mac의 경우 [Homebrew](https://brew.sh/index_ko)로 쉽게 설치가 가능하지만 특정 버전을 설치하기 위해서 binary 설치로 진행하겠습니다.

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.19.0/kind-darwin-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Check kind CLI

brew를 통해서 kind가 제대로 설치되었는지 version과 CLI help를 확인해봅시다. 아래처럼 나온다면 kind CLI가 제대로 설치되었다고 보시면 됩니다.

```sh
$ kind --version
kind version 0.19.0
$ kind --help
kind creates and manages local Kubernetes clusters using Docker container 'nodes'

Usage:
  kind [command]

Available Commands:
  build       Build one of [node-image]
  completion  Output shell completion code for the specified shell (bash, zsh or fish)
  create      Creates one of [cluster]
  delete      Deletes one of [cluster]
  export      Exports one of [kubeconfig, logs]
  get         Gets one of [clusters, nodes, kubeconfig]
  help        Help about any command
  load        Loads images into nodes
  version     Prints the kind CLI version

Flags:
  -h, --help              help for kind
      --loglevel string   DEPRECATED: see -v instead
  -q, --quiet             silence all stderr output
  -v, --verbosity int32   info log verbosity, higher value produces more output
      --version           version for kind

Use "kind [command] --help" for more information about a command.
```

## Create Cluster

아래처럼 `kind create cluster` 명령어를 입력해주시면 간단하게 cluster 생성이 가능합니다.

```sh
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.27.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

### Check kubernetes cluster-info and nodes

kind cluster 생성 후 자동으로 context가 `kind-kind`로 변경되어서 `kubectl cluster-info`, `kubectl get nodes -owide`로 확인해보면 제대로 kubernetes cluster가 생성된 것을 확인하실 수 있습니다.

```sh
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:58181
CoreDNS is running at https://127.0.0.1:58181/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes -owide       
NAME                 STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   17m   v1.27.1   172.18.0.2    <none>        Debian GNU/Linux 11 (bullseye)   5.15.82-0-virt   containerd://1.6.21
```

### Check docker containers

아래처럼 현재 구동중인 container를 확인해보면 `kindset/node` image를 사용 중인 container 하나만 구동 중인 모습을 확인하실 수 있습니다.

```sh
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                       NAMES
8a885e2d2b28   kindest/node:v1.27.1   "/usr/local/bin/entr…"   2 minutes ago   Up 2 minutes   127.0.0.1:58181->6443/tcp   kind-control-plane
```

## Setup LoadBalancer for kind with metalLB

### Setup routing table for docker network

- Reference: https://opencredo.com/blogs/building-the-best-kubernetes-test-cluster-on-macos/

위 blog를 참고하여 macbook과 colima machine에 대한 설정을 진행합니다. (제가 작성한 script를 이용하여 아래 command를 수행하면 위 설정을 자동으로 수행가능합니다.)

```sh
curl -L https://gist.githubusercontent.com/evir35/6fbd7382629e60bef08ac39e1968f2df/raw/c638578991b4a8b16304dde327f9621dae2bc03c/colima-network-setup.sh 2>/dev/null | bash -- <&1
```

### Install metalLB on kind cluster

아래 명령어를 사용하여 kind cluster에 metalLB를 설치합니다. ([참조](https://kind.sigs.k8s.io/docs/user/loadbalancer/))

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=90s
```

### Docker kind network IP range 확인

아래 명령어로 확인을 해보면 제 local의 경우 `172.18.0.0/16` IP range가 kind network에서 사용 가능한 것을 확인하실 수 있습니다.

```sh
$ docker network inspect -f '{{.IPAM.Config}}' kind
[{172.18.0.0/16  172.18.0.1 map[]} {fc00:f853:ccd:e793::/64   map[]}]
```

### MetalLB IP range 설정

```sh
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250 # Please update this addresses to your IP range
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```

### MetalLB LoadBalancer를 사용하는 example resource 생성

```sh
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: http-echo
spec:
  containers:
  - name: foo-app
    image: strm/helloworld-http
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: http-echo
spec:
  containers:
  - name: bar-app
    image: strm/helloworld-http
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  type: LoadBalancer
  selector:
    app: http-echo
  ports:
  # Default port used by the image
  - port: 80
EOF
```

### Test metalLB

생성된 Service에서 LB의 IP를 확인합니다. 저의 경우는 `172.18.255.200` IP로 LB가 생성되었습니다.

```sh
$ kubectl get service
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
foo-service   LoadBalancer   10.96.220.49   172.18.255.200   80:30790/TCP   15d
kubernetes    ClusterIP      10.96.0.1      <none>           443/TCP        15d
```

해당 IP로 curl을 해보면 아래처럼 번갈아가면서 hostname이 나오는 것을 확인하실 수 있습니다.

```sh
$ curl 172.18.255.200
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from bar-app</h1></body></html
$ curl 172.18.255.200
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from foo-app</h1></body></html
```

## Conclusion

colima와 kind를 사용하여 로컬에서 k8s cluster를 구축하고, metalLB를 이용하여 Loadbalancer Type의 서비스를 활용할 수 있도록 구성을 해보았습니다. 아직 network 관련 세팅이 필요한 점이 조금 아쉽지만 간단한 테스트를 위해서 local에서 k8s를 구동해서 테스트 후 삭제하였다가, 새롭게 다시 만드는 식으로 테스트가 가능해서 괜찮은 방식으로 생각됩니다.

## Reference

- colima: [https://github.com/abiosoft/colima](https://github.com/abiosoft/colima)
- kind: [https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/)
- metallb: [https://metallb.universe.tf/](https://metallb.universe.tf/)
- Building the best Kubernetes test cluster on MacOS: [https://opencredo.com/blogs/building-the-best-kubernetes-test-cluster-on-macos/](https://opencredo.com/blogs/building-the-best-kubernetes-test-cluster-on-macos/)

## ETC

### Errors with colima

kind를 이용해 kubernetes cluster를 만들던 도중 아래와 같은 에러를 조우하였습니다.

```sh
$ kind create cluster       
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼 
 ✗ Preparing nodes 📦  
Deleted nodes: ["kind-control-plane"]
ERROR: failed to create cluster: command "docker run --name kind-control-plane --hostname kind-control-plane --label io.x-k8s.kind.role=control-plane --privileged --security-opt seccomp=unconfined --security-opt apparmor=unconfined --tmpfs /tmp --tmpfs /run --volume /var --volume /lib/modules:/lib/modules:ro -e KIND_EXPERIMENTAL_CONTAINERD_SNAPSHOTTER --detach --tty --label io.x-k8s.kind.cluster=kind --net kind --restart=on-failure:1 --init=false --cgroupns=private --publish=127.0.0.1:57848:6443/TCP -e KUBECONFIG=/etc/kubernetes/admin.conf kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72" failed with error: exit status 125
Command Output: 8a9ea5e5f1310b7afd801ef35ec105ce9b69c585a9ac1e343c8bfecb42ef3577
docker: Error response from daemon: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error mounting "cgroup" to rootfs at "/sys/fs/cgroup": mount cgroup:/sys/fs/cgroup/openrc (via /proc/self/fd/7), flags: 0xe, data: openrc: invalid argument: unknown.
```

[해당 이슈](https://github.com/kubernetes-sigs/kind/issues/3277)와 관련된 문제로 보입니다. colima에서 생성하는 VM에서 cgroup v2를 지원하지 않아서 발생하는 문제로 보입니다. 관련 PR이 merge되지 않은 `0.19.0`에서는 에러가 발생하지 않는다는 [comment](https://github.com/kubernetes-sigs/kind/issues/3277#issuecomment-1643943889)를 보고 `0.19.0` binary를 직접 설치하였더니 문제없이 kind cluster가 생성되었습니다.