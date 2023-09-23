---
title: Guide_NeuVector
description: Guide_NeuVector
slug: Guide_NeuVector
date: 2023-09-23T06:33:10+08:00
categories:
    - Knowledge Base Category
tags:
    - NeuVector
    - SUSE
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# NeuVector

## 0. install RKE2

### 0.1. 下載安裝腳本
```shell=
rancher@rms1:~> curl -sfL https://get.rke2.io --output install.sh
rancher@rms1:~> chmod +x install.sh
```

### 0.2. 設定RKE2基本餐數
```shell=
rancher@rms1:~> sudo mkdir -p /etc/rancher/rke2/
[sudo] root 的密碼：
rancher@rms1:~> sudo vim /etc/rancher/rke2/config.yaml
rancher@rms1:~> cat /etc/rancher/rke2/config.yaml
node-name:
  - "rms1"
token: my-shared-secret
```

### 0.3. 安裝rke2
```shell=
rancher@rms1:~> sudo ./install.sh
[WARN]  /usr/local is read-only or a mount point; installing to /opt/rke2
[INFO]  finding release for channel v1.24.6+rke2r1
[INFO]  using v1.24.6+rke2r1 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.24.6+rke2r1/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.24.6+rke2r1/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /opt/rke2
[INFO]  updating tarball contents to reflect install path
[INFO]  moving systemd units to /etc/systemd/system
[INFO]  install complete; you may want to run:  export PATH=$PATH:/opt/rke2/bin
rancher@rms1:~> export PATH=$PATH:/opt/rke2/bin
```

### 0.4. enable rke2 and setup kubeconfig
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

### 0.5. check pod status
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

## 1. install helm3

### 1.1. check kubernetes version

```shell=
rancher@rms1:~> kubectl get no 
NAME   STATUS   ROLES                       AGE   VERSION
rms1   Ready    control-plane,etcd,master   37h   v1.24.6+rke2r1
```

### 1.2. 下載與安裝helm
```shell=
rancher@rms1:~> wget https://get.helm.sh/helm-v3.9.4-linux-amd64.tar.gz
--2022-10-16 12:34:56--  https://get.helm.sh/helm-v3.9.4-linux-amd64.tar.gz
Resolving get.helm.sh (get.helm.sh)... 152.199.39.108, 2606:2800:247:1cb7:261b:1f9c:2074:3c
Connecting to get.helm.sh (get.helm.sh)|152.199.39.108|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14026634 (13M) [application/x-tar]
Saving to: ‘helm-v3.9.4-linux-amd64.tar.gz’

helm-v3.9.4-linux-amd64.tar 100%[========================================>]  13.38M  7.86MB/s    in 1.7s    

2022-10-16 12:34:59 (7.86 MB/s) - ‘helm-v3.9.4-linux-amd64.tar.gz’ saved [14026634/14026634]

rancher@rms1:~> tar zxvf helm-v3.9.4-linux-amd64.tar.gz 
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
rancher@rms1:~> sudo cp linux-amd64/helm /usr/local/bin/
[sudo] password for root: 

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

## 2. install NeuVector

### 2.1. 加入neuvector helm chart
```shell=
rancher@rms1:~> helm repo add neuvector https://neuvector.github.io/neuvector-helm/
"neuvector" has been added to your repositories

rancher@rms1:~> helm search repo neuvector/core
NAME          	CHART VERSION	APP VERSION	DESCRIPTION                             
neuvector/core	2.2.3        	5.0.3      	Helm chart for NeuVector's core services
rancher@rms1:~> helm search repo neuvector/core -l
NAME          	CHART VERSION	APP VERSION	DESCRIPTION                                       
neuvector/core	2.2.3        	5.0.3      	Helm chart for NeuVector's core services          
neuvector/core	2.2.2        	5.0.2      	Helm chart for NeuVector's core services          
neuvector/core	2.2.1        	5.0.1      	Helm chart for NeuVector's core services          
neuvector/core	2.2.0        	5.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.9.2        	4.4.4-s2   	Helm chart for NeuVector's core services          
neuvector/core	1.9.1        	4.4.4      	Helm chart for NeuVector's core services          
neuvector/core	1.9.0        	4.4.4      	Helm chart for NeuVector's core services          
neuvector/core	1.8.9        	4.4.3      	Helm chart for NeuVector's core services          
neuvector/core	1.8.8        	4.4.2      	Helm chart for NeuVector's core services          
neuvector/core	1.8.7        	4.4.1      	Helm chart for NeuVector's core services          
neuvector/core	1.8.6        	4.4.0      	Helm chart for NeuVector's core services          
neuvector/core	1.8.5        	4.3.2      	Helm chart for NeuVector's core services          
neuvector/core	1.8.4        	4.3.2      	Helm chart for NeuVector's core services          
neuvector/core	1.8.3        	4.3.2      	Helm chart for NeuVector's core services          
neuvector/core	1.8.2        	4.3.1      	Helm chart for NeuVector's core services          
neuvector/core	1.8.0        	4.3.0      	Helm chart for NeuVector's core services          
neuvector/core	1.7.7        	4.2.2      	Helm chart for NeuVector's core services          
neuvector/core	1.7.6        	4.2.2      	Helm chart for NeuVector's core services          
neuvector/core	1.7.5        	4.2.0      	Helm chart for NeuVector's core services          
neuvector/core	1.7.2        	4.2.0      	Helm chart for NeuVector's core services          
neuvector/core	1.7.1        	4.2.0      	Helm chart for NeuVector's core services          
neuvector/core	1.7.0        	4.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.6.9        	4.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.6.8        	4.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.6.7        	4.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.6.6        	4.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.6.5        	4.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.6.4        	4.0.0      	Helm chart for NeuVector's core services          
neuvector/core	1.6.1        	4.0.0      	NeuVector Full Lifecycle Container Security Pla...

rancher@rms1:~> helm show values neuvector/core --version 2.2.3 > neuvector-values.yaml

```

### 2.2. 調整yaml - part-1(replicas for test)
```yaml=
controller:
...
  replicas: 1
...
...
...
  scanner:
    enabled: true
    replicas: 1
...
...
```

### 2.3. 調整yaml - part-2(CRI for k3s)
```yaml=
k3s:
  enabled: true
```

### 2.4. 建立NeuVector NameSpace and create neuvector
```shell=
rancher@rms1:~> kubectl create ns neuvector
namespace/neuvector created

rancher@rms1:~> helm install neuvector neuvector/core --version 2.2.3 --namespace neuvector --values neuvector-values.yaml
NAME: neuvector
LAST DEPLOYED: Sun Oct 16 13:10:22 2022
NAMESPACE: neuvector
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the NeuVector URL by running these commands:
  NODE_PORT=$(kubectl get --namespace neuvector -o jsonpath="{.spec.ports[0].nodePort}" services neuvector-service-webui)
  NODE_IP=$(kubectl get nodes --namespace neuvector -o jsonpath="{.items[0].status.addresses[0].address}")
  echo https://$NODE_IP:$NODE_PORT

rancher@rms1:~> kubectl -n neuvector get po 
NAME                                        READY   STATUS    RESTARTS   AGE
neuvector-controller-pod-6cf58c7894-hvvlw   1/1     Running   0          42s
neuvector-enforcer-pod-rpzqg                1/1     Running   0          42s
neuvector-manager-pod-8488cb586c-87q9n      1/1     Running   0          42s
neuvector-scanner-pod-7d6bc8f775-pfwkk      1/1     Running   0          42s

rancher@rms1:~> kubectl -n neuvector get svc
NAME                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
neuvector-service-webui           NodePort    10.43.189.243   <none>        8443:30805/TCP                  66s
neuvector-svc-admission-webhook   ClusterIP   10.43.103.17    <none>        443/TCP                         67s
neuvector-svc-controller          ClusterIP   None            <none>        18300/TCP,18301/TCP,18301/UDP   67s
neuvector-svc-crd-webhook         ClusterIP   10.43.185.195   <none>        443/TCP                         66s

```

## 3. 其他
### 3.1. image registry
1. Name: Nginx
2. Registry: https://registry.hub.docker.com/
3. Filter: nginx:stable

### 3.2. Response Rule
1. Category: `Security Event`
2. Group: `nv.default`
3. Criteria: `name:Process.Profile.Violation`
4. Action: `Quarantine`
5. Status: `Enabled`

### 3.3. CLI reference

#### 3.3.1. nginx install recon-ng
```shell=
rancher@rms1:~> kubectl create deploy web --image=nginx --port=80 --replicas=1
deployment.apps/web created

rancher@rms1:~> kubectl get po 
NAME                  READY   STATUS    RESTARTS   AGE
web-cff6559d7-jddwt   1/1     Running   0          8s

rancher@rms1:~> kubectl exec -it web-cff6559d7-jddwt -- /bin/bash
root@web-cff6559d7-jddwt:/# apt update
Get:1 http://deb.debian.org/debian bullseye InRelease [116 kB]
Get:2 http://deb.debian.org/debian-security bullseye-security InRelease [48.4 kB]
Get:3 http://deb.debian.org/debian bullseye-updates InRelease [44.1 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 Packages [8184 kB]
Get:5 http://deb.debian.org/debian-security bullseye-security/main amd64 Packages [189 kB]                  
Get:6 http://deb.debian.org/debian bullseye-updates/main amd64 Packages [6340 B]                            
Fetched 8588 kB in 8s (1050 kB/s)                                                                           
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
1 package can be upgraded. Run 'apt list --upgradable' to see it.

root@web-cff6559d7-jddwt:/# apt install recon-ng
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  javascript-common libgpm2 libjs-jquery libjs-skeleton libjs-sphinxdoc libjs-underscore libmpdec3
...
...
...
0 upgraded, 69 newly installed, 0 to remove and 1 not upgraded.
Need to get 13.0 MB of archives.
After this operation, 55.7 MB of additional disk space will be used.
Do you want to continue? [Y/n] y

...
...
...
Setting up recon-ng (5.1.1-3) ...
Processing triggers for libc-bin (2.31-13+deb11u4) ...
root@web-cff6559d7-jddwt:/# recon-ng
[*] Version check disabled.

    _/_/_/    _/_/_/_/    _/_/_/    _/_/_/    _/      _/            _/      _/    _/_/_/
   _/    _/  _/        _/        _/      _/  _/_/    _/            _/_/    _/  _/       
  _/_/_/    _/_/_/    _/        _/      _/  _/  _/  _/  _/_/_/_/  _/  _/  _/  _/  _/_/_/
 _/    _/  _/        _/        _/      _/  _/    _/_/            _/    _/_/  _/      _/ 
_/    _/  _/_/_/_/    _/_/_/    _/_/_/    _/      _/            _/      _/    _/_/_/    


                                          /\
                                         / \\ /\
    Sponsored by...               /\  /\/  \\V  \/\
                                 / \\/ // \\\\\ \\ \/\
                                // // BLACK HILLS \/ \\
                               www.blackhillsinfosec.com

                  ____   ____   ____   ____ _____ _  ____   ____  ____
                 |____] | ___/ |____| |       |   | |____  |____ |
                 |      |   \_ |    | |____   |   |  ____| |____ |____
                                   www.practisec.com

                      [recon-ng v5.1.1, Tim Tomes (@lanmaster53)]                       

[*] No modules enabled/installed.

[recon-ng][default] > 
```

#### 3.3.2. nginx bash
```shell=
rancher@rms1:~> kubectl get po 
NAME                  READY   STATUS    RESTARTS   AGE
web-cff6559d7-jddwt   1/1     Running   0          5m4s
rancher@rms1:~> kubectl exec -it web-cff6559d7-jddwt -- /bin/bash
```


