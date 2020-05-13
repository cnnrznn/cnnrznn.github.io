---
layout: single
title:  "Bootstrapping a Kubernetes Cluster"
header:
  teaser:
categories: 
  - distributed systems
  - infrastructure
---

After beginning the [install
guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
for kubernetes, I noticed disparities between my Ubuntu desktop machine and
the guides. In this post, I will walk you through what I did to bootstrap a
kubernetes cluster on my Ubuntu 18.04 (bionic) desktop machine.

## Install Docker
Kubernetes does not directly manage containers. You must choose a container
runtime that kubernetes will use for managing containers. Thanks to the [Open
Container Initiative](https://www.opencontainers.org/), the interfaces for
container runtimes have been standardized and are moving in the direction of
being plug-and-play.

I chose Docker because its familiar and have it already installed on my
machine. For a fresh install, follow the instructions
[here](https://docs.docker.com/engine/install/ubuntu/). My docker version writing this was
```shell
cnnrznn@cfz:~/source/cnnrznn.github.io/_drafts$ docker version
Client: Docker Engine - Community
 Version:           19.03.5
 API version:       1.40
 Go version:        go1.12.12
 Git commit:        633a0ea838
 Built:             Wed Nov 13 07:50:12 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.5
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.12
  Git commit:       633a0ea838
  Built:            Wed Nov 13 07:48:43 2019
  OS/Arch:          linux/amd64
  Experimental:     true
 containerd:
  Version:          1.2.10
  GitCommit:        b34a5c8af56e510852c35414db4c1f4fa6172339
 runc:
  Version:          1.0.0-rc8+dev
  GitCommit:        3e425f80a8c931f88e6d94a8c831b9d5aa481657
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
cnnrznn@cfz:~/source/cnnrznn.github.io/_drafts$ 
```

## Install `kubeadm`, `kubectl`, and `kubelet`
This is where the install process started to differ for me. There
instructions for installing these tools did not work.

```shell
No apt package "kubelet", but there is a snap with that name.
Try "snap install kubelet"
```

So I did just that.

```shell
snap install --classic kubelet
snap install --classic kubeadm
snap install --classic kubectl
```

The `--classic` flag is necessary for installing these tools as they contain
security vulnerabilities.

## Creating a single control-plane cluster
Run the following to initialize the control-plane process.
```shell
kubeadm init
```

For me, this did not initially work, complaining:
```shell
cnnrznn@cfz:~$ sudo kubeadm init 
W0513 15:33:51.878270   23730 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.2
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
	[WARNING FileExisting-ebtables]: ebtables not found in system path
	[WARNING FileExisting-socat]: socat not found in system path
	[WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileExisting-conntrack]: conntrack not found in system path
	[ERROR Port-10250]: Port 10250 is in use
```

Let's investigate each of these.

- `detected cgroupfs`: This one was straightforward. In
`/etc/docker/daemon.json`, add this line: `exec-opts":
["native.cgroupdriver=systemd"],"` to the json object
- 