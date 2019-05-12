---
title: "Skaffold to k3s RaspberryPi Cluster"
date: 2019-05-12T12:29:36+09:00
draft: false
---

### Introduction

Skaffold is a useful tool for deploying to Kubernetes. You can deploy many Kubernetes resources with one Skaffold command even if it is Helm or Kastomize definition. In cluster for development, it is works as resident process then, the cluster reacted each time you change the local definition of Kubernetes resources.

### Installing Skaffold

Please get source code first. An error message appears but it is no problem.
```bash
$ go get github.com/GoogleCloudPlatform/skaffold
can't load package: package github.com/GoogleCloudPlatform/skaffold: no Go files in /Users/yoshwata/go/src/github.com/GoogleCloudPlatform/skaffold
```

Go to your $GOPATH and make it.
```bash
$ cd $GOPATH/src/github.com/GoogleCloudPlatform/skaffold
$ make install
```

Then Skaffold is installed. Please check the version.
```bash
$ skaffold version
v0.29.0-5-g06d01b12
```

### Deploy to k3s raspi cluster

Skaffold has much of example to show the functionality, so we use it.

```bash
$ cd examples/getting-started
```

Edit your `Dockerfile` like this.

```diff
index 184d6cce..ef2ec686 100644
--- a/examples/getting-started/Dockerfile
+++ b/examples/getting-started/Dockerfile
@@ -1,7 +1,7 @@
 FROM golang:1.10.1-alpine3.7 as builder
 COPY main.go .
-RUN go build -o /app main.go
+RUN GOOS=linux GOARCH=arm GOARM=7 go build -o /app main.go

-FROM alpine:3.7
+FROM arm32v7/alpine
 CMD ["./app"]
 COPY --from=builder /app .
```
These change make compatible of ARM cluster.

Modify the pod definition. Please check your username of `docker.io`.

```diff
index 14cc6911..4f1531b4 100644
--- a/examples/getting-started/k8s-pod.yaml
+++ b/examples/getting-started/k8s-pod.yaml
@@ -5,4 +5,6 @@ metadata:
 spec:
   containers:
   - name: getting-started
+    image: <USERNAME>/skaffold-example:v0.29.0-5-g06d01b12-dirty
```

This is a definition to pushing to `docker.io`.
```diff
index 3eedc115..aa3ca0fa 100644
--- a/examples/getting-started/skaffold.yaml
+++ b/examples/getting-started/skaffold.yaml
@@ -2,7 +2,9 @@ apiVersion: skaffold/v1beta10
 kind: Config
 build:
   artifacts:
+  - image: skaffold-example
 deploy:
   kubectl:
     manifests:
```

Then execute this command.
```bash
$ skaffold dev --default-repo docker.io/<YOURNAME>
```

### Check what you deployed

All preparation is done, so let's check the pod working.

```bash
$ kubectl get po
NAME              READY   STATUS    RESTARTS   AGE
getting-started   1/1     Running   0          10m
```

You can see what the pod saying with kubectl logs.

```bash
$ kubectl logs getting-started -f
Hello world!
Hello world!
Hello world!
Hello world!
```

Skaffold process also said same `Hello world` things sometimes, but now I can not find it somehow.
