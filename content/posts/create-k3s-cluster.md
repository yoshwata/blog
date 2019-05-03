---
title: "Create k3s Cluster"
date: 2019-05-03T10:45:43+09:00
draft: false
toc: false
images:
tags:
  - raspberrypi
  - kubernetes
---

### Introduction

k3s is a light weight Kubernetes distribution. That works with only 40MB binary and 512MB memory consumption.

It seems to be reasonable to install my ARM system or Raspberry pi, then I find that k3s is great tool for getting personal kubernetes cluster.

I introduce about the procedure to create cluster.

What you need is

- Two or more RaspberryPi
- SD cards [installed Raspbian](https://yoshwata-blog.netlify.com/posts/write-raspbian-lite/) and enabled SSH

### Install and launch k3s master

Log in to your master node with ssh and check the k3s release from here.  
https://github.com/rancher/k3s/releases

I recommend to use latest release.
Then hit the command.
```bash
$ wget https://github.com/rancher/k3s/releases/download/{{ k3s_version }}/k3s-armhf && \
chmod +x k3s-armhf && \
sudo mv k3s-armhf /usr/local/bin/k3s
```

Let's start your k3s server.

```bash
$ sudo k3s server
```

Your terminal will taken by k3s server, so you can't execute command anymore. Please login with another ssh session.

After login, k3s server create token to join workers to cluster. This token will be used later.

```bash
$ sudo cat /var/lib/rancher/k3s/server/node-token
```

### Instal and launch k3s worker

Let's log into your worker node with ssh.

What you should do in your node is below.
`<Server_IP>` is your master's IP, and `<NODE_TOKEN>` is the token taken with previous step.

```bash
$ sudo k3s agent -s <SERVER_IP> -t <NODE_TOKEN>
```

Creating k3s cluster is done. Please check your nodes after logging in to your master node.
```bash
$ sudo k3s kubectl get nodes
NAME    STATUS   ROLES    AGE     VERSION
pi001   Ready    <none>   4d16h   v1.14.1-k3s.4
pi002   Ready    <none>   4d17h   v1.14.1-k3s.4
pi003   Ready    <none>   4d13h   v1.14.1-k3s.4
```

### (Optional) Launch k3s automatically on boot

It's too much hassle to launching server and agent by hand every time. If you want to save time, write some scripts to `rc.local`.

This is master example. Please write it to `/etc/rc.local` to your master node.
```bash

#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

sudo k3s server &

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
  fi
  
exit 0
```

Also, this is worker example.
```bash

#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

sudo k3s agent -s {{ target }} -t {{ token }} &

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
  fi
  
exit 0
```

