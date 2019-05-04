---
title: "Install Helm to ARM k3s cluster"
date: 2019-05-04T21:43:09+09:00
draft: false
toc: false
images:
tags:
  - k3s
  - raspberrypi
---

### Introduction

Helm is a tool for deploying apps to Kubernetes. It is time consuming that deploying a bunch of pods by hand, but Helm is very useful for such situation. Helm enables users to deploy many resources Kubernetes at once. It also works like "apt" or "yum" because you can load Helm charts that made by someone else from Helm repository.

This page introduces that how to install Helm to your ARM k3s cluster.

### Install helm cli

It is important that download an appropriate binary for your system. You can find a bunch of binaries from this page.  
https://github.com/helm/helm/releases

What you should download is `Linux arm` version. This is example of download command.

```bash
$ curl -O https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-arm64.tar.gz
```
Please check the latest version before you download.

Then, unarchive and move it.
```bash
$ tar -zxvf helm-v2.13.1-linux-arm64.tar.gz
$ mv linux-arm/helm /usr/local/bin/helm
```

### Install tiller

Tiller is a serer component of Helm. Tiller deploys resources by your instruction of helm CLI. It is necessary to append authority for the behavior.

First, add `tiller` service account.
```bash
kubectl -n kube-system create serviceaccount tiller
```

Then, give `cluster-admin` role to `tiller`
```bash
$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```
This binding is too redundant, so it might be better to constrain it.

Initialize Helm
```bash
$ helm init --service-account tiller
```

Then, tiller is deployed. You can check it with kubectl.
```bash
$ k get po -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-857cdbd8b4-vskmp         1/1     Running   12         6d3h
svclb-traefik-64f7684b86-896zs   3/3     Running   38         6d3h
svclb-traefik-64f7684b86-t9dcs   3/3     Running   23         5d12h
tiller-deploy-564579f564-wgqfp   1/1     Running   5          4d7h
traefik-55bd9646fc-xqq65         1/1     Running   3          4d
```

### Test

You can use helm command like this.
```bash
$ helm install stable/mysql
```

Then some resources for mysql will deployed to your cluster.

To delete all resources related with installed chart, execute command like this.
```bash
$ helm del --purge punk-octopus
```

`--purge` option deletes the chart from local store. You can see list of charts what you've downloaded by `helm list`.


