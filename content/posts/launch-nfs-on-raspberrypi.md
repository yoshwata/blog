---
title: "Launch NFS on Raspberrypi"
date: 2019-05-05T11:12:16+09:00
draft: false
toc: false
images:
tags:
  - raspberrypi
  - raspbian
---

### Introduction

NFS is very useful tool to mount volumes and share files. That is also used as a PersistentVolume of Kubernetes, so it is meaningfull that to know how to launch nfs server on your RaspberryPi on recent k3s movement.

### Install NFS

If you use raspbian, you just execute a command.
```bash
$ sudo apt-get install -y nfs-common nfs-server
```

Please make directory to be mounted.
```bash
$ sudo mkdir /mnt/share
$ sudo chmod -R 777 /mnt/share
```

Then, add setting to `/etc/exports`
```bash
$ echo '/mnt/share 192.168.123.0/24(rw,async,no_root_squash)' >> /etc/exports
```
Please check appropriate IP on your cluster.

`async` means write data to the volume asyncly. `no_root_squash` means disable the feature of `root_squash`. `root_squash` means that it blocks to get root priviledges by remote access users. `no_root_squash` is not secure, but sometimes it blocks to change existing files. Please consider about what is the best setting for your system.

OK, then start the server.
```bash
$ systemctl start rpcbind nfs-server
```

It is optional, but you can set auto start with this command.
```bash
$ systemctl enable rpcbind nfs-server
```

You can check nfs works or not with this command.
```
$ exportfs
```
It will show error if something wrongs.

### Mount from another host

Please log in to another host and mount the volume with this command.
```
sudo mount -t nfs -o resvport,rw 192.168.123.11:/mnt/share /home/yoshwata/mnt
```

Then, check the nfs works by putting a file from remote host.


