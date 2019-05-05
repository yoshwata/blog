---
title: "Launch Plex on RaspberryPi k3s Cluster"
date: 2019-05-05T11:02:59+09:00
draft: false
toc: false
images:
tags:
  - k3s
  - kubernetes
  - helm
  - raspberrypi
---

### Introduction

Plex is a media server works on Linux. You can enjoy movie or music from Plex, and the UI is very useful. This works as DLNA server too, and the setting is very easy.

If you buy PlesPass, which is paid version of Plex, additional function is unlocked for you. However, free version is also enough for the basic funcionality.

People may operate Plex on docker, but operating it on Kubernetes is meaningful for the users who operate many apps other than Plex. You can mange many apps in one platform Kubernetes which has many nice feature like resource management, scaling and so forth.

Please refer below about Plex.  
https://www.plex.tv/

### Prerequisites

- [Setting of nfs](http://localhost:1313/posts/launch-nfs-on-raspberrypi/)
 
### Clone and modify kube-plex

First, clone this repository.
```
$ git clone https://github.com/munnerz/kube-plex.git
```

Then please modify like this.

https://github.com/yoshwata/kube-plex/blob/53f29d9079d94fc96b4ca7f95eb294a0cbd02c48/charts/kube-plex/templates/deployment.yaml#L88

Add the `VERSION` key and `latest` value. This means that the deployment uses latest version of plex.

Next please add annotation like this.  
https://github.com/yoshwata/kube-plex/blob/53f29d9079d94fc96b4ca7f95eb294a0cbd02c48/charts/kube-plex/templates/volumes.yaml#L28

I think that the `slow` value can change anyting. I can not apply my k3s without this annotation and I don't know why.

Then, modify like this.  
https://github.com/yoshwata/kube-plex/blob/53f29d9079d94fc96b4ca7f95eb294a0cbd02c48/charts/kube-plex/values.yaml#L63

The line of `arm` is because this chart deploy only `amd64` archtecture, so Plex never deployed on your ARM cluster without this setting.

`disktype` line is optional. I use this selector because only one node has huge capasity of disk, and the disk is mounted for media data. By this setting Plex never deployed other nodes that has not much of capasity.

If you use `disktype` selecter, don't forget to set label to your node like.
```
$ kubectl label nodes <nodename> distype=huge
```

### Apply Persistent Volume

Please make these files after checking `<YOUR-NFS-IP>`.

`nfs-pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs001
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs"
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: <YOUR-NFS-IP>
    path: /mnt/share/
```

`nfs-claim.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-claim1
  namespace: plex
  annotations:
    "volume.beta.kubernetes.io/storage-class": "nfs"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

`plex-config-pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: plex-config-pv
  annotations:
    volume.beta.kubernetes.io/storage-class: "slow"
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/var/plex/config"
```

Then apply all.
```bash
$ kubectl apply -f <filename>
```

Please check `nfs-claim1` pvc is bound on `nfs-pv`.
```bash
$ kubectl get pvc -n plex
```

### Launch plex

Move direcoty to the top of kube-plex repository, then execute this command.
```bash
$ helm install ./charts/kube-plex --name plex --namespace plex --set claimToken="claim-<token>" --set image.repository=linuxserver/plex --set image.tag=arm32v7-latest --set persistence.data.claimName=nfs-claim1 --set persistence.config.size=2Gi --set kubePlex.image.repository=yoshwata/kube-plex_linux_arm7l --set service.type=ClusterIP --set service.externalIPs[0]="<IP of the node Plex launch>"
```

Helm can install chart with your favorite option by useing `--set` option. Please specify the value like `claimToken` or `externalIPs`.

`yoshwata/kube-plex_linux_arm7l` is arm version of kube-plex. Now I do not CICD about this image, so please rebuild yourself when it bacome obsolate.

Then you can access to plex with blowser.

```
http://<externalIP>/web/index.html
```

`kube-plex` has another feature to delegete transcode jobs to other pods. I will introduce about that after it goes well.

