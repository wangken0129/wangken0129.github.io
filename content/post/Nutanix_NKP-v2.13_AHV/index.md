---
title: Nutanix_NKP-v2.13_AHV
description: Nutanix_NKP-v2.13_AHV
slug: Nutanix_NKP-v2.13_AHV
date: 2025-02-25T07:14:25+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - NKP
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Nutanix NKP v2.13.1 Install (air-gapped)

NKP 安裝 Lab，安裝在 AHV 環境上 ，透過 cli 方式安裝



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

[nkp@nkp-bastion ~]$ getenforce
Permissive

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
export SSH_KEY_FILE="/home/nkp/.ssh/id_rsa.pub"
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

