---
title: Kindでkubernetes clusterをlocalで動かす方法
date: 2022-08-16T22:34:50+09:00
tags: [container,kubernetes,kind]
---
## KindでKubernetes clusterをlocalで動かす方法

> kindはDockerコンテナ「nodes」を使ってローカルでKubernetesクラスタを動かすためのツールです。 kindは主にKubernetes自体のテスト用に設計されましたが、ローカルでの開発やCIに使うこともできます。

[kind](https://kind.sigs.k8s.io/)は上の説明のようにlocalでkubernetesをtest用に使いたい時、dockerを利用してkubernetes clusterを簡単に構築できるように助けてくれるopensource toolです。名前の由来も面白いですが、`Kubernetes IN Docker`から`K`, `IN`, `D`を取って `kind`になりました。[kind github repository](https://github.com/kubernetes-sigs/kind/)のdescriptionを確認すると分かります。

今日はkind clusterをlocalで生成して、nginxをnodeportで公開し、削除することでkindをテストしてみます。

## 環境設定

今回のpostで使ってるlocal環境は下記のような環境です。

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.6.3
```

## Install kind

Macの場合[Homebrew](https://brew.sh/index_ko)で簡単にインストールが可能ですが、特定のバージョンをインストールするためbinaryでインストールします。

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.19.0/kind-darwin-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Check kind CLI

brewを使ってkindが正しくインストールされたかversionとCLI helpを確認してみましょう。下記のように表示されたらkind CLIがちゃんとインストールされたとみてください。

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

下記のように`kind create cluster`コマンドを入力すると簡単にclusterを生成することができます。

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

### kubernetes cluster-info and nodesの確認

kind cluster生成後、自動的にcontextが`kind-kind`に変更されて`kubectl cluster-info`、`kubectl get nodes -owide`で確認してみると、ちゃんとkubernetes clusterが生成されたことが確認できます。

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

下記のように現在動いてるcontainerを確認すると`kindset/node`imageを使ってるcontainer一つだけ動いてることが確認できます。

```sh
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                       NAMES
8a885e2d2b28   kindest/node:v1.27.1   "/usr/local/bin/entr…"   2 minutes ago   Up 2 minutes   127.0.0.1:58181->6443/tcp   kind-control-plane
```

## metalLBでkind用のLoadBalancerを設定する

### dockerネットワーク用ルーティングテーブルの設定

* Reference: https://opencredo.com/blogs/building-the-best-kubernetes-test-cluster-on-macos/

上のblogを参考してmacbookとcolima machineの設定をします。 (私が作ったscriptを使って下記のcommandを実行したら、上の設定を自動でできます)

```sh
curl -L https://gist.githubusercontent.com/evir35/6fbd7382629e60bef08ac39e1968f2df/raw/c638578991b4a8b16304dde327f9621dae2bc03c/colima-network-setup.sh 2>/dev/null | bash -- <&1
```

### Install metalLB on kind cluster

下記のコマンドを使ってkind clusterにmetalLBをインストールします。([参考](https://kind.sigs.k8s.io/docs/user/loadbalancer/))

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system \ ...
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=90s
```

### Docker kind network IP range確認

下記のコマンドで確認したら、私のlocalの場合`172.18.0.0.0/16`IP rangeがkind networkで使えることが確認できます。

```sh
$ docker network inspect -f '{{.IPAM.Config}}'kind
[{172.18.0.0.0/16 172.18.0.1 map[]}].{fc00:f853:ccd:e793::/64 map[]}]].
```

### MetalLB IP range設定

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

### MetalLB LoadBalancerを使うexample resourceを生成します。

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

生成されたServiceでLBのIPを確認します。私の場合は`172.18.255.200`IPでLBが生成されました。

```sh
$ kubectl get service
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
foo-service   LoadBalancer   10.96.220.49   172.18.255.200   80:30790/TCP   15d
kubernetes    ClusterIP      10.96.0.1      <none>           443/TCP        15d
```

このIPでcurlをしてみると、下記のようにhostnameが交互に出ることが確認できます。

```sh
$ curl 172.18.255.200
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from bar-app</h1></body></html
$ curl 172.18.255.200
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from foo-app</h1></body></html
```

## Conclusion

colimaとkindを使ってローカルでk8s clusterを構築して、metalLBを使ってLoadbalancer Typeのサービスを活用できるように構成してみました。 まだnetwork関連の設定が必要な点が少し残念ですが、簡単なテストのためlocalでk8sを起動してテストした後、削除して、新しく作り直す方法でテストができるので、いい方法だと思います。

## Reference

* colima:<https://github.com/abiosoft/colima>
* kind:<https://kind.sigs.k8s.io/>
* metallb:<https://metallb.universe.tf/>
* Building the best Kubernetes test cluster on MacOS:<https://opencredo.com/blogs/building-the-best-kubernetes-test-cluster-on-macos/>

## その他

### Errors with colima

kindを使ってkubernetes clusterを作る時、下記のようなエラーが発生しました。

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

[この問題](https://github.com/kubernetes-sigs/kind/issues/3277)は、colimaで生成するVMでcgroup v2をサポートしてないので発生する問題だと思われます。関連PRがmergeされてない`0.19.0`ではエラーが発生しないという[コメントを見](https://github.com/kubernetes-sigs/kind/issues/3277#issuecomment-1643943889)て `0.19.0`binaryを直接インストールしたら問題なくkind clusterが生成されました。
