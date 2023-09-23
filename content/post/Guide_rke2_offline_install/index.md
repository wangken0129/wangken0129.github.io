---
title: Guide_rke2_offline_install
description: Guide_rke2_offline_install
slug: Guide_rke2_offline_install
date: 2023-09-23T07:02:38+08:00
categories:
    - Knowledge Base Category
tags:
    - SUSE
    - Rancher
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# rke2 offline install

## 1. 下載1.24.6離線安裝所需image
```shell=
rancher@rms1:~> sudo mkdir /root/rke2-artifacts && cd /root/rke2-artifacts/
mkdir: cannot create directory ‘/root/rke2-artifacts’: File exists
rancher@rms1:~> sudo su
rms1:/home/rancher # cd /root/rke2-artifacts/
rms1:~/rke2-artifacts # ll
total 0
rms1:~/rke2-artifacts # curl -OLs https://github.com/rancher/rke2/releases/download/v1.24.6%2Brke2r1/rke2-images.linux-amd64.tar.zst
rms1:~/rke2-artifacts # curl -OLs https://github.com/rancher/rke2/releases/download/v1.24.6%2Brke2r1/rke2.linux-amd64.tar.gz
rms1:~/rke2-artifacts # curl -OLs https://github.com/rancher/rke2/releases/download/v1.24.6%2Brke2r1/sha256sum-amd64.txt
rms1:~/rke2-artifacts # curl -sfL https://get.rke2.io --output install.sh
rms1:~/rke2-artifacts # ll
total 823052
-rw-r--r-- 1 root root     21438 Oct 11 20:19 install.sh
-rw-r--r-- 1 root root 794531974 Oct 11 20:17 rke2-images.linux-amd64.tar.zst
-rw-r--r-- 1 root root  48238609 Oct 11 20:18 rke2.linux-amd64.tar.gz
-rw-r--r-- 1 root root      3626 Oct 11 20:18 sha256sum-amd64.txt
```

## 2. 解壓縮與基本設定
```shell=
rms1:~/rke2-artifacts # INSTALL_RKE2_ARTIFACT_PATH=/root/rke2-artifacts sh install.sh
[WARN]  /usr/local is read-only or a mount point; installing to /opt/rke2
[INFO]  staging local checksums from /root/rke2-artifacts/sha256sum-amd64.txt
[INFO]  staging zst airgap image tarball from /root/rke2-artifacts/rke2-images.linux-amd64.tar.zst
[INFO]  staging tarball from /root/rke2-artifacts/rke2.linux-amd64.tar.gz
[INFO]  verifying airgap tarball
grep: /tmp/rke2-install.XRvs61dJ7e/rke2-images.checksums: No such file or directory
[INFO]  installing airgap tarball to /var/lib/rancher/rke2/agent/images
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /opt/rke2
[INFO]  updating tarball contents to reflect install path
[INFO]  moving systemd units to /etc/systemd/system
[INFO]  install complete; you may want to run:  export PATH=$PATH:/opt/rke2/bin
rms1:~/rke2-artifacts # export PATH=$PATH:/opt/rke2/bin
```

## 3. 叢集基礎組態

```shell=
rms1:~/rke2-artifacts # mkdir -p /etc/rancher/rke2/
[sudo] root 的密碼：
rms1:~/rke2-artifacts # vim /etc/rancher/rke2/config.yaml
rms1:~/rke2-artifacts # cat /etc/rancher/rke2/config.yaml
node-name:
  - "rms1"
token: my-shared-secret
```

## 4. 啟用RKE2服務

```shell=
rms1:~/rke2-artifacts # systemctl enable --now rke2-server
Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /etc/systemd/system/rke2-server.service.
```

## 5. 設定一般帳號使用kubectl
```shell=
rms1:~/rke2-artifacts # exit
exit
rancher@rms1:~> mkdir .kube
rancher@rms1:~> sudo cp /etc/rancher/rke2/rke2.yaml .kube/config
[sudo] password for root:
rancher@rms1:~> sudo chown rancher .kube/config
rancher@rms1:~> sudo cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/
rancher@rms1:~> kubectl get po -A
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE
kube-system   cloud-controller-manager-rms1                           1/1     Running     0          3m5s
kube-system   etcd-rms1                                               1/1     Running     0          2m46s
kube-system   helm-install-rke2-canal-lpv22                           0/1     Completed   0          3m
kube-system   helm-install-rke2-coredns-xndpj                         0/1     Completed   0          3m
kube-system   helm-install-rke2-ingress-nginx-5r5sq                   0/1     Completed   0          3m
kube-system   helm-install-rke2-metrics-server-wkz6c                  0/1     Completed   0          3m
kube-system   kube-apiserver-rms1                                     1/1     Running     0          2m37s
kube-system   kube-controller-manager-rms1                            1/1     Running     0          2m30s
kube-system   kube-proxy-rms1                                         1/1     Running     0          2m57s
kube-system   kube-scheduler-rms1                                     1/1     Running     0          2m37s
kube-system   rke2-canal-clqp4                                        2/2     Running     0          2m41s
kube-system   rke2-coredns-rke2-coredns-76cb76d66-xpnqg               1/1     Running     0          2m42s
kube-system   rke2-coredns-rke2-coredns-autoscaler-58867f8fc5-hzz2l   1/1     Running     0          2m42s
kube-system   rke2-ingress-nginx-controller-zfbvx                     1/1     Running     0          93s
kube-system   rke2-metrics-server-6979d95f95-kmbcv                    1/1     Running     0          109s
```

## 6. 加入其他節點

```shell=
```

## 7. 安裝helm

```shell=
rancher@rms1:~> wget https://get.helm.sh/helm-v3.9.4-linux-amd64.tar.gz
--2022-10-11 20:33:57--  https://get.helm.sh/helm-v3.9.4-linux-amd64.tar.gz
Resolving get.helm.sh (get.helm.sh)... 152.199.39.108, 2606:2800:247:1cb7:261b:1f9c:2074:3c
Connecting to get.helm.sh (get.helm.sh)|152.199.39.108|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14026634 (13M) [application/x-tar]
Saving to: ‘helm-v3.9.4-linux-amd64.tar.gz’

helm-v3.9.4-linux-amd64.tar.gz  100%[======================================================>]  13.38M  11.0MB/s    in 1.2s

2022-10-11 20:34:00 (11.0 MB/s) - ‘helm-v3.9.4-linux-amd64.tar.gz’ saved [14026634/14026634]

rancher@rms1:~> tar zxvf helm-v3.9.4-linux-amd64.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
rancher@rms1:~> sudo cp linux-amd64/helm /usr/local/bin/
```

## 8. 啟用cert-manager(optional)
```shell=
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
...Successfully got an update from the "jetstack" chart repository
Update Complete. ⎈Happy Helming!⎈
rancher@rms1:~> helm install cert-manager jetstack/cert-manager \
> --namespace cert-manager \
> --create-namespace \
> --version v1.7.1

NAME: cert-manager
LAST DEPLOYED: Tue Oct 11 20:35:43 2022
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

rancher@rms1:~> kubectl -n cert-manager get po
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-646c67487-p9w77               1/1     Running   0          72s
cert-manager-cainjector-7cb8669d6b-cwghz   1/1     Running   0          72s
cert-manager-webhook-696c5db7ff-sjbw4      1/1     Running   0          72s
```


## 9. 啟用Rancher 2.6.8

```shell=
rancher@rms1:~> helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
"rancher-stable" has been added to your repositories

rancher@rms1:~> kubectl create namespace cattle-system
namespace/cattle-system created

rancher@rms1:~> helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "rancher-stable" chart repository
Update Complete. ⎈Happy Helming!⎈

rancher@rms1:~> helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.example.com
NAME: rancher
LAST DEPLOYED: Tue Oct 11 20:37:49 2022
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


rancher@rms1:~> kubectl -n cattle-system get po
NAME                       READY   STATUS              RESTARTS   AGE
rancher-69595dc9c4-mrhwt   0/1     ContainerCreating   0          2m17s
rancher-69595dc9c4-nhl5m   0/1     ContainerCreating   0          2m17s
rancher-69595dc9c4-tswd8   0/1     ContainerCreating   0          2m17s
rancher@rms1:~> kubectl -n cattle-system get po
NAME                       READY   STATUS              RESTARTS   AGE
rancher-69595dc9c4-mrhwt   0/1     Running             0          2m27s
rancher-69595dc9c4-nhl5m   0/1     Running             0          2m27s
rancher-69595dc9c4-tswd8   0/1     ContainerCreating   0          2m27s
rancher@rms1:~> kubectl -n cattle-system get po
NAME                       READY   STATUS    RESTARTS        AGE
rancher-69595dc9c4-mrhwt   1/1     Running   2 (77s ago)     7m12s
rancher-69595dc9c4-nhl5m   1/1     Running   1 (2m39s ago)   7m12s
rancher-69595dc9c4-tswd8   1/1     Running   2 (77s ago)     7m12s
```