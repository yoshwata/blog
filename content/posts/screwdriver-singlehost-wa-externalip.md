---
title: "Screwdriver Singlehost Workaround of ExternalIP"
date: 2019-06-23T10:01:04+09:00
draft: false
toc: false
images:
tags:
  - Kubernetes
  - Screwdriver.cd
---

### Overview
[Screwdriver.cd](https://screwdriver.cd/) is a one of a CI/CD tool which woks on Kubernetes. You can set builds with easy `yaml` definition with this tool.

Screwdriver.cd has a Helm chart, but this needs workaround to make it works with single host.
I'd like to introduce how to deploy it with single host mode and with Helm chart.

### Registering OAuth apps
Screwdriver.cd works with GitHub, so please register it as an OAuth app from [here](https://github.com/settings/developers).
Authorization callback URL should be like below.
```
http://<YOUR_HOST>:9001/v4/auth/login
```
Other settings like Application name is not so severe that you can set them what you like.

### Modify scm-settings
Please clone [this](https://github.com/yoshwata/screwdriver-chart.git) and set branch `wa-external-ip`.
This branch is what I modified to make Screwdriver.cd work with single host.

You can find a file `example-scm-settings.json` from cloned repository. Please edit `oauthClientId` and `oauthClientSecret` with
what you get when you register OAuth app. The part of `secret`, which will be checked the length of string, should be like `SUPER-SECRET-SIGNING-THING` or so.

### Generate secrets
To generate secret file for k8s, you can use script `generate_secrets.sh` of the repo.
```bash
$ ./generate_secrets.sh
```
This script JWT key pair and manifest files of secrets which includes contents of `example-scm-settings.json` that you made before.

Then apply it.
```bash
$ kubectl apply -f screwdriver-api-secrets.yaml
```

### Deploying Screwdriver.cd
You can deply Screwdriver.cd with this command.
Please modify <IP_ADDRESS> part with your master IP of your k8s cluster.
```bash
$ helm dependency update
$ helm install . --name my-screwdriver --namespace test-screwdriver --set api.externalIPs=<IP_ADDRESS> --set ingress.hosts.api=<YOUR_HOST> --set ingress.hosts.ui=<YOUR_HOST> --set ingress.hosts.store=<YOUR_HOST>
```

Please check components(api/ui/store) of Screwdriver.cd are running. Screwdriver.cd api will restert two or three times until db start completely.
```
NAMESPACE          NAME                                        READY   STATUS      RESTARTS   AGE
kube-system        coredns-695688789-hqbgl                     1/1     Running     0          28d
kube-system        helm-install-traefik-gb69m                  0/1     Completed   0          28d
kube-system        svclb-traefik-7mm9m                         2/2     Running     0          28d
kube-system        svclb-traefik-968wz                         2/2     Running     0          16d
kube-system        tiller-deploy-598f58dd45-962hs              1/1     Running     0          3h54m
kube-system        traefik-55bd9646fc-l975b                    1/1     Running     0          28d
test-screwdriver   my-screwdriver-postgresql-666494b9f-rdtdd   1/1     Running     0          11m
test-screwdriver   prod-sdapi-dfb567575-7tgdb                  1/1     Running     2          11m
test-screwdriver   prod-sdstore-78fcc5f587-6wp6g               1/1     Running     0          11m
test-screwdriver   prod-sdui-987d556f8-2btdp                   1/1     Running     0          11m
```

Let's build with Screwdriver.cd.

Please put easy `screwdriver.yaml` file to your repository.
```yaml
shared:
  image: node:8
jobs:
  main:
    requires: [ "~pr", "~commit" ]
    steps:
      - echo: echo 'foo'
```

You can access Screwdriver.cd with `http://<YOUR_HOST>:9000`.
Click  `Create Pipeline` button and set the repository of your `screwdriver.yaml`

Then please push `Start` button of your pipeline screen.
A build will start and you can see the build is works as your `screwdriver.yaml`.

### Supplements about this workaround
I will add some comments to [this PR](https://github.com/yoshwata/screwdriver-chart/pull/1/files).
This workaround is not use `ingress` but `externalIP`. If you would like to use `ingress` and path base routing, please check [this issue](https://github.com/screwdriver-cd/screwdriver/issues/1661).
You need to modify source codes of Screwdriver.cd to use `ingress`.
