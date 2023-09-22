---
title: Running a Kubernetes cluster locally with kind
date: 2022-08-16T22:34:50+09:00
tags: [container,kubernetes,kind]
---

## Running a Kubernetes cluster locally with kind

> kind is a tool for running local Kubernetes clusters using Docker container "nodes". kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI. kind is a tool for running local Kubernetes clusters using Docker container "nodes".

[kind](https://kind.sigs.k8s.io/) is an opensource tool that helps you easily build a kubernetes cluster using docker when you want to use kubernetes locally for testing, as described above. The name is also interesting: `Kubernetes IN Docker`, where `K`, `IN`, and `D` are taken from each word to form `kind`. Check out the description in the [kind github repository](https://github.com/kubernetes-sigs/kind/).

Today, we're going to test kind by creating a kind cluster locally, exposing nginx as a nodeport, and deleting it.

## Environment

The local environment used in this post is as follows.

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.6.3
```

## Install kind

On Mac, you can easily install [Homebrew](https://brew.sh/index_ko), but to install a specific version, let's proceed with binary installation.

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.19.0/kind-darwin-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Check kind CLI

Let's check the version and CLI help to see if kind is installed properly via brew. If it looks like below, the kind CLI is installed properly.

```sh
kind --version
kind version 0.19.0
kind --help
kind creates and manages local Kubernetes clusters using Docker container 'nodes'

Usage:
  $ kind [command]

Available Commands:
  build Build one of [node-image].
  completion Output shell completion code for the specified shell (bash, zsh or fish)
  create Creates one of [cluster]
  delete Deletes one of [cluster].
  export Exports one of [kubeconfig, logs].
  get Gets one of [clusters, nodes, kubeconfig].
  help Help about any command
  load Loads images into nodes
  version Prints the kind CLI version

Flags:
  -h, --help help for kind
      --loglevel string DEPRECATED: see -v instead
  -q, --quiet silence all stderr output
  -v, --verbosity int32 info log verbosity, higher value produces more output
      --version version for kind

Use "kind [command] --help" for more information about a command.
```

## Create Cluster

You can create a cluster by entering the `kind create cluster` command like below.

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

After creating a kind cluster, the context is automatically changed `to kind-kind`, so you can check with `kubectl cluster-info` and `kubectl get nodes -owide` to see that the kubernetes cluster was created properly.

```sh
kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:58181
CoreDNS is running at https://127.0.0.1:58181/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes -owide      
name status roles age version internal-ip external-ip os-image kernel-version container-runtime
kind-control-plane Ready control-plane 17m v1.27.1 172.18.0.2 <none> Debian GNU/Linux 11 (bullseye) 5.15.82-0-virt containerd://1.6.21
```

### Check docker containers

If you check the currently running containers like below, you can see that only one container is running, which is using the `kindset/node` image.

```sh
$ docker ps
container id image command created status ports names
8a885e2d2b28 kindest/node:v1.27.1 "/usr/local/bin/entr..."   2 minutes ago Up 2 minutes 127.0.0.1:58181->6443/tcp kind-control-plane
```

## Setup LoadBalancer for kind with metalLB

### Setup routing table for docker network

* Reference: https://opencredo.com/blogs/building-the-best-kubernetes-test-cluster-on-macos/

Refer to the blog above and proceed with the setup for macbook and colima machine. (You can automatically perform the above setup by executing the command below using the script I wrote).

```sh
curl -L https://gist.githubusercontent.com/evir35/6fbd7382629e60bef08ac39e1968f2df/raw/c638578991b4a8b16304dde327f9621dae2bc03c/colima-network-setup.sh 2>/dev/null | bash -- <&1
```

### Install metalLB on kind cluster

Execute the command below to install metalLB on kind cluster ([reference](https://kind.sigs.k8s.io/docs/user/loadbalancer/)).

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=90s
```

### Check the Docker kind network IP range

If you execute the command below, you can see that the `172.18.0.0/16` IP range is available on the kind network in my local case.

```sh
$ docker network inspect -f '{{.IPAM.Config}}' kind
[{172.18.0.0/16 172.18.0.1 map[]} {fc00:f853:ccd:e793::/64 map[]}]
```

### Set the MetalLB IP range

```sh
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250 # Please update these addresses to your IP range
---]
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```

### Create an example resource using the MetalLB LoadBalancer

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

Check the IP of the LB in the created Service. In my case, the LB was created with the IP `172.18.255.200`.

```sh
kubectl get service
name type cluster-ip external-ip port(s) age
foo-service LoadBalancer 10.96.220.49 172.18.255.200 80:30790/TCP 15d
kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 15d
```

If you curl to those IPs, you'll see the hostnames alternating as shown below.

```sh
$ curl 172.18.255.200
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from bar-app</h1></body></html
$ curl 172.18.255.200
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from foo-app</h1></body></html
```

## Conclusion

I built a k8s cluster locally using colima and kind, and configured it to utilize a Loadbalancer Type service using metalLB. It's a bit unfortunate that I still need to configure network-related settings, but I think it's a good way to run k8s locally for a simple test, delete it after testing, and recreate it.

## References

* colima: <https://github.com/abiosoft/colima>
* kind: <https://kind.sigs.k8s.io/>
* metallb: <https://metallb.universe.tf/>
* Building the best Kubernetes test cluster on MacOS: <https://opencredo.com/blogs/building-the-best-kubernetes-test-cluster-on-macos/>

## ETC

### Errors with colima

While creating a kubernetes cluster using kind, I encountered the following error.

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

This appears to be a related [issue](https://github.com/kubernetes-sigs/kind/issues/3277), caused by the lack of support for cgroup v2 in VMs created by colima. I saw a [comment](https://github.com/kubernetes-sigs/kind/issues/3277#issuecomment-1643943889) that the error does not occur `in 0.19.0`, where the relevant PRs have not been merged, so I installed the `0.19.0` binary myself and a kind cluster was created without issue.
