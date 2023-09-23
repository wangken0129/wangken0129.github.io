---
title: Guide_RKE2&Rancher_install
description: Guide_RKE2&Rancher_install
slug: Guide_RKE2&Rancher_install
date: 2023-09-23T06:57:16+08:00
categories:
    - Knowledge Base Category
tags:
    - SUSE
    - Rancher
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# RKE2 and Rancher install

## 0. Install DNS

```shell=
sudo zypper in -t pattern dhcp_dns_server
```

## 1. login
```shell=
sam@sam:~> ssh rancher@192.168.122.41
Password:
Last failed login: Wed Sep 21 08:33:47 CST 2022 from 192.168.122.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Wed Sep 21 08:31:58 2022

rancher@rms1:~> curl -sfL https://get.rke2.io --output install.sh
rancher@rms1:~> chmod +x install.sh
```

## 2. config rke2 basic parameters
```shell=
rancher@rms1:~> sudo mkdir -p /etc/rancher/rke2/
[sudo] root 的密碼：
rancher@rms1:~> sudo vim /etc/rancher/rke2/config.yaml
rancher@rms1:~> cat /etc/rancher/rke2/config.yaml
node-name:
  - "rms1"
token: my-shared-secret
```

## 3. install rke2 with 1.23.9
```shell=
rancher@rms1:~> sudo INSTALL_RKE2_CHANNEL=v1.23.9+rke2r1 ./install.sh
[WARN]  /usr/local is read-only or a mount point; installing to /opt/rke2
[INFO]  finding release for channel v1.23.9+rke2r1
[INFO]  using v1.23.9+rke2r1 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.23.9+rke2r1/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.23.9+rke2r1/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /opt/rke2
[INFO]  updating tarball contents to reflect install path
[INFO]  moving systemd units to /etc/systemd/system
[INFO]  install complete; you may want to run:  export PATH=$PATH:/opt/rke2/bin
rancher@rms1:~> export PATH=$PATH:/opt/rke2/bin
```

## 4. enable rke2 and setup kubeconfig
```shell=
rancher@rms1:~> sudo systemctl enable rke2-server
Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /etc/systemd/system/rke2-server.service.
rancher@rms1:~> sudo systemctl start rke2-server
rancher@rms1:~> mkdir .kube
rancher@rms1:~> sudo cp /etc/rancher/rke2/rke2.yaml .kube/config
[sudo] root 的密碼：
rancher@rms1:~> sudo chown rancher .kube/config
rancher@rms1:~> sudo cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/
```

## 5. check pod status
```shell=
rancher@rms1:~> kubectl get po
No resources found in default namespace.
rancher@rms1:~> kubectl get po -A
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE
kube-system   cloud-controller-manager-rms1                         1/1     Running     0          15m
kube-system   etcd-rms1                                             1/1     Running     0          15m
kube-system   helm-install-rke2-canal-6bpd4                           0/1     Completed   0          15m
kube-system   helm-install-rke2-coredns-mjflj                         0/1     Completed   0          15m
kube-system   helm-install-rke2-ingress-nginx-76r2c                   0/1     Completed   0          15m
kube-system   helm-install-rke2-metrics-server-wkc4k                  0/1     Completed   0          15m
kube-system   kube-apiserver-rms1                                   1/1     Running     0          15m
kube-system   kube-controller-manager-rms1                          1/1     Running     0          15m
kube-system   kube-proxy-rms1                                       1/1     Running     0          15m
kube-system   kube-scheduler-rms1                                   1/1     Running     0          15m
kube-system   rke2-canal-8x56p                                        2/2     Running     0          15m
kube-system   rke2-coredns-rke2-coredns-545d64676-zlnk9               1/1     Running     0          15m
kube-system   rke2-coredns-rke2-coredns-autoscaler-5dd676f5c7-zrdbb   1/1     Running     0          15m
kube-system   rke2-ingress-nginx-controller-xhxr6                     1/1     Running     0          14m
kube-system   rke2-metrics-server-6564db4569-542hx                    1/1     Running     0          14m
```

## 6. install helm3
```shell=
rancher@rms1:~> wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
--2022-09-21 09:06:57--  https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
Resolving get.helm.sh (get.helm.sh)... 152.199.39.108, 2606:2800:247:1cb7:261b:1f9c:2074:3c
Connecting to get.helm.sh (get.helm.sh)|152.199.39.108|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 13633605 (13M) [application/x-tar]
Saving to: ‘helm-v3.8.2-linux-amd64.tar.gz’

helm-v3.8.2-linux-amd64.tar.gz     100%[=============================================================>]  13.00M  5.95MB/s    in 2.2s

2022-09-21 09:07:00 (5.95 MB/s) - ‘helm-v3.8.2-linux-amd64.tar.gz’ saved [13633605/13633605]

rancher@rms1:~> tar zxvf helm-v3.8.2-linux-amd64.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
rancher@rms1:~> ls
bin  helm-v3.8.2-linux-amd64.tar.gz  install.sh  linux-amd64  public_html
rancher@rms1:~> sudo cp linux-amd64/helm /usr/local/bin/
[sudo] root 的密碼：
rancher@rms1:~> helm --help


The Kubernetes package manager

Common actions for Helm:

- helm search:    search for charts
- helm pull:      download a chart to your local directory to view
- helm install:   upload the chart to Kubernetes
- helm list:      list releases of charts
...
...
...
```


## 7. install rancher and cert-manager
```shell=
rancher@rms1:~> helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
"rancher-stable" has been added to your repositories
rancher@rms1:~> kubectl create namespace cattle-system
namespace/cattle-system created
rancher@rms1:~> kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created


rancher@rms1:~> helm repo add jetstack https://charts.jetstack.io

"jetstack" has been added to your repositories

rancher@rms1:~> helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "rancher-stable" chart repository
...Successfully got an update from the "jetstack" chart repository
Update Complete. ⎈Happy Helming!⎈
rancher@rms1:~> helm install cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--version v1.7.1

NAME: cert-manager
LAST DEPLOYED: Wed Sep 21 09:11:15 2022
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.7.1 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/

rancher@rms1:~> kubectl get pods --namespace cert-manager
NAME                                     READY   STATUS    RESTARTS   AGE
cert-manager-76d44b459c-zhpp2            1/1     Running   0          32s
cert-manager-cainjector-9b679cc6-6tzd8   1/1     Running   0          32s
cert-manager-webhook-57c994b6b9-4dfvs    1/1     Running   0          32s

rancher@rms1:~> helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.example.com --version 2.6.6
NAME: rancher
LAST DEPLOYED: Wed Sep 21 09:14:06 2022
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

If you provided your own bootstrap password during installation, browse to https://rancher.example.com to get started.

If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:


echo https://rancher.example.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')


To get just the bootstrap password on its own, run:


kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'


Happy Containering!
```

## 8. check rancher status 
```shell=
rancher@rms1:~> kubectl -n cattle-system get po
NAME                       READY   STATUS              RESTARTS   AGE
rancher-7fd65d9cd6-8krrq   0/1     ContainerCreating   0          16s
rancher-7fd65d9cd6-h28fw   0/1     ContainerCreating   0          16s
rancher-7fd65d9cd6-k9hrr   0/1     ContainerCreating   0          16s

rancher@rms1:~> watch kubectl -n cattle-system get po

rancher@rms1:~> kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment spec update to be observed...
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "rancher" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "rancher" rollout to finish: 2 of 3 updated replicas are available...
deployment "rancher" successfully rolled out

rancher@rms1:~> kubectl -n cattle-system get po
NAME                       READY   STATUS    RESTARTS      AGE
rancher-7fd65d9cd6-8krrq   1/1     Running   1 (51s ago)   3m11s
rancher-7fd65d9cd6-h28fw   1/1     Running   0             3m11s
rancher-7fd65d9cd6-k9hrr   1/1     Running   1 (51s ago)   3m11s
```

## notice
1. must use DNS for rancher.
2. downstream must resolve rancher portal from dns.
3. good luck.