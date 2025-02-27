---
title: Nutanix_NKP-v2.13_Preprovisioned
description: Nutanix_NKP-v2.13_Preprovisioned
slug: Nutanix_NKP-v2.13_Preprovisioned
date: 2025-02-27T09:42:42+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - NKP
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Nutanix NKP v2.13 Pre-Provisioned 環境

Pre-Provisioned 預配置的方式，可以讓 NKP 裝在任何環境，只要先安裝好 OS 以及基本的設定完成後

即可部署 NKP 叢集，不論是實體機或是虛擬機都可以安裝。



## 安裝大致流程

可調整資源、自定義設定，生產環境建議使用此方法 ，self-managed 僅支援此方法

1.確認安裝環境

2.連線 bootstrap host (Bastion Host)

3.確認 Container Service

4.確認 Kubernetes CLI

5.確認 NKP CLI

6.設定 Local Registry (air-gapped)

7.推送 NKP Image 到 Local Registry (air-gapped)

8.建立 bootstrap cluster (air-gapped or pre-provisioned or KIB or NIB )

9.建立 machine image (optional)

10.建立 management cluster 



## 環境資訊

此 LAB 僅用 1 個 control plane 加上 3 個 worker node 來測試，生產環境建議 3 個 control plane 加 4 個 worker node

NKP Version : 2.13

OS Image : Rocky 9.5



## Bastion Host 準備

### 安裝主機

創建虛擬機 Rocky 9.1 ，記憶體最少 8GB，最小安裝即可

![image-20250227172457416](https://kenkenny.synology.me:5543/images/2025/02/image-20250227172457416.png)

### 環境準備

設定網路、主機名稱、SSH，為簡化安裝流程選擇停用防火牆（Pre-Provisioned的節點需停用） 、
 關閉 SELinux，調整完後建議重開機確認，使用root or 一般使用者 + sudo 

```
設定網路：
# nmcli c s     >> 顯示網卡名稱
# nmcli c m ens3 ipv4.method manual ipv4.addresses xxx.xx.xx.xx/24 ipv4.gateway xxx.xx.xx.xxx ipv4.dns 8.8.8.8   >> 設定ip,dns,gateway
# nmcli c u  ens3   >> 啟用網卡

主機名稱：
# hostnamectl set-hostname nkp-bastion

SSH Key：
$ ssh-keygen >> 產生金鑰
$ ssh-copy-id -i .ssh/id_rsa.pub nkp@xxx.xx.xx.xx   >> 複製金鑰到其他 Host 上

停用防火牆：
# systemctl disable firewalld --now

關閉SELinux：
# vi /etc/selinux/config
SELINUX=disable

```

### Container Service & 工具

下載安裝NKP CLI、Container Service ( podman 4.0 or docker 20.10 以上 )、其他工具、Kubectl、KIB

請先確認環境所需的 NKP 以及 Kubernetes 版本，此範例用 NKP 2.13 對應 Kubernetes 1.30.5

```
Podman 下載安裝：
$ sudo yum install -y podman

$ podman version

or Docker 下載安裝：
$ sudo yum install -y docker-ce docker-ce-cli containerd.io

其他工具下載安裝：
$ sudo yum install -y yum-utils bzip2 wget tar

```

![image-20250227172655721](https://kenkenny.synology.me:5543/images/2025/02/image-20250227172655721.png)



### NKP CLI

下載安裝NKP CLI、Container Service ( podman 4.0 or docker 20.10 以上 )、其他工具、Kubectl、KIB

請先確認環境所需的 NKP 以及 Kubernetes 版本，此範例用 NKP 2.13 對應 Kubernetes 1.30.5



```
NKP CLI 下載：

$ curl -o nkp_v.2.13.0.tar.gz “https://download.nutanix.com/urlxxx”

NKP CLI 安裝：

$ tar xvf nkp_v2.13.0.tar.gz
$ ls -l
$ sudo cp nkp /usr/local/bin
$ nkp version
```

![image-20250227173028571](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173028571.png)

![image-20250227173015476](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173015476.png)

![image-20250227173035349](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173035349.png)

### Kubectl

```
Kubectl 安裝：
https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.30.md#downloads-for-v1305

$ wget "https://dl.k8s.io/v1.30.5/kubernetes-client-linux-amd64.tar.gz"
$ tar xvf kubernetes-client-linux-amd64.tar.gz
$ sudo cp kubernetes/client/bin/kubectl /usr/local/bin/
$ kubectl version

```

![image-20250227173058472](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173058472.png)

### **KIB**

```
KIB Konvoy Image Builder 下載安裝 ( 有需要再裝 )：

$ curl -o konvoy-image-bundle-v2.18.0.tar.gz ”https://url”
$ tar xvf konvoy-image-bundle-v2.18.0.tar.gz
$ ls -l
$ sudo cp konvoy-image /usr/local/bin
$ konvoy-image version

```

![image-20250227173137067](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173137067.png)

## 預配置主機準備

```
Control Plane & Worker Machines 建議：

control plane*3 :  4 cores , 16 GiB 
worker node*4 : 8 cores , 32 GiB
root 目錄 15% 可用空間， /var/lib/kubelet 以及 /var/lib/containerd 95 GiB 以上空間
預設Storage Class 是用 localvolume 所以 /mnt/disks 底下至少要有mount 一顆 55GiB 硬碟 (官方建議4顆)
停用防火牆 firewalld、swap
防火牆需求開通 nutanix support portal
root 或是不用密碼執行 sudo 的使用者，並先跟Bastion做金鑰交換

至少 1 個 Loadbalancer IP 或是設定外部 Loadbalance ( 含 api 及 Ingress )

OS images:
Air-gapped 環境需先準備好用 Konvoy Image Builder 產生出需要的安裝包並用 konvoy upload 上傳至節點



Control Plane * 1：
	8 vCPU , 8 GiB RAM , 120 GiB 根目錄 , 60 GiB /mnt/disks/nkp1 一顆
	最小安裝,另外安裝 tar , 停用 firewalld , swap , selinux

Worker Machines * 3 ：
	4 vCPU , 8 GiB RAM , 120 GiB 根目錄 , 60 GiB /mnt/disks/nkp1 一顆
	最小安裝,另外安裝 tar , 停用 firewalld , swap , selinux

```

![image-20250227172309690](https://kenkenny.synology.me:5543/images/2025/02/image-20250227172309690.png)

![image-20250227172316729](https://kenkenny.synology.me:5543/images/2025/02/image-20250227172316729.png)

## 部署 NKP Pre-provisioned



### **定義環境變數** 

此範例使用 root 使用者，如要其他使用者請先設定好Docker or Podman 權限

並且在 control 與 worker 上要設定 /etc/sudoers “<username> ALL=(ALL:ALL) NOPASSWD: ALL”

此為 single control plane 配置，生產環境建議至少 3 個 control plane，4 個 worker node

Storage Class 建議用外部 Storage

```
在 Bastion 上定義環境變數
export CONTROL_PLANE_1_ADDRESS="172.16.90.131"
export WORKER_1_ADDRESS="172.16.90.132"
export WORKER_2_ADDRESS="172.16.90.133"
export WORKER_3_ADDRESS="172.16.90.134"
export SSH_USER="root"
export SSH_PRIVATE_KEY_SECRET_NAME="$CLUSTER_NAME-ssh-key"
export CLUSTER_NAME="nkp-pro-preprovisioned” >> .bashrc
source .bashrc

```

### **inventory.yaml** 

從 NKP 文件複製下來

確認 preprovisioned_inventory.yaml  內容是否正確
vi preprovisioned_inventory.yaml

確認control plane 、worker 數量及IP、secret 名稱等等

![image-20250227173246700](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173246700.png)

![image-20250227173255503](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173255503.png)

### bootstrap

```
建立 bootstrap cluster 並部署 preprovisioned_inventory.yaml
# nkp create bootstrap
# kubectl get pods -A
# kubectl apply -f preprovisioned_inventory.yaml
建立 override.yaml 設定 docker.io or Registry 及帳號密碼
# kubectl create secret generic $CLUSTER_NAME-user-overrides --from-file=overrides.yaml=overrides.yaml 
# kubectl label secret $CLUSTER_NAME-user-overrides clusterctl.cluster.x-k8s.io/move= 

```

![image-20250227173347739](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173347739.png)

![image-20250227173352792](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173352792.png)

![image-20250227173400637](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173400637.png)

### Dry run

```
部署NKP Preprovisioned Cluster (non-airgapped)，使用單節點或是MetalLB，依需求修改下方資訊
Kube-VIP 方式部署則需要加上--virtual-ip-interface eth1 

nkp create cluster preprovisioned \
  --cluster-name=${CLUSTER_NAME} \
  --control-plane-endpoint-host=<control plane endpoint host> \
  --control-plane-replicas=1 \
  --worker-replicas=3 \
  --ssh-private-key-file=<path-to-ssh-private-key> \
  --registry-mirror-url=https://registry-1.docker.io \
  --registry-mirror-username=xxx \
  --registry-mirror-password=xxx \
  --dry-run \
  --output=yaml > deploy-nkp-$CLUSTER_NAME.yaml


檢查 deploy-nkp-$CLUSTER_NAME.yaml
# vi deploy-nkp-nkp-pro-preprovisioned.yaml
```

![image-20250227173502810](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173502810.png)

### Create Cluster

```
部署 nkp cluster

# kubectl create -f deploy-nkp-nkp-pro-preprovisioned.yaml

觀察狀態

# kubectl wait --for=condition=ControlPlaneReady "clusters/${CLUSTER_NAME}" --timeout=30m
```

![image-20250227173547345](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173547345.png)

確認 Cluster

```
# kubectl get cluster
# kubectl describe cluster
```

![image-20250227173613836](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173613836.png)

![image-20250227173618232](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173618232.png)

#### 取得kubeconfig

```
取得kubeconfig檔案
# nkp get kubeconfig -c ${CLUSTER_NAME} > ${CLUSTER_NAME}.conf
# kubectl get nodes --kubeconfig=${CLUSTER_NAME}.conf
# kubectl get pods -A --kubeconfig=${CLUSTER_NAME}.conf
```

![image-20250227173719200](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173719200.png)

#### **Pivot**

原bootstrap 部署出來只是 workload cluster，此動作是要將此叢集變成 Self-Manged or Management

```
將 Cluster API 元件部署到 workload cluster 
# nkp create capi-components --kubeconfig=${CLUSTER_NAME}.conf

再將 life cycle services 從 bootstrap 移轉到 workload cluster，並確認NKP describe cluster
# nkp move capi-resources --to-kubeconfig ${CLUSTER_NAME}.conf
# nkp describe cluster --kubeconfig ${CLUSTER_NAME}.conf -c ${CLUSTER_NAME}
```

![image-20250227173802640](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173802640.png)

#### **MetalLB**

**如果沒有外部** **LoadBalancer** **的話，可以使用** **MetalLB** **，需設定** **Loadbalancer IP Pool**

**IP** **網段需與** **Machine** **同網段，可以只設定一個** **Ex. 192.168.1.240-192.168.1.240**

**原廠文件提供範例，調整** **addresses** **後** **apply** **該設定檔**

\# kubectl apply -f metallb-conf.yaml --kubeconfig ${CLUSTER_NAME}.conf 

![image-20250227173825119](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173825119.png)

#### Install Kommander

**移除** **bootstrap cluster** 

\# nkp delete bootstrap --kubeconfig $HOME/.kube/config

**取得** **kommander.yaml**

\# nkp install kommander --init > kommander.yaml 

**預設** **storage class** **是** **localvolumeprovisioner**，**volume mode** **無法為** **Block** **，
** **故需修改** **yaml** **調整** **rook-ceph-cluster** **設定，****原則上建議使用外部** **Storage Class** **( NFS or Nutanix CSI )**

**下方服務不需要的可以設成** **false** **最小安裝參照** [https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_13:top-addl-kommander-config-c](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_13:top-addl-kommander-config-c.html)[.html](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_13:top-addl-kommander-config-c.html)

![image-20250227173900072](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173900072.png)

**設定** **NKP Catalog Applications，並部署 **Kommander

\# nkp install kommander --installer-config kommander.yaml --kubeconfig=${CLUSTER_NAME}.conf --wait-timeout=1h

![image-20250227173923224](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173923224.png)

**觀察安裝過程**

\# watch kubectl get pod --n kommander

\# kubectl -n kommander wait --for condition=Ready helmreleases --all --timeout 15m

![image-20250227173938405](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173938405.png)

取得 UI 連結、帳號密碼

\# nkp open dashboard --kubeconfig=${CLUSTER_NAME}.conf

![image-20250227173956097](https://kenkenny.synology.me:5543/images/2025/02/image-20250227173956097.png)
