---
title: Nutanix_NKP-v2.13_AHV
description: Nutanix_NKP-v2.13_AHV
slug: Nutanix_NKP-v2.13_AHV
date: 2025-03-07T05:45:12+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - NKP
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Nutanix NKP v2.13.1 Install with private registry

NKP 安裝 Lab，安裝在 AHV 環境上 ，透過 cli 方式安裝

會建立 image registry 來放置 NKP 需要的 image，建置完成後會升級到 2.14



## 環境資訊

### 版本

Prism Central: 2024.2.0.3

AOS: 6.5.1.1

NKP: 2.13.1

Kubernetes: 1.30.5

OS Image: ubuntu-22.04

KIB: v2.18.0

### IP 與名稱

Prism Central: https://10.38.14.10:9440

Prism Element (PHX-SPOC014-1): https://10.38.14.7:9440/ 

bastion ip : 10.38.14.24

Storage Container: default

Subnet: primary

kubeVIP: 10.38.14.51

kube Ingress ip Range:  10.38.14.52-10.38.14.54

Subnet Mask: 255.255.255.192

Gateway IP Address: 10.38.14.1



## Bastion

### 軟體下載

nkp-airgapped, nkp cli , kib 從 Portal 上直接用 copy link 方式下載

```
[nkp@nkp-bastion ~]$ curl -o nkp-air-gapped-bundle_v2.13.1_linux_amd64.tar.gz "https://download.nutanix.com/downloads/nkp/v2.13.1/nkp-air-gapped-bundle_v2.13.1_linux_amd64.tar.gz?Expires=1740156924&Key-Pair-xx__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 14.3G  100 14.3G    0     0   317M      0  0:00:46  0:00:46 --:--:--  289M

[nkp@nkp-bastion ~]$ curl -o nkp_v2.13.1_linux_amd64.tar.gz "https://download.nutanix.com/downloads/nkp/v2.13.1/nkp_v2.13.1_linux_amd64.tar.gz?Expires=1740156983&Key-Pair-xx__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 97.5M  100 97.5M    0     0  16.5M      0  0:00:05  0:00:05 --:--:-- 17.5M

[nkp@nkp-bastion ~]$ curl -o konvoy-image-bundle-v2.18.0_linux_amd64.tar.gz "https://download.nutanix.com/downloads/nkp/v2.13.1/konvoy-image-bundle-v2.18.0_linux_amd64.tar.gz?Expires=1740157007&Key-Pair-xxx__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  334M  100  334M    0     0  15.7M      0  0:00:21  0:00:21 --:--:-- 15.9M

[nkp@nkp-bastion ~]$ ll
total 15484132
-rw-r--r--. 1 nkp nkp   350719341 Feb 20 22:57 konvoy-image-bundle-v2.18.0_linux_amd64.tar.gz
-rw-r--r--. 1 nkp nkp 15402765171 Feb 20 22:56 nkp-air-gapped-bundle_v2.13.1_linux_amd64.tar.gz
-rw-r--r--. 1 nkp nkp   102264118 Feb 20 22:56 nkp_v2.13.1_linux_amd64.tar.gz
```

### 軟體安裝

```
[nkp@nkp-bastion ~]$ sudo yum install podman yum-utils bzip2 wget tar -y
[nkp@nkp-bastion ~]$ sudo yum install git-all -y
[nkp@nkp-bastion ~]$ wget https://get.helm.sh/helm-v3.17.1-linux-amd64.tar.gz
[nkp@nkp-bastion ~]$ sudo cp linux-amd64/helm /usr/local/bin

[nkp@nkp-bastion ~]$ mkdir -p nkptools/nkp
[nkp@nkp-bastion ~]$ mkdir -p nkptools/kib
[nkp@nkp-bastion ~]$ mkdir -p nkptools/nkp-airgap
[nkp@nkp-bastion ~]$ tar xvf nkp_v2.13.1_linux_amd64.tar.gz -C nkptools/nkp
[nkp@nkp-bastion ~]$ tar xvf konvoy-image-bundle-v2.18.0_linux_amd64.tar.gz -C nkptools/kib
[nkp@nkp-bastion ~]$ tar xvf nkp-air-gapped-bundle_v2.13.1_linux_amd64.tar.gz -C nkptools/nkp-airgap

[nkp@nkp-bastion ~]$ sudo cp nkptools/nkp/nkp /usr/local/bin/
[nkp@nkp-bastion ~]$ sudo cp nkptools/kib/konvoy-image /usr/local/bin/
[nkp@nkp-bastion ~]$ sudo cp nkptools/nkp-airgap/nkp-v2.13.1/kubectl /usr/local/bin/

[nkp@nkp-bastion ~]$ nkp version
diagnose: v0.10.1
imagebuilder: v0.20.0
kommander: v2.13.1
konvoy: v2.13.1
mindthegap: v1.16.0
nkp: v2.13.1

[nkp@nkp-bastion ~]$ kubectl version
Client Version: v1.30.5
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

### 基本設定

disable firewalld, selinux

```
[nkp@nkp-bastion ~]$ sudo systemctl disable firewalld --now
Removed "/etc/systemd/system/multi-user.target.wants/firewalld.service".
Removed "/etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service".

[nkp@nkp-bastion ~]$ sudo vi /etc/selinux/config
SELINUXTYPE=disabled

reboot

[nkp@nkp-bastion ~]$ getenforce
Disabled

```



### Registry

#### 啟動 Registry

```
"switch to root" (方便使用，也可以用一般使用者，但要另外設定)

[root@nkp-bastion ~] podman login registry-1.docker.io

"Start registry"

[root@nkp-bastion ~] podman run -d -p 5000:5000 --restart always --name nkp-registry registry:latest
[root@nkp-bastion ~] export BOOTSTRAP_HOST_IP=10.38.14.24 >> .bashrc
[root@nkp-bastion ~] echo "export REGISTRY_URL=http://10.38.14.24:5000" >> .bashrc
[root@nkp-bastion ~] source .bashrc

```

#### 推送 NKP Image

```
"Push Image"

[root@nkp-bastion ~] nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/container-images/kommander-image-bundle-v2.13.1.tar --to-registry=$REGISTRY_URL --to-registry-insecure-skip-tls-verify
 ✓ Creating temporary directory
 ✓ Unarchiving image bundle "/home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/container-images/kommander-image-bundle-v2.13.1.tar"
 ✓ Parsing image bundle config
 ✓ Starting temporary Docker registry
 ✓ Pushing bundled images [================================>123/123] (time elapsed 57s) 
 
[root@nkp-bastion ~] nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/container-images/konvoy-image-bundle-v2.13.1.tar --to-registry=$REGISTRY_URL --to-registry-insecure-skip-tls-verify
 ✓ Creating temporary directory
 ✓ Unarchiving image bundle "/home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/container-images/konvoy-image-bundle-v2.13.1.tar"
 ✓ Parsing image bundle config
 ✓ Starting temporary Docker registry
 ✓ Pushing bundled images [================================>115/115] (time elapsed 35s) 
 
[root@nkp-bastion ~] nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/container-images/nkp-catalog-applications-image-bundle-v2.13.1.tar --to-registry=$REGISTRY_URL --to-registry-insecure-skip-tls-verify
 ✓ Creating temporary directory
 ✓ Unarchiving image bundle "/home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/container-images/nkp-catalog-applications-image-bundle-v2.13.1.tar"
 ✓ Parsing image bundle config
 ✓ Starting temporary Docker registry
 ✓ Pushing bundled images [====================================>8/8] (time elapsed 04s) 
```

![image-20250221163239471](https://kenkenny.synology.me:5543/images/2025/02/image-20250221163239471.png)

#### load image

```
podman load -i /home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/konvoy-bootstrap-image-v2.13.1.tar
podman load -i /home/nkp/nkptools/nkp-airgap/nkp-v2.13.1/nkp-image-builder-image-v0.20.0.tar

[root@nkp-bastion ~] podman images
REPOSITORY                              TAG         IMAGE ID      CREATED        SIZE
localhost/mesosphere/konvoy-bootstrap   v2.13.1     e7b2ed9a825c  5 weeks ago    2.13 GB
docker.io/library/registry              latest      26b2eb03618e  16 months ago  26 MB
localhost/mesosphere/nkp-image-builder  v0.20.0     f8ce7f154c3b  45 years ago   779 MB
```



## Create Cluster

上述環境、Bastion 準備完成後，可以開始佈建 NKP Management Cluster



### Bootstrap Cluster

```
[root@nkp-bastion ~] nkp create bootstrap
 ✓ Creating a bootstrap cluster 
 ✓ Initializing new CAPI components 
 ✓ Initializing new CAPI components 
 ✓ Creating ClusterClass resources 
 
 [root@nkp-bastion ~] kubectl get nodes
NAME                                     STATUS   ROLES           AGE     VERSION
konvoy-capi-bootstrapper-control-plane   Ready    control-plane   2m27s   v1.30.5
[root@nkp-bastion ~] kubectl get pods -A
NAMESPACE                           NAME                                                             READY   STATUS    RESTARTS   AGE
caaph-system                        caaph-controller-manager-5c4b55f8b6-n6l5j                        1/1     Running   0          93s
capa-system                         capa-controller-manager-6c9654f67d-4cr6z                         1/1     Running   0          108s
capg-system                         capg-controller-manager-6f85f89dbc-z4qgv                         1/1     Running   0          96s
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-679764b5b7-f5558       1/1     Running   0          111s
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-6db6bbdd85-zbhkg   1/1     Running   0          110s
...
```

### Machine Image (if required)

```
echo "export NUTANIX_USER=admin" >> ~/.bashrc

echo "export NUTANIX_PASSWORD=nx2Tech702!" >> ~/.bashrc

echo "export PE_CLUSTER_NAME=PHX-SPOC014-1" >> ~/.bashrc

echo "export PC_ENDPOINT=10.38.14.10" >> ~/.bashrc

echo "export SUBNET=primary" >> ~/.bashrc

source ~/.bashrc

[root@nkp-bastion ~] env
REGISTRY_URL=http://10.38.14.24:5000
PE_CLUSTER_NAME=PHX-SPOC014-1
NUTANIX_USER=admin
PC_ENDPOINT=10.38.14.10
BOOTSTRAP_HOST_IP=10.38.14.24
...



[root@nkp-bastion ~] nkp create image nutanix ubuntu-22.04 --cluster=$PE_CLUSTER_NAME --endpoint=$PC_ENDPOINT --subnet=$SUBNET --insecure

==> Wait completed after 6 minutes 33 seconds

==> Builds finished. The artifacts of successful builds are:
--> nutanix.kib_image: nkp-ubuntu-22.04-1.30.5-20250221084439
--> nutanix.kib_image: nkp-ubuntu-22.04-1.30.5-20250221084439
--> nutanix.kib_image: nkp-ubuntu-22.04-1.30.5-20250221084439

echo "export VM_IMAGE=nkp-ubuntu-22.04-1.30.5-20250221084439" >> ~/.bashrc
source ~/.bashrc
```



![](https://kenkenny.synology.me:5543/images/2025/02/image-20250221164530515.png)

![image-20250221164753010](https://kenkenny.synology.me:5543/images/2025/02/image-20250221164753010.png)

![image-20250221164808447](https://kenkenny.synology.me:5543/images/2025/02/image-20250221164808447.png)

![image-20250221164837208](https://kenkenny.synology.me:5543/images/2025/02/image-20250221164837208.png)

![image-20250221164908398](https://kenkenny.synology.me:5543/images/2025/02/image-20250221164908398.png)

![image-20250221165050977](https://kenkenny.synology.me:5543/images/2025/02/image-20250221165050977.png)

![image-20250221164925341](https://kenkenny.synology.me:5543/images/2025/02/image-20250221164925341.png)

![image-20250221165127407](https://kenkenny.synology.me:5543/images/2025/02/image-20250221165127407.png)

![image-20250221164704789](https://kenkenny.synology.me:5543/images/2025/02/image-20250221164704789.png)

![image-20250221165200360](https://kenkenny.synology.me:5543/images/2025/02/image-20250221165200360.png)

### Management Cluster

#### Env 設定環境變數

```
env.sh
export MGMT_CLUSTER_NAME=nkp-poc
export PE_CLUSTER_NAME=PHX-SPOC014-1
export SUBNET=primary
export MGMT_CP_ENDPOINT=10.38.14.51
export VM_IMAGE=nkp-ubuntu-22.04-1.30.5-20250221084439
export STORAGE_CONTAINER=default
export MGMT_LB_IP_RANGE=10.38.14.52-10.38.14.54
export NUTANIX_USER=admin
export NUTANIX_PASSWORD=nx2Tech702!
export PC_ENDPOINT=10.38.14.10
export CSI_HYPERVISOR_ATTACHED="true"
export CSI_FILESYSTEM="ext4"
export CONTROL_PLANE_IP="10.38.14.51"
export SSH_PUBLIC_KEY="/home/nkp/.ssh/id_rsa.pub"
export REGISTRY_URL="http://10.38.14.24:5000"

source env.sh
```

#### Create  Cluster

```
nkp create cluster nutanix \
--cluster-name=$MGMT_CLUSTER_NAME \
--control-plane-prism-element-cluster=$PE_CLUSTER_NAME \
--control-plane-subnets=$SUBNET \
--control-plane-endpoint-ip=$MGMT_CP_ENDPOINT \
--control-plane-vm-image=$VM_IMAGE \
--worker-prism-element-cluster=$PE_CLUSTER_NAME \
--worker-subnets=$SUBNET \
--worker-vm-image=$VM_IMAGE \
--worker-memory=16 \
--csi-storage-container=$STORAGE_CONTAINER \
--kubernetes-service-load-balancer-ip-range=$MGMT_LB_IP_RANGE \
--endpoint="https://$PC_ENDPOINT:9440" \
--insecure=true \
--registry-mirror-url=$REGISTRY_URL \
--ssh-public-key-file=$SSH_PUBLIC_KEY \
--dry-run \
--output=yaml > deploy-nkp-$MGMT_CLUSTER_NAME.yaml

[root@nkp-bastion ~] ll
total 16
-rw-------. 1 root root  997 Feb 20 22:43 anaconda-ks.cfg
-rw-r--r--. 1 root root 5913 Feb 21 01:20 deploy-nkp-nkp-poc.yaml
-rwxr-xr-x. 1 root root  586 Feb 21 01:17 env.sh

[root@nkp-bastion ~] kubectl create -f deploy-nkp-nkp-poc.yaml
cluster.cluster.x-k8s.io/nkp-poc created
secret/nkp-poc-pc-credentials created
secret/nkp-poc-pc-credentials-for-csi created
configmap/kommander-bootstrap-configuration created
secret/prism-central-metadata created
secret/global-nutanix-credentials created

watch -n 5 nkp describe cluster --cluster-name $MGMT_CLUSTER_NAME
```

![image-20250221172657309](https://kenkenny.synology.me:5543/images/2025/02/image-20250221172657309.png)

```
[root@nkp-bastion ~] nkp get kubeconfig --cluster-name nkp-poc > nkp-poc.conf
[root@nkp-bastion ~] chmod 600 nkp-poc.conf
[root@nkp-bastion ~] ll
total 32
-rw-------. 1 root root  997 Feb 20 22:43 anaconda-ks.cfg
-rw-r--r--. 1 root root 5913 Feb 21 01:20 deploy-nkp-nkp-poc.yaml
-rw-r--r--. 1 root root 5913 Feb 21 01:21 deploy-nkp-nkp-poc.yaml_bk
-rwxr-xr-x. 1 root root  586 Feb 21 01:17 env.sh
-rw-------. 1 root root 5543 Feb 21 01:28 nkp-poc.conf

echo "export KUBECONFIG=~/nkp-poc.conf" >> ~/.bashrc
source .bashrc 

[root@nkp-bastion ~] kubectl get nodes
NAME                             STATUS   ROLES           AGE     VERSION
nkp-poc-d78fs-dkfnd              Ready    control-plane   5m43s   v1.30.5
nkp-poc-d78fs-gmllf              Ready    control-plane   8m8s    v1.30.5
nkp-poc-d78fs-x989r              Ready    control-plane   4m25s   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-4jbfm   Ready    <none>          6m17s   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-cxlx7   Ready    <none>          6m40s   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-kwvlj   Ready    <none>          6m15s   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-m9dkw   Ready    <none>          6m18s   v1.30.5


"建立 capi-components"
[root@nkp-bastion ~] nkp create capi-components
 ✓ Initializing new CAPI components 
 ✓ Initializing new CAPI components 
 ✓ Creating ClusterClass resources 
```

#### Delete bootstrap

```
"從 bootstratp 移轉 capi-resources"

[root@nkp-bastion ~] unset KUBECONFIG
[root@nkp-bastion ~] kubectl get nodes
NAME                                     STATUS   ROLES           AGE   VERSION
konvoy-capi-bootstrapper-control-plane   Ready    control-plane   60m   v1.30.5

[root@nkp-bastion ~] nkp move capi-resources --to-kubeconfig nkp-poc.conf
 ✓ Moving cluster resources 

You can now view resources in the moved cluster by using the --kubeconfig flag with kubectl.
For example: kubectl --kubeconfig="nkp-poc.conf" get nodes

[root@nkp-bastion ~] nkp delete bootstrap
 ✓ Deleting bootstrap cluster 
 
[root@nkp-bastion ~] source ~/.bashrc
[root@nkp-bastion ~] kubectl get nodes
NAME                             STATUS   ROLES           AGE   VERSION
nkp-poc-d78fs-dkfnd              Ready    control-plane   12m   v1.30.5
nkp-poc-d78fs-gmllf              Ready    control-plane   14m   v1.30.5
nkp-poc-d78fs-x989r              Ready    control-plane   10m   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-4jbfm   Ready    <none>          12m   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-cxlx7   Ready    <none>          13m   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-kwvlj   Ready    <none>          12m   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-m9dkw   Ready    <none>          12m   v1.30.5
[root@nkp-bastion ~] kubectl get ns
NAME                                STATUS   AGE
caaph-system                        Active   4m56s
capa-system                         Active   5m9s
capg-system                         Active   4m59s
capi-kubeadm-bootstrap-system       Active   5m10s
capi-kubeadm-control-plane-system   Active   5m9s
capi-system                         Active   5m11s
cappp-system                        Active   5m1s
capv-system                         Active   5m
capvcd-system                       Active   4m58s
capx-system                         Active   4m57s
capz-system                         Active   5m7s
caren-system                        Active   3m50s
cert-manager                        Active   5m23s
default                             Active   14m
git-operator-system                 Active   88s
kommander                           Active   13m
kommander-flux                      Active   109s
kube-node-lease                     Active   14m
kube-public                         Active   14m
kube-system                         Active   14m
metallb-system                      Active   14m
node-feature-discovery              Active   13m
ntnx-system                         Active   14m

```

#### 取得 Dashboard

```

watch kubectl get pods -n kommander

[root@nkp-bastion ~] nkp get dashboard 
Username: clever_euclid
Password: 8JW63deYbPpIIrCtTPBXJihIF8xsWxw0YTCSVvcAa1SupyZ14GqEEVFGXzAb7ZB7
URL: https://10.38.14.52/dkp/kommander/dashboard
```

串接LDAP、License匯入完成後登入畫面

![image-20250225151046730](https://kenkenny.synology.me:5543/images/2025/02/image-20250225151046730.png)

#### 其他叢集資訊

```
"叢集資訊"

[root@nkp-bastion ~] kubectl get namespace kube-system --output jsonpath={.metadata.uid}
7e1c859a-0d30-46a3-83d5-21398e4d7cc4

[root@nkp-bastion ~] kubectl get cluster
NAME      CLUSTERCLASS   PHASE         AGE     VERSION
nkp-poc   nkp-nutanix    Provisioned   2d20h   v1.30.5

"POC License Key"
# AEAAN-AAAYY-T9A7V-5UL6S-49UQ2-H89DG-EP2AR

[root@nkp-bastion ~] kubectl describe cluster nkp-poc
Name:         nkp-poc
Namespace:    default
Labels:       cluster.x-k8s.io/cluster-name=nkp-poc
              cluster.x-k8s.io/provider=nutanix
              konvoy.d2iq.io/cluster-name=nkp-poc
              konvoy.d2iq.io/provider=nutanix
              topology.cluster.x-k8s.io/owned=
Annotations:  caren.nutanix.com/cluster-uuid: 019527cf-0ec5-7b3b-b4aa-3f50c7dc3b4f
API Version:  cluster.x-k8s.io/v1beta1
Kind:         Cluster
Metadata:
  Creation Timestamp:  2025-02-21T09:36:01Z
  Finalizers:
    cluster.cluster.x-k8s.io
  Generation:        2
  Resource Version:  10471
  UID:               4236d3d9-3d77-4c0b-8494-e67dfe774f92
Spec:
  Cluster Network:
    Pods:
      Cidr Blocks:
        192.168.0.0/16
    Services:
      Cidr Blocks:
        10.96.0.0/12
  Control Plane Endpoint:
    Host:  10.38.14.51
    Port:  6443
  Control Plane Ref:
    API Version:  controlplane.cluster.x-k8s.io/v1beta1
    Kind:         KubeadmControlPlane
    Name:         nkp-poc-d78fs
    Namespace:    default
  Infrastructure Ref:
    API Version:  infrastructure.cluster.x-k8s.io/v1beta1
    Kind:         NutanixCluster
    Name:         nkp-poc-5cvwf
    Namespace:    default
  Topology:
    Class:  nkp-nutanix
    Control Plane:
      Metadata:
      Replicas:  3
    Variables:
      Name:  clusterConfig
      Value:
        Addons:
          Ccm:
            Credentials:
              Secret Ref:
                Name:  nkp-poc-pc-credentials
            Strategy:  HelmAddon
          Cluster Autoscaler:
            Strategy:  HelmAddon
          Cni:
            Provider:  Cilium
            Strategy:  HelmAddon
          Csi:
            Default Storage:
              Provider:              nutanix
              Storage Class Config:  volume
            Providers:
              Nutanix:
                Credentials:
                  Secret Ref:
                    Name:  nkp-poc-pc-credentials-for-csi
                Storage Class Configs:
                  Volume:
                    Allow Expansion:  true
                    Parameters:
                      csi.storage.k8s.io/fstype:  ext4
                      Description:                CSI StorageClass nutanix-volume for nkp-poc
                      Flash Mode:                 DISABLED
                      Hypervisor Attached:        ENABLED
                      Storage Container:          default
                      Storage Type:               NutanixVolumes
                    Reclaim Policy:               Delete
                    Volume Binding Mode:          WaitForFirstConsumer
                Strategy:                         HelmAddon
            Snapshot Controller:
              Strategy:  HelmAddon
          Nfd:
            Strategy:  HelmAddon
          Service Load Balancer:
            Configuration:
              Address Ranges:
                End:    10.38.14.54
                Start:  10.38.14.52
            Provider:   MetalLB
        Control Plane:
          Nutanix:
            Machine Details:
              Boot Type:  uefi
              Cluster:
                Name:  PHX-SPOC014-1
                Type:  name
              Image:
                Name:       nkp-ubuntu-22.04-1.30.5-20250221084439
                Type:       name
              Memory Size:  16Gi
              Subnets:
                Name:            primary
                Type:            name
              System Disk Size:  80Gi
              Vcpu Sockets:      4
              Vcpus Per Socket:  1
        Dns:
          Core DNS:
        Encryption At Rest:
          Providers:
            Secretbox:
        Global Image Registry Mirror:
          URL:  http://10.38.14.24:5000
        Nutanix:
          Control Plane Endpoint:
            Host:  10.38.14.51
            Port:  6443
            Virtual IP:
              Provider:  KubeVIP
          Prism Central Endpoint:
            Credentials:
              Secret Ref:
                Name:  nkp-poc-pc-credentials
            Insecure:  true
            URL:       https://10.38.14.10:9440
    Version:           v1.30.5
    Workers:
      Machine Deployments:
        Class:  default-worker
        Metadata:
          Annotations:
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size:  4
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size:  4
        Name:                                                             md-0
        Variables:
          Overrides:
            Name:  workerConfig
            Value:
              Nutanix:
                Machine Details:
                  Boot Type:  uefi
                  Cluster:
                    Name:  PHX-SPOC014-1
                    Type:  name
                  Image:
                    Name:       nkp-ubuntu-22.04-1.30.5-20250221084439
                    Type:       name
                  Memory Size:  16Gi
                  Subnets:
                    Name:            primary
                    Type:            name
                  System Disk Size:  80Gi
                  Vcpu Sockets:      8
                  Vcpus Per Socket:  1
Status:
  Conditions:
    Last Transition Time:  2025-02-21T09:36:14Z
    Status:                True
    Type:                  Ready
    Last Transition Time:  2025-02-21T09:36:10Z
    Status:                True
    Type:                  ControlPlaneInitialized
    Last Transition Time:  2025-02-21T09:36:14Z
    Status:                True
    Type:                  ControlPlaneReady
    Last Transition Time:  2025-02-21T09:36:11Z
    Status:                True
    Type:                  InfrastructureReady
    Last Transition Time:  2025-02-21T09:37:32Z
    Status:                True
    Type:                  KommanderInitialized
    Last Transition Time:  2025-02-21T09:36:13Z
    Status:                True
    Type:                  TopologyReconciled
  Control Plane Ready:     true
  Infrastructure Ready:    true
  Observed Generation:     2
  Phase:                   Provisioned
Events:                    <none>
```



## Upgrade NKP

### 目前版本

NKP : v2.13.1 預計升級到 v2.14

Kubernetes: v1.30.5 預計升級到 v1.31.4

OS image : Ubuntu 22.04 , Kernel 5.15.0-119-generic

KiB : 2.18 預計升級到 2.22.2

![image-20250306140245123](https://kenkenny.synology.me:5543/images/2025/03/image-20250306140245123.png)

![image-20250306140338580](https://kenkenny.synology.me:5543/images/2025/03/image-20250306140338580.png)

### 升級前先確認

1. NKP Release Notes
2. 更新的項目及版本是否會影響現有系統及應用程式
3. 用 nkp version 指令確認 NKP 版本
4. 確認版本相容性  [Upgrade Compatibility Tables](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_14:top-ug-nkp-compat-table-c.html)
5. 重要系統套用 Deploy Pod Disruption Budget (PDB) https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
6. 使用 Velero 備份現有設定



### Upgrade Path

![image-20250306133207749](https://kenkenny.synology.me:5543/images/2025/03/image-20250306133207749.png)

### Upgrade Order (Nutanix)

**先升級 Management Cluster 再升級 Workload Cluster**

1. Bastion 升級 nkp, kib  指令版本
2. 下載 nkp-airgapped bundle 包並升級 kubectl
3. 上傳 Kommander, Konvoy, Catalog Application (Ultimate) 等 image 到 private registry
4. 製作或下載新版本的 OS image ( 請查看版本相容性確認是否需要更新 )
5. 升級 Kommander
6. 升級 Cluster 並指定 OS Image
   在 Nutanix 環境此步驟會直接升級 Kubernetes 版本

### Upgrade NKP PRO

#### 1.  Bastion 指令升級及bundle

 檔案從 Support Portal  copy link下載 

```
[nkp@nkp-bastion nkp-2.14]$ pwd
/home/nkp/nkp-2.14

[nkp@nkp-bastion nkp-2.14]$ curl -o nkp_v2.14.0_linux_amd64.tar.gz "https://download.nutanix.com/downloads/nkp/v2.14.0/nkp_v2.14.0_linux_amd64.tar.gz?Expires=17xxxxxQ__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 90.5M  100 90.5M    0     0   319M      0 --:--:-- --:--:-- --:--:--  319M

[nkp@nkp-bastion nkp-2.14]$ curl -o konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz "https://download.nutanix.com/downloads/nkp/v2.14.0/konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz?Expires=174127xxxxxx__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  330M  100  330M    0     0  16.1M      0  0:00:20  0:00:20 --:--:-- 15.8M

[nkp@nkp-bastion nkp-2.14]$ ls -l
total 16674272
-rw-r--r--. 1 nkp nkp   346426397 Mar  5 22:47 konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz
-rw-r--r--. 1 nkp nkp    94896342 Mar  5 22:45 nkp_v2.14.0_linux_amd64.tar.gz
```

解壓縮

```
[nkp@nkp-bastion nkp-2.14]$ tar xvf konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz -C ../nkptools/kib

[nkp@nkp-bastion nkp-2.14]$ tar xvf nkp_v2.14.0_linux_amd64.tar.gz -C ../nkptools/nkp

```

將指令複製到 /usr/local/bin

```
[nkp@nkp-bastion ~]$ sudo cp nkptools/nkp/nkp /usr/local/bin/

[nkp@nkp-bastion ~]$ sudo cp nkptools/kib/konvoy-image /usr/local/bin/

[nkp@nkp-bastion ~]$ nkp version
diagnose: v0.10.1
imagebuilder: v0.22.3
kommander: v2.14.0
konvoy: v2.14.0
mindthegap: v1.16.0
nkp: v2.14.0

[nkp@nkp-bastion ~]$ konvoy-image version
Writing manifest to image destination
Loaded image: localhost/mesosphere/konvoy-image-builder:v2.22.2
konvoy-image, version v2.22.2 (branch: , revision: )
		build date:       
		go version:       go1.22.12
		platform:         linux/amd64
```



#### 2. 下載 airgapped 包

```
[nkp@nkp-bastion nkp-2.14]$ curl -o nkp-air-gapped-bundle_v2.14.0_linux_amd64.tar.gz "https://download.nutanix.com/downloads/nkp/v2.14.0/nkp-air-gapped-bundle_v2.14.0_linux_amd64.tar.gz?Expires=174127xxxxx"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 15.4G  100 15.4G    0     0  17.3M      0  0:15:15  0:15:15 --:--:-- 15.1M

[nkp@nkp-bastion nkp-2.14]$ tar xvf nkp-air-gapped-bundle_v2.14.0_linux_amd64.tar.gz -C ../nkptools/nkp-airgap

[nkp@nkp-bastion ~]$ sudo cp nkptools/nkp-airgap/nkp-v2.14.0/kubectl /usr/local/bin/
[nkp@nkp-bastion ~]$ kubectl version
Client Version: v1.31.4
Kustomize Version: v5.4.2
Server Version: v1.30.5
```

#### 3. 上傳 Image

"Switch to root" 環境變數確認

```
[root@nkp-bastion ~]# echo "export REGISTRY_URL=http://10.38.14.24:5000" >> .bashrc
[root@nkp-bastion ~]# source .bashrc
[root@nkp-bastion ~]# echo $REGISTRY_URL
http://10.38.14.24:5000

[root@nkp-bastion ~]# echo "NUTANIX_USERNAME=admin" >> .bashrc
[root@nkp-bastion ~]# echo "NUTANIX_PASSWORD=nx2Tech702!" >> .bashrc
[root@nkp-bastion ~]# echo "export PE_CLUSTER_NAME=PHX-SPOC014-1" >> ~/.bashrc
[root@nkp-bastion ~]# echo "export PC_ENDPOINT=10.38.14.10" >> ~/.bashrc
[root@nkp-bastion ~]# echo "export SUBNET=primary" >> ~/.bashrc
[root@nkp-bastion ~]# source .bashrc

[root@nkp-bastion ~]# ll /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/
total 12847076
-rw-r--r--. 1 nkp nkp         81 Mar  4 01:08 NOTICES.txt
-rw-r--r--. 1 nkp nkp 8202238464 Mar  4 01:08 kommander-image-bundle-v2.14.0.tar
-rw-r--r--. 1 nkp nkp 4069966848 Mar  4 02:46 konvoy-image-bundle-v2.14.0.tar
-rw-r--r--. 1 nkp nkp  883191296 Mar  4 00:46 nkp-catalog-applications-image-bundle-v2.14.0.tar
```



```
"Push Image"

# Kommander
[root@nkp-bastion ~] nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/kommander-image-bundle-v2.14.0.tar --to-registry=$REGISTRY_URL --to-registry-insecure-skip-tls-verify

 ✓ Creating temporary directory
 ✓ Unarchiving image bundle "/home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/kommander-image-bundle-v2.14.0.tar" 
 ✓ Parsing image bundle config
 ✓ Starting temporary Docker registry
 ✓ Pushing bundled images [================================>137/137] (time elapsed 1m02s) 

 
# Konvoy
[root@nkp-bastion ~] nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/konvoy-image-bundle-v2.14.0.tar --to-registry=$REGISTRY_URL --to-registry-insecure-skip-tls-verify

 ✓ Creating temporary directory
 ✓ Unarchiving image bundle "/home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/konvoy-image-bundle-v2.14.0.tar" 
 ✓ Parsing image bundle config
 ✓ Starting temporary Docker registry
 ✓ Pushing bundled images [================================>119/119] (time elapsed 28s) 
[root@nkp-bastion ~]# 

 
# Catalog 
[root@nkp-bastion ~] nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/nkp-catalog-applications-image-bundle-v2.14.0.tar --to-registry=$REGISTRY_URL --to-registry-insecure-skip-tls-verify

 ✓ Creating temporary directory
 ✓ Unarchiving image bundle "/home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/nkp-catalog-applications-image-bundle-v2.14.0.tar"
 ✓ Parsing image bundle config
 ✓ Starting temporary Docker registry
 ✓ Pushing bundled images [====================================>8/8] (time elapsed 00s) 

```

#### 4. 製作 OS Image

##### For Rocky 

直接在 Prism Central 新增 Support Portal 上的 Image

![image-20250306145124197](https://kenkenny.synology.me:5543/images/2025/03/image-20250306145124197.png)

Add Image

![image-20250306145228491](https://kenkenny.synology.me:5543/images/2025/03/image-20250306145228491.png)

貼上 URL 並點選 Add URL 

![image-20250306145309174](https://kenkenny.synology.me:5543/images/2025/03/image-20250306145309174.png)

點選 save

![image-20250306145400309](https://kenkenny.synology.me:5543/images/2025/03/image-20250306145400309.png)

##### For Others (ubuntu)

```
[root@nkp-bastion ~]# nkp create image nutanix ubuntu-22.04 --cluster=$PE_CLUSTER_NAME --endpoint=$PC_ENDPOINT --subnet=$SUBNET --insecure

Provisioning and configuring image
Manifest files extracted to /root/.nkp-image-builder-3070850318
Trying to pull docker.io/mesosphere/nkp-image-builder:v0.22.3...
Getting image source signatures
Copying blob sha256:99febc243941c0928306275946d862c586d2bd46be4fc4c4e318938a8662d85f
Copying blob sha256:c926b61bad3b94ae7351bafd0c184c159ebf0643b085f7ef1d47ecdc7316833c
Copying blob sha256:89fb4fbbd97a1c94aa08f217f0cb914c612179746c6b3706103a99317e65bcda
Copying blob sha256:fdc5ed007f9c02f44943008862f7c6b62ad238018c6bf0412ab688445e42bbd2
Copying blob sha256:e9db0040a66483f9870e0f445877fa1352a405a970814dea2aff29598198ff6d
Copying blob sha256:acb23a6002388c658931180fb73706eeee11696de5da1bce33df4a8a206de2a0
Copying blob sha256:1166ced2b6761e3267b16e6b82ca07ee8fd00c6626c3507ccc8034ceaf722494
Copying blob sha256:72df65ad9aeb923b7c2cbc5d06a5e16cb2fda0041a0dc04416c02223e54caee4
Copying blob sha256:319333dc043f729fe7dca70c6188d77614743bb43e47693f348f5cafb1425cbe
Copying blob sha256:517f3ad3da58ceb19857ea8e995ea92fb76ae37df29d314265f3d0e8da77060f
Copying config sha256:2c008cec7e3e56932b4510d51067ff0b604822190f3589d091ec265f58e4bf01
Writing manifest to image destination
nutanix.kib_image: output will be in this color.

==> nutanix.kib_image: Creating Packer Builder virtual machine...
...
Build 'nutanix.kib_image' finished after 7 minutes 28 seconds.

==> Wait completed after 7 minutes 28 seconds

==> Builds finished. The artifacts of successful builds are:
--> nutanix.kib_image: nkp-ubuntu-22.04-1.31.4-20250306081227
--> nutanix.kib_image: nkp-ubuntu-22.04-1.31.4-20250306081227
--> nutanix.kib_image: nkp-ubuntu-22.04-1.31.4-20250306081227

OS Image:
nkp-ubuntu-22.04-1.31.4-20250306081227
```

![image-20250306163817087](https://kenkenny.synology.me:5543/images/2025/03/image-20250306163817087.png)

#### 5. 升級 Kommander

##### 原版本

![image-20250306164047890](https://kenkenny.synology.me:5543/images/2025/03/image-20250306164047890.png)

![image-20250306164118766](https://kenkenny.synology.me:5543/images/2025/03/image-20250306164118766.png)

```
[root@nkp-bastion ~]# kubectl get helmreleases -A
NAMESPACE                     NAME                          AGE   READY   STATUS
kommander-default-workspace   karma-traefik-certs           12d   True    Helm install succeeded for release kommander-default-workspace/karma-traefik-certs.v1 with chart kommander-cert-federation@0.0.11
kommander-default-workspace   kubecost-traefik-certs        12d   True    Helm install succeeded for release kommander-default-workspace/kubecost-traefik-certs.v1 with chart kommander-cert-federation@0.0.11
kommander-default-workspace   prometheus-traefik-certs      12d   True    Helm install succeeded for release kommander-default-workspace/prometheus-traefik-certs.v1 with chart kommander-cert-federation@0.0.11
kommander                     ai-navigator-app              10d   True    Helm install succeeded for release kommander/ai-navigator-cluster-info-api.v1 with chart ai-navigator-cluster-info-api@0.2.8
kommander                     cluster-observer-2360587938   12d   True    Helm install succeeded for release kommander/cluster-observer-2360587938.v1 with chart cluster-observer@1.4.1
kommander                     dex                           12d   True    Helm upgrade succeeded for release kommander/dex.v2 with chart dex@2.14.0
kommander                     dex-k8s-authenticator         12d   True    Helm upgrade succeeded for release kommander/dex-k8s-authenticator.v4 with chart dex-k8s-authenticator@1.4.1
kommander                     fluent-bit                    10d   True    Helm install succeeded for release kommander/kommander-fluent-bit.v1 with chart fluent-bit@0.47.7
kommander                     gatekeeper                    12d   True    Helm install succeeded for release kommander/kommander-gatekeeper.v1 with chart gatekeeper@3.17.0
kommander                     gatekeeper-proxy-mutations    12d   True    Helm install succeeded for release kommander/gatekeeper-proxy-mutations.v1 with chart gatekeeper-proxy-mutations@v0.0.1
kommander                     grafana-logging               10d   True    Helm install succeeded for release kommander/grafana-logging.v1 with chart grafana@8.5.8
kommander                     grafana-loki                  10d   True    Helm install succeeded for release kommander/grafana-loki.v1 with chart loki-distributed@0.79.4
kommander                     istio                         10d   True    Helm upgrade succeeded for release istio-system/istio.v2 with chart istio@1.23.3
kommander                     jaeger                        10d   True    Helm install succeeded for release istio-system/jaeger.v1 with chart jaeger-operator@2.56.0
kommander                     karma-traefik-certs           12d   True    Helm install succeeded for release kommander/karma-traefik-certs.v1 with chart kommander-cert-federation@0.0.11
kommander                     kiali                         10d   True    Helm install succeeded for release istio-system/kiali.v1 with chart kiali-operator@1.89.7
kommander                     kommander                     12d   True    Helm install succeeded for release kommander/kommander.v1 with chart kommander@v2.13.1
kommander                     kommander-appmanagement       12d   True    Helm install succeeded for release kommander/kommander-appmanagement.v1 with chart kommander-appmanagement@v2.13.1
kommander                     kommander-operator            12d   True    Helm install succeeded for release kommander/kommander-operator.v1 with chart kommander-operator@0.3.2
kommander                     kommander-ui                  12d   True    Helm install succeeded for release kommander/kommander-kommander-ui.v1 with chart kommander-ui@15.23.17
kommander                     kube-oidc-proxy               12d   True    Helm install succeeded for release kommander/kube-oidc-proxy.v1 with chart kube-oidc-proxy@0.3.4
kommander                     kube-prometheus-stack         10d   True    Helm install succeeded for release kommander/kube-prometheus-stack.v1 with chart kube-prometheus-stack@65.5.0
kommander                     kubecost-traefik-certs        12d   True    Helm install succeeded for release kommander/kubecost-traefik-certs.v1 with chart kommander-cert-federation@0.0.11
kommander                     kubefed                       12d   True    Helm install succeeded for release kube-federation-system/kubefed.v1 with chart kubefed@0.10.4
kommander                     kubernetes-dashboard          10d   True    Helm install succeeded for release kommander/kubernetes-dashboard.v1 with chart kubernetes-dashboard@7.10.0
kommander                     kubetunnel                    10d   True    Helm install succeeded for release kommander/kubetunnel.v1 with chart kubetunnel@v0.0.38
kommander                     logging-operator              10d   True    Helm install succeeded for release kommander/logging-operator.v1 with chart logging-operator@4.2.3
kommander                     logging-operator-logging      10d   True    Helm install succeeded for release kommander/logging-operator-logging.v1 with chart logging-operator-logging@4.2.2
kommander                     object-bucket-claims          10d   True    Helm install succeeded for release kommander/object-bucket-claims.v1 with chart object-bucket-claim@0.1.11
kommander                     prometheus-adapter            10d   True    Helm install succeeded for release kommander/prometheus-adapter.v1 with chart prometheus-adapter@4.11.0
kommander                     prometheus-traefik-certs      12d   True    Helm install succeeded for release kommander/prometheus-traefik-certs.v1 with chart kommander-cert-federation@0.0.11
kommander                     reloader                      12d   True    Helm install succeeded for release kommander/kommander-reloader.v1 with chart reloader@1.1.0
kommander                     rook-ceph                     10d   True    Helm install succeeded for release kommander/rook-ceph.v1 with chart rook-ceph@v1.14.5
kommander                     rook-ceph-cluster             10d   True    Helm install succeeded for release kommander/rook-ceph-cluster.v1 with chart rook-ceph-cluster@v1.14.5
kommander                     traefik                       12d   True    Helm install succeeded for release kommander/kommander-traefik.v1 with chart traefik@27.0.2
kommander                     traefik-forward-auth-mgmt     12d   True    Helm install succeeded for release kommander/traefik-forward-auth-mgmt.v1 with chart traefik-forward-auth@0.3.10
kommander                     velero                        10d   True    Helm install succeeded for release kommander/velero.v1 with chart velero@7.2.2

```

##### 執行升級指令 

大約20分鐘時間

```
[root@nkp-bastion ~]# cd /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/

[root@nkp-bastion nkp-v2.14.0]# nkp upgrade kommander \
--charts-bundle ./application-charts/nkp-kommander-charts-bundle-v2.14.0.tar.gz \
--kommander-applications-repository ./application-repositories/kommander-applications-v2.14.0.tar.gz
 ✓ Ensuring upgrading conditions are met
 ✓ Ensuring kubecost can be upgraded
 ✓ Ensuring DKP licenses are migrated (if any)
 ✓ Ensuring application definitions are updated 
 ✓ Ensuring helm-mirror implementation is migrated to ChartMuseum
 ✓ Ensuring root CA duration is upgraded
 ✓ Ensuring DKA config overrides are cleaned up
 ✓ Ensuring core Kommander application [kommander-appmanagement] is upgraded 
 ✓ Ensuring core Kommander applications [git-operator kommander kommander-flux kommander-ui] are upgraded 
 ✓ Ensuring new/updated management configOverrides configmaps are applied
 ✓ Installing COSI controller 
 ✓ Ensuring new default Kommander applications are enabled
 ✓ Ensuring 23 platform Kommander applications are upgraded [==================================>23/23] (time elapsed 11m07s) 
 ✓ Ensuring new Kommander applications are installed
```

![image-20250306171232968](https://kenkenny.synology.me:5543/images/2025/03/image-20250306171232968.png)

Kommander UI 已升級，並有新的應用程式出現 (Harbor, Nutanix COSI, CloudNativePG, kubecost)

![image-20250306171317317](https://kenkenny.synology.me:5543/images/2025/03/image-20250306171317317.png)

![image-20250306171430142](https://kenkenny.synology.me:5543/images/2025/03/image-20250306171430142.png)

![image-20250306171411257](https://kenkenny.synology.me:5543/images/2025/03/image-20250306171411257.png)

![image-20250306171452670](https://kenkenny.synology.me:5543/images/2025/03/image-20250306171452670.png)

#### 6. 升級 Cluster

確認環境變數

```
修改 VM_IMAGE_NAME

[root@nkp-bastion ~]# vi env.sh 

[root@nkp-bastion ~]# source env.sh 
[root@nkp-bastion ~]# echo $MANAGEMENT_CLUSTER_NAME
nkp-poc
[root@nkp-bastion ~]# echo $VM_IMAGE_NAME
nkp-ubuntu-22.04-1.31.4-20250306081227

[root@nkp-bastion nkp-v2.14.0]# kubectl get nodes
NAME                             STATUS   ROLES           AGE   VERSION
nkp-poc-d78fs-dkfnd              Ready    control-plane   12d   v1.30.5
nkp-poc-d78fs-gmllf              Ready    control-plane   12d   v1.30.5
nkp-poc-d78fs-x989r              Ready    control-plane   12d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-4jbfm   Ready    <none>          12d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-cxlx7   Ready    <none>          12d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-kwvlj   Ready    <none>          12d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-m9dkw   Ready    <none>          12d   v1.30.5
```

執行升級指令

![image-20250306172955285](https://kenkenny.synology.me:5543/images/2025/03/image-20250306172955285.png)

```
[root@nkp-bastion ~]# nkp upgrade cluster nutanix \
      --cluster-name ${MANAGEMENT_CLUSTER_NAME} \
      --vm-image ${VM_IMAGE_NAME}
 ✓ Deleting VCD Infrastructure Provider 
 ✓ Upgrading CAPI components 
 ✓ Updating ClusterClass resources 
cluster.cluster.x-k8s.io/nkp-poc upgraded
⠈⠱ Upgrading the cluster 

```

Prism Central 自動開始建立新的虛擬機加入叢集，Control 跟 worker 每一台採取先建後拆的方式自動升級

所以 IP 數量至少要多2個比較保險

![image-20250306172414800](https://kenkenny.synology.me:5543/images/2025/03/image-20250306172414800.png)

![image-20250306173056948](https://kenkenny.synology.me:5543/images/2025/03/image-20250306173056948.png)

```
用另一個 session 連線到 bastion 確認狀態

[nkp@nkp-bastion ~]$ kubectl get nodes
NAME                             STATUS     ROLES           AGE   VERSION
nkp-poc-d78fs-czzph              NotReady   control-plane   47s   v1.31.4
nkp-poc-d78fs-dkfnd              Ready      control-plane   12d   v1.30.5
nkp-poc-d78fs-gmllf              Ready      control-plane   13d   v1.30.5
nkp-poc-d78fs-x989r              Ready      control-plane   12d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-4jbfm   Ready      <none>          13d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-cxlx7   Ready      <none>          13d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-kwvlj   Ready      <none>          13d   v1.30.5
nkp-poc-md-0-clx8j-g5pnb-m9dkw   Ready      <none>          13d   v1.30.5

[nkp@nkp-bastion ~]$ kubectl get nodes -o wide
NAME                             STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
nkp-poc-d78fs-czzph              Ready    control-plane   7m12s   v1.31.4   10.38.14.43   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-d78fs-nqtx9              Ready    control-plane   3m48s   v1.31.4   10.38.14.40   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-d78fs-x989r              Ready    control-plane   13d     v1.30.5   10.38.14.30   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-4jbfm   Ready    <none>          13d     v1.30.5   10.38.14.31   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-cxlx7   Ready    <none>          13d     v1.30.5   10.38.14.50   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-kwvlj   Ready    <none>          13d     v1.30.5   10.38.14.48   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-m9dkw   Ready    <none>          13d     v1.30.5   10.38.14.47   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1

[nkp@nkp-bastion ~]$ kubectl get nodes
NAME                             STATUS     ROLES           AGE     VERSION
nkp-poc-d78fs-czzph              Ready      control-plane   12m     v1.31.4
nkp-poc-d78fs-mcn2q              Ready      control-plane   4m59s   v1.31.4
nkp-poc-d78fs-nqtx9              Ready      control-plane   8m47s   v1.31.4
nkp-poc-md-0-clx8j-g5pnb-4jbfm   Ready      <none>          13d     v1.30.5
nkp-poc-md-0-clx8j-g5pnb-cxlx7   Ready      <none>          13d     v1.30.5
nkp-poc-md-0-clx8j-g5pnb-kwvlj   Ready      <none>          13d     v1.30.5
nkp-poc-md-0-clx8j-g5pnb-m9dkw   Ready      <none>          13d     v1.30.5
nkp-poc-md-0-clx8j-gfbbp-sh8b2   NotReady   <none>          13s     v1.31.4

[nkp@nkp-bastion ~]$ kubectl get nodes -o wide
NAME                             STATUS     ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
nkp-poc-d78fs-czzph              Ready      control-plane   12m     v1.31.4   10.38.14.43   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-d78fs-mcn2q              Ready      control-plane   5m29s   v1.31.4   10.38.14.38   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-d78fs-nqtx9              Ready      control-plane   9m17s   v1.31.4   10.38.14.40   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-4jbfm   Ready      <none>          13d     v1.30.5   10.38.14.31   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-cxlx7   Ready      <none>          13d     v1.30.5   10.38.14.50   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-kwvlj   Ready      <none>          13d     v1.30.5   10.38.14.48   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-g5pnb-m9dkw   Ready      <none>          13d     v1.30.5   10.38.14.47   <none>        Ubuntu 22.04.5 LTS   5.15.0-133-generic   containerd://1.7.22-d2iq.1
nkp-poc-md-0-clx8j-gfbbp-sh8b2   NotReady   <none>          43s     v1.31.4   10.38.14.19   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
```

升級完成，大約花費30分鐘左右

![image-20250306175039600](https://kenkenny.synology.me:5543/images/2025/03/image-20250306175039600.png)

```
[nkp@nkp-bastion ~]$ kubectl get nodes -o wide
NAME                             STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
nkp-poc-d78fs-czzph              Ready    control-plane   26m     v1.31.4   10.38.14.43   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-d78fs-mcn2q              Ready    control-plane   19m     v1.31.4   10.38.14.38   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-d78fs-nqtx9              Ready    control-plane   22m     v1.31.4   10.38.14.40   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-md-0-clx8j-gfbbp-g6rtw   Ready    <none>          7m50s   v1.31.4   10.38.14.25   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-md-0-clx8j-gfbbp-qzrkh   Ready    <none>          3m6s    v1.31.4   10.38.14.30   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-md-0-clx8j-gfbbp-sh8b2   Ready    <none>          14m     v1.31.4   10.38.14.19   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1
nkp-poc-md-0-clx8j-gfbbp-xlmc7   Ready    <none>          11m     v1.31.4   10.38.14.27   <none>        Ubuntu 22.04.5 LTS   5.15.0-134-generic   containerd://1.7.24-d2iq.1

[nkp@nkp-bastion ~]$ kubectl get nodes
NAME                             STATUS   ROLES           AGE     VERSION
nkp-poc-d78fs-czzph              Ready    control-plane   26m     v1.31.4
nkp-poc-d78fs-mcn2q              Ready    control-plane   19m     v1.31.4
nkp-poc-d78fs-nqtx9              Ready    control-plane   23m     v1.31.4
nkp-poc-md-0-clx8j-gfbbp-g6rtw   Ready    <none>          7m59s   v1.31.4
nkp-poc-md-0-clx8j-gfbbp-qzrkh   Ready    <none>          3m15s   v1.31.4
nkp-poc-md-0-clx8j-gfbbp-sh8b2   Ready    <none>          14m     v1.31.4
nkp-poc-md-0-clx8j-gfbbp-xlmc7   Ready    <none>          12m     v1.31.4
```

Management 升級完成後之後再升級 workload cluster

以下為範例

```
# kubectl get cluster -A
# export  WORKLOAD_CLUSTER_NAME=demo-prod-01
# export  WORKLOAD_CLUSTER_NAMESPACE=demo-zone-c4zz7-qjq6g

# nkp upgrade cluster nutanix \
--cluster-name ${WORKLOAD_CLUSTER_NAME} \
--vm-image ${VM_IMAGE_NAME} -n WORKLOAD_CLUSTER_NAME
```

