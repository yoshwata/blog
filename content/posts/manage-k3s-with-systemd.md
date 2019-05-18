---
title: "Manage k3s With systemd"
date: 2019-05-18T14:14:07+09:00
draft: false
toc: false
images:
tags:
  - untagged
---

### Introduction

[This post](https://yoshwata-blog.netlify.com/posts/create-k3s-cluster/) shows a way to install k3s manually and add `rc.local` script for automatic launch. I found that [k3s install script](https://github.com/rancher/k3s/blob/master/install.sh) is more useful because it supports `systemd`, so I introduce it.

### Prerequisites
- [2 or more Raspberry pi installed Raspbian lite](https://yoshwata-blog.netlify.com/posts/write-raspbian-lite/)

### Install master

Installing master is pretty easy. It is just hit this command.
```bash
$ curl -sfL https://get.k3s.io | sh -
```
This script detects what `k3s` binary should be installed to your system. It also registers the master process to systemd, so master process will be launched in this script.

If you can not find master process in `ps` command, please execute the command to start server from systemd.
```bash
$ sudo systemctl restart k3s
```

Check your node token before you go to next step.
```bash
$ sudo cat /var/lib/rancher/k3s/server/node-token
```

### Install node

Please log in to your node machine with ssh.

Installing node is tricky a little. If you want the Raspi work as node, you need to give some environments to the install script.
Check your ip address of your master and node token you got previous step.
```bash
$ curl -sfL https://get.k3s.io | K3S_URL=https://{{ master_ip }}:6443 K3S_TOKEN={{ token }} sh -
```

If the agent process doesn't start automatically, please execute this command.
```bash
$ sudo systemctl restart k3s-agent
```

The install script seems to be arranged for master, so disable master process not to launch it on reboot.
```bash
$ sudo systemctl disable k3s
```

### Check cluster

Please back to your master node and check all of your nodes become a member of the cluster.
```bash
$ sudo k3s kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
pi001   Ready    <none>   12d   v1.14.1-k3s.4
pi002   Ready    <none>   12d   v1.14.1-k3s.4
pi003   Ready    <none>   12d   v1.14.1-k3s.4
```

All node will back to cluster even if you reboot it because systemd launches k3s server or agents.

