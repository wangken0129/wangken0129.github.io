---
title: Nutanix_NKP-v2.12_AHV
description: Nutanix_NKP-v2.12_AHV
slug: Nutanix_NKP-v2.12_AHV
date: 2025-02-24T07:52:30+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - NKP
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Nutanix NKP v2.12 For AHV

Nutanix 在2024年收購了D2iQ這間公司，並且將DKP重新調整為NKP

讓Nutanix的客戶可以在原來的NCI基礎上使用多叢集管理的Kubernetes平台也取代了原本的NKE

平台功能有包含叢集的監控與Log、負載平衡、SSO、生命週期管理、Service Mesh等等

主要的競爭者應該為Rancher、Anthos 等等，並沒有像是OCP有開發者介面，所以還是著重在管理上。



安裝選擇 NKP on Nutanix AHV (non-airgap)

## Reference

https://www.nutanix.com/tech-center/blog/nkp-kubernetes-done-the-nutanix-way

https://www.nutanix.com/blog/nutanix-announces-nutanix-kubernetes-platform

https://www.nutanix.com/products/kubernetes-management-platform

https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_12:Nutanix-Kubernetes-Platform-v2_12

（非官方）

https://jonashogman.com/nkp-post-1-preparing-the-setup-environment/ 

https://blog.ntnx.jp/entry/2024/09/16/235808 

## Architecture

NKP 以目前的文件來看，可以安裝在Nutanix、vSphere、Bare metal (pre-provisioned)、EKS、AKS、Azure、GCP等等

各安裝的方法以及需求皆有所不同 (airgap or non-airgap 這些都要考慮到)。

架構主要分為三種型態，以及另一種 Self-Managed Cluster

1. Management Cluster：

   安裝 NKP 的上游叢集 (Upstream Cluster)，負責管理多個叢集，不負責使用者工作負載，

   可以管理部署出來的Managed Cluster，也可以管理連接上來的 Attached Cluster。

2. Managed Cluster：

   由 NKP Management Cluster 部署出來的叢集，又稱 NKP Cluster ，主要負責使用者工作負載，

   並透過Management Cluster來管理生命週期跟應用程式。

3. Attached Cluster：

   這是非NKP Management Cluster 部署出來的外部叢集，像是 EKS、AKS 或是其他認證的 Kubernetes 叢集(目前無認證清單)，

   以D2iQ的[文件](https://docs.d2iq.com/dkp/2.7/attach-an-existing-kubernetes-cluster)來說是All Kubernetes Conformant clusters (符合、一致的？)

   透過Management Cluster來管理應用程式，生命週期由原來的地方管理。

   

   <img src="https://kenkenny.synology.me:5543/images/2024/09/image-20240930155523134.png" alt="image-20240930155523134" style="zoom:50%;" />

4. Self-Managed Cluster：

   一樣是裝 NKP，但非上游叢集，可以跑使用者工作負載，不能 Attach 其他叢集做管理。



整體架構圖：

![img](https://kenkenny.synology.me:5543/images/2024/09/nkp-architecture.png)

多叢集環境架構圖：

![img](https://kenkenny.synology.me:5543/images/2024/09/multi-cluster-env.png)

NKP Self-Managed Cluster：

![img](https://kenkenny.synology.me:5543/images/2024/09/single-cluster-env.png)

## Prerequisites

### Prism Central Role

(LAB環境建議是用admin)

預設已有Kubernetes Infrastructure Provisions role 

需要新增Attach&Detach Volume Group To AHV VM、Update VM Disk、view container的權限、Create Volume Group Disk、

View VM Stats、Category(預設的有此權限，但實際安裝發現是 CSI 使用API v4的去呼叫)、View Prism Central 等等 

並且新增一個 local user 設定為此角色即可

這裡為 k8sadmin 這個user account

![image-20240919113110948](https://kenkenny.synology.me:5543/images/2024/09/image-20240919113110948.png)

![image-20240930180523023](https://kenkenny.synology.me:5543/images/2024/09/image-20240930180523023.png)

新增Category權限(apiv4)

![image-20241001092859405](https://kenkenny.synology.me:5543/images/2024/10/image-20241001092859405.png)

![image-20241030133619055](https://kenkenny.synology.me:5543/images/2024/10/image-20241030133619055.png)

![image-20241030140157113](https://kenkenny.synology.me:5543/images/2024/10/image-20241030140157113.png)

![image-20241030140047715](https://kenkenny.synology.me:5543/images/2024/10/image-20241030140047715.png)

![image-20241030134006664](https://kenkenny.synology.me:5543/images/2024/10/image-20241030134006664.png)

### Resource Requirements

以下為各環境所需的資源，基本上還需另外一台Bastion作為安裝使用。

1. NKP PRO / Ultimate Management Cluster： 3 control plane 、 4 worker node

![image-20240930163551021](https://kenkenny.synology.me:5543/images/2024/09/image-20240930163551021.png)

2. NKP PRO / Ultimate Managed Cluster： 3 control plane 、 4 worker node 

3 worker node 適用於不啟用 velero or logging stack 的叢集。

![image-20240930164501193](https://kenkenny.synology.me:5543/images/2024/09/image-20240930164501193.png)

3. NKP Starter Cluster（僅適用於Nutanix 的環境）：  3 control plane、最低 2 worker node

control plane 文件沒特別寫數量，但還是要三個，另外有支援single control plane的叢集，僅適用於測試環境。

![image-20240930164224865](https://kenkenny.synology.me:5543/images/2024/09/image-20240930164224865.png)

![image-20240930164238522](https://kenkenny.synology.me:5543/images/2024/09/image-20240930164238522.png)



### Sofware Requirements

For AHV 環境，NKP 相關的軟體可以在Nutanix Support Portal 上下載

1. NKP Node Image

   Nutanix 有包好 Rocky Linux for NKP 的 image，另外也可以選用Ubuntu 要另外用Konvoy Image Builder去做

   差別在於GPU Passthrough 是否支援，此 Lab 就用Rocky Linux實作

   ![image-20240919113329110](https://kenkenny.synology.me:5543/images/2024/09/image-20240919113329110.png)

   

   1. 在Support Portal 複製Download Link

   ![image-20240919113253425](https://kenkenny.synology.me:5543/images/2024/09/image-20240919113253425.png)

   2. 上傳到Prism Central 即可

   ![image-20240919113231539](https://kenkenny.synology.me:5543/images/2024/09/image-20240919113231539.png)

   ![image-20240919113639830](https://kenkenny.synology.me:5543/images/2024/09/image-20240919113639830.png)

   

2. An x86_64 Linux or MacOS machine

3. NKP Binary File

   ![image-20240930170142468](https://kenkenny.synology.me:5543/images/2024/09/image-20240930170142468.png)

4. container runtime

5. kubectl

6. Konvoy Image Builder in KIB

7. NKP air-gapped bundle -> **Air-gapped Only (additional prerequisites)**  

8. Local image registry -> **Air-gapped Only (additional prerequisites)**  



### Nutanix prerequisites

Prism Central : 2024.1 later

AOS: 6.5 , 6.8 later

IP Address for nodes : 7

IP Address for API : 1

IP Address for Traefik (metallb) : 1



### Bastion

#### Software

已預先安裝好RHEL v9.3 、Podman v4.2.0、kubectl v5.0.4 for v.1.28.0 ，並下載 nkp binary、konvoy-image-bundle

另外要準備docker image的帳號密碼，不然下載檔案太多會被擋下來

![image-20240930171618399](https://kenkenny.synology.me:5543/images/2024/09/image-20240930171618399.png)

![image-20240930172056841](https://kenkenny.synology.me:5543/images/2024/09/image-20240930172056841.png)

![image-20240930172045210](https://kenkenny.synology.me:5543/images/2024/09/image-20240930172045210.png)

#### SSH-Key

```
[nkp@ken-rhel9 ~]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/nkp/.ssh/id_rsa):
Created directory '/home/nkp/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/nkp/.ssh/id_rsa
Your public key has been saved in /home/nkp/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:r5lyQDXM972bfYgElhrfz1bkTR2x9GwGIeIP918fbiI nkp@ken-rhel9.nutanixlab.local
The key's randomart image is:
+---[RSA 3072]----+
|       o  . . o+.|
|        =... ..+o|
|       . ooo.. .B|
|      . . ++...o+|
|     .  S= o. .*o|
|      . ... o o.B|
|       .  .E = Oo|
|      . .+  o X o|
|       o+    .  .|
+----[SHA256]-----+
```

#### untar files

```

[nkp@ken-rhel9 ~]$ tar -xvf nkp_v2.12.0_linux_amd64.tar.gz
NOTICES
nkp
[nkp@ken-rhel9 ~]$ tar -xvf konvoy-image-bundle-v2.13.2_linux_amd64.tar.gz
LICENSE
README.md
ansible/.tool-versions
ansible/.yamllint
...

[nkp@ken-rhel9 ~]$ sudo mv nkp konvoy-image /usr/local/bin/
```



## Installation

### nkp create cluster nutanix

1. nkp create cluster

   ```
   [nkp@ken-rhel9 ~]$ nkp create cluster nutanix
   
   image registry: https://registry-1.docker.io
   
   Subnet 需要IPAM
   ```

   ![image-20240930173457822](https://kenkenny.synology.me:5543/images/2024/09/image-20240930173457822.png)

   ![image-20240930174329479](https://kenkenny.synology.me:5543/images/2024/09/image-20240930174329479.png)

   ![image-20240930175545467](https://kenkenny.synology.me:5543/images/2024/09/image-20240930175545467.png)

   ![image-20240930175602735](https://kenkenny.synology.me:5543/images/2024/09/image-20240930175602735.png)

   ![image-20240930175735795](https://kenkenny.synology.me:5543/images/2024/09/image-20240930175735795.png)

   ![image-20240930175810479](https://kenkenny.synology.me:5543/images/2024/09/image-20240930175810479.png)

   ![image-20240930175903070](https://kenkenny.synology.me:5543/images/2024/09/image-20240930175903070.png)

   ![image-20240930180010307](https://kenkenny.synology.me:5543/images/2024/09/image-20240930180010307.png)

2. 切換成root、PC 新增container 權限

   ![image-20240930180346824](https://kenkenny.synology.me:5543/images/2024/09/image-20240930180346824.png)

   ![image-20240930180626964](https://kenkenny.synology.me:5543/images/2024/09/image-20240930180626964.png)

   ![image-20240930180755815](https://kenkenny.synology.me:5543/images/2024/09/image-20240930180755815.png)

   

   ![image-20240930181036285](https://kenkenny.synology.me:5543/images/2024/09/image-20240930181036285.png)

   ![image-20240930181112546](https://kenkenny.synology.me:5543/images/2024/09/image-20240930181112546.png)

   ![image-20240930181232862](https://kenkenny.synology.me:5543/images/2024/09/image-20240930181232862.png)

   ![image-20240930181324796](https://kenkenny.synology.me:5543/images/2024/09/image-20240930181324796.png)

   ![image-20240930181941554](https://kenkenny.synology.me:5543/images/2024/09/image-20240930181941554.png)

   ![image-20240930182250767](https://kenkenny.synology.me:5543/images/2024/09/image-20240930182250767.png)

   ![image-20241001091037144](https://kenkenny.synology.me:5543/images/2024/10/image-20241001091037144.png)

3. Check logs

   ```
   [nkp@ken-rhel9 ~]$ kubectl --kubeconfig="./nkp-mgmt.conf" get pods -A
   NAMESPACE                           NAME                                                                                                          READY   STATUS      RESTARTS       AGE
   caaph-system                        caaph-controller-manager-7557bfbd76-25psr                                                                     1/1     Running     1 (149m ago)   15h
   capa-system                         capa-controller-manager-549fff5698-4js7x                                                                      1/1     Running     0              15h
   capg-system                         capg-controller-manager-54df78c867-ngvkg                                                                      1/1     Running     0              15h
   capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-589f87d995-ph2ll                                                    1/1     Running     0              15h
   capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-869b46bd8d-ccbl4                                                1/1     Running     0              15h
   capi-system                         capi-controller-manager-7bd8c69994-6562l                                                                      1/1     Running     0              15h
   cappp-system                        cappp-controller-manager-6cc595974c-nphps                                                                     1/1     Running     0              15h
   capv-system                         capv-controller-manager-7c6679579f-qdvmx                                                                      1/1     Running     0              15h
   capvcd-system                       capvcd-controller-manager-647bbc5685-8x8p8                                                                    1/1     Running     0              15h
   capx-system                         capx-controller-manager-6b4c9976f6-zjw5r                                                                      1/1     Running     0              15h
   capz-system                         azureserviceoperator-controller-manager-697d757c4b-ffrcs                                                      2/2     Running     0              15h
   capz-system                         capz-controller-manager-75c684766c-7z29w                                                                      1/1     Running     0              15h
   caren-system                        cluster-api-runtime-extensions-nutanix-869f6bc85d-wr864                                                       1/1     Running     0              15h
   caren-system                        helm-repository-86c695db8f-w6xwc                                                                              1/1     Running     0              15h
   cert-manager                        cert-manager-c49657b87-hkgdf                                                                                  1/1     Running     0              15h
   cert-manager                        cert-manager-cainjector-7b9b545679-l6pjf                                                                      1/1     Running     0              15h
   cert-manager                        cert-manager-webhook-647c6946df-cdtl9                                                                         1/1     Running     0              15h
   default                             cluster-autoscaler-01924266-3064-7c78-9b30-7e4445c87fa1-85txw8                                            l   1/1     Running     0              15h
   git-operator-system                 git-operator-admin-credentials-rotate-28795680-7fpm9                                                          0/1     Completed   0              81m
   git-operator-system                 git-operator-controller-manager-585cc87d78-h47zp                                                              2/2     Running     1 (80m ago)    15h
   git-operator-system                 git-operator-git-0                                                                                            0/3     Pending     0              15h
   kommander-flux                      helm-controller-dc4455cd5-rlpdg                                                                               1/1     Running     0              15h
   kommander-flux                      kustomize-controller-755bbdfc55-dmf4z                                                                         1/1     Running     0              15h
   kommander-flux                      notification-controller-86c44d47f9-bbm9b                                                                      1/1     Running     0              15h
   kommander-flux                      source-controller-68bc9cbf4d-zgfd7                                                                            1/1     Running     0              15h
   kommander                           kommander-bootstrap-mmq8g                                                                                     1/1     Running     0              15h
   kommander                           runtime-extension-kommander-59d9c676b7-fjqrv                                                                  1/1     Running     0              15h
   kube-system                         cilium-4vt4l                                                                                                  1/1     Running     0              15h
   kube-system                         cilium-7gjsw                                                                                                  1/1     Running     0              15h
   kube-system                         cilium-gkbcm                                                                                                  1/1     Running     0              15h
   kube-system                         cilium-operator-58c5d5d6d-dtknv                                                                               1/1     Running     0              15h
   kube-system                         cilium-operator-58c5d5d6d-f7x2w                                                                               1/1     Running     0              15h
   kube-system                         cilium-q7bnb                                                                                                  1/1     Running     0              15h
   kube-system                         cilium-qr94t                                                                                                  1/1     Running     0              15h
   kube-system                         cilium-r8k72                                                                                                  1/1     Running     0              15h
   kube-system                         cilium-wlvds                                                                                                  1/1     Running     0              15h
   kube-system                         coredns-76f75df574-wrg54                                                                                      1/1     Running     0              15h
   kube-system                         coredns-76f75df574-zqcr8                                                                                      1/1     Running     0              15h
   kube-system                         etcd-nkp-mgmt-xxl2n-7fdv6                                                                                     1/1     Running     0              15h
   kube-system                         etcd-nkp-mgmt-xxl2n-cc69v                                                                                     1/1     Running     0              15h
   kube-system                         etcd-nkp-mgmt-xxl2n-z5c49                                                                                     1/1     Running     0              15h
   kube-system                         kube-apiserver-nkp-mgmt-xxl2n-7fdv6                                                                           1/1     Running     0              15h
   kube-system                         kube-apiserver-nkp-mgmt-xxl2n-cc69v                                                                           1/1     Running     0              15h
   kube-system                         kube-apiserver-nkp-mgmt-xxl2n-z5c49                                                                           1/1     Running     0              15h
   kube-system                         kube-controller-manager-nkp-mgmt-xxl2n-7fdv6                                                                  1/1     Running     0              15h
   kube-system                         kube-controller-manager-nkp-mgmt-xxl2n-cc69v                                                                  1/1     Running     0              15h
   kube-system                         kube-controller-manager-nkp-mgmt-xxl2n-z5c49                                                                  1/1     Running     0              15h
   kube-system                         kube-proxy-45pft                                                                                              1/1     Running     0              15h
   kube-system                         kube-proxy-5dv8f                                                                                              1/1     Running     0              15h
   kube-system                         kube-proxy-bh7hq                                                                                              1/1     Running     0              15h
   kube-system                         kube-proxy-fm4gj                                                                                              1/1     Running     0              15h
   kube-system                         kube-proxy-hkx6m                                                                                              1/1     Running     0              15h
   kube-system                         kube-proxy-hmzqk                                                                                              1/1     Running     0              15h
   kube-system                         kube-proxy-zvj58                                                                                              1/1     Running     0              15h
   kube-system                         kube-scheduler-nkp-mgmt-xxl2n-7fdv6                                                                           1/1     Running     0              15h
   kube-system                         kube-scheduler-nkp-mgmt-xxl2n-cc69v                                                                           1/1     Running     0              15h
   kube-system                         kube-scheduler-nkp-mgmt-xxl2n-z5c49                                                                           1/1     Running     0              15h
   kube-system                         kube-vip-nkp-mgmt-xxl2n-7fdv6                                                                                 1/1     Running     0              15h
   kube-system                         kube-vip-nkp-mgmt-xxl2n-cc69v                                                                                 1/1     Running     0              15h
   kube-system                         kube-vip-nkp-mgmt-xxl2n-z5c49                                                                                 1/1     Running     0              15h
   kube-system                         nutanix-cloud-controller-manager-598b4c7669-dd2nl                                                             1/1     Running     0              15h
   kube-system                         snapshot-controller-5c7f9fc58-dlbm2                                                                           1/1     Running     0              15h
   metallb-system                      metallb-controller-94f95d674-24tnf                                                                            1/1     Running     0              15h
   metallb-system                      metallb-speaker-587jl                                                                                         4/4     Running     0              15h
   metallb-system                      metallb-speaker-h4g7r                                                                                         4/4     Running     0              15h
   metallb-system                      metallb-speaker-kxbp4                                                                                         4/4     Running     0              15h
   metallb-system                      metallb-speaker-qxhsr                                                                                         4/4     Running     0              15h
   metallb-system                      metallb-speaker-v8r8j                                                                                         4/4     Running     0              15h
   metallb-system                      metallb-speaker-wdkbx                                                                                         4/4     Running     0              15h
   metallb-system                      metallb-speaker-xqvbj                                                                                         4/4     Running     0              15h
   node-feature-discovery              node-feature-discovery-gc-7f54d58d99-slpr2                                                                    1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-master-ccf75997b-86x9c                                                                 1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-worker-6t9lf                                                                           1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-worker-78c8h                                                                           1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-worker-f25gw                                                                           1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-worker-l6q5k                                                                           1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-worker-nk99v                                                                           1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-worker-rjj8w                                                                           1/1     Running     0              15h
   node-feature-discovery              node-feature-discovery-worker-rq5dm                                                                           1/1     Running     0              15h
   ntnx-system                         nutanix-csi-precheck-job-nmkj7                                                                                0/1     Error       0              149m
   [nkp@ken-rhel9 ~]$ kubectl --kubeconfig="./nkp-mgmt.conf" logs -f ^C
   [nkp@ken-rhel9 ~]$ kubectl --kubeconfig="./nkp-mgmt.conf" logs   nutanix-csi-precheck-job-nmkj7  -n ntnx-system
   W0930 22:51:08.489458       1 client_config.go:614] Neither --kubeconfig nor --master was specified.  Using the inClust                       erConfig.  This might not work.
   2024-09-30T22:51:08.493Z client.go:227: [INFO] nutanix_files: rest request method:GET path: /cluster/version, data: <ni                       l>
   2024-09-30T22:51:08.726Z prism_central.go:31: [INFO] PC Version Info pc.2024.1.0.1
   2024-09-30T22:51:08.726Z main.go:252: [INFO] PC version:pc.2024.1.0.1 is supported
   2024-09-30T22:51:08.74Z main.go:145: [WARN] Error getting the data from ConfigMap ntnx-cluster-configmap/ntnx-system :                        configmaps "ntnx-cluster-configmap" not found
   2024-09-30T22:51:08.74Z main.go:164: [WARN] Failed to get categories from ntnx-system/ntnx-cluster-configmap, err: conf                       igmaps "ntnx-cluster-configmap" not found, will use kube-system uuid as category
   2024-09-30T22:51:08.745Z main.go:174: [INFO] Cluster UUID: k8s-d92bdb3c-dc75-4132-a59d-61d5f3531913
   2024-09-30T22:51:08.746Z categories.go:48: [INFO] creating a new category api client
   2024-09-30T22:51:08.746Z main.go:200: [INFO] Category to be associated to CSI Provisioned Vol KubernetesClusterUUID/k8s                       -d92bdb3c-dc75-4132-a59d-61d5f3531913
   2024-09-30T22:51:08.746Z categories_idempotent.go:35: [INFO] Retrieving categories for key "KubernetesClusterUUID"
   2024-09-30 22:51:23.968 INFO - GET https://172.16.90.75:9440/api/prism/v4.0.b1/config/categories?%24filter=key+in+%28%2                       7KubernetesClusterUUID%27%29
   2024-09-30 22:51:23.979 INFO - HTTP/1.1 403 FORBIDDEN
   2024-09-30T22:51:23.98Z categories.go:115: [ERROR] Failed to get all categories with status: 403 FORBIDDEN, error: {"da                       ta":{"error":[{"message":"Access Denied because no or incorrect permissions were set in the request http://172.16.90.75                       /api/prism/v4.0.b1/config/categories","code":"RBA-10001","locale":"en_US","errorGroup":"RBAC_AUTHORIZATION_ERROR","seve                       rity":"ERROR","$objectType":"prism.v4.error.AppMessage"}],"$errorItemDiscriminator":"List<prism.v4.error.AppMessage>","                       $objectType":"prism.v4.error.ErrorResponse"},"$dataItemDiscriminator":"prism.v4.error.ErrorResponse"}
   2024-09-30T22:51:23.98Z categories_idempotent.go:94: [ERROR] failed to get category for fqName "KubernetesClusterUUID/k                       8s-d92bdb3c-dc75-4132-a59d-61d5f3531913", err: %!w(*fmt.wrapError=&{Failed to get all categories with status: 403 FORBI                       DDEN, error: {"data":{"error":[{"message":"Access Denied because no or incorrect permissions were set in the request ht                       tp://172.16.90.75/api/prism/v4.0.b1/config/categories","code":"RBA-10001","locale":"en_US","errorGroup":"RBAC_AUTHORIZA                       TION_ERROR","severity":"ERROR","$objectType":"prism.v4.error.AppMessage"}],"$errorItemDiscriminator":"List<prism.v4.err                       or.AppMessage>","$objectType":"prism.v4.error.ErrorResponse"},"$dataItemDiscriminator":"prism.v4.error.ErrorResponse"}                        {[123 34 100 97 116 97 34 58 123 34 101 114 114 111 114 34 58 91 123 34 109 101 115 115 97 103 101 34 58 34 65 99 99 10                       1 115 115 32 68 101 110 105 101 100 32 98 101 99 97 117 115 101 32 110 111 32 111 114 32 105 110 99 111 114 114 101 99                        116 32 112 101 114 109 105 115 115 105 111 110 115 32 119 101 114 101 32 115 101 116 32 105 110 32 116 104 101 32 114 1                       01 113 117 101 115 116 32 104 116 116 112 58 47 47 49 55 50 46 49 54 46 57 48 46 55 53 47 97 112 105 47 112 114 105 115                        109 47 118 52 46 48 46 98 49 47 99 111 110 102 105 103 47 99 97 116 101 103 111 114 105 101 115 34 44 34 99 111 100 10                       1 34 58 34 82 66 65 45 49 48 48 48 49 34 44 34 108 111 99 97 108 101 34 58 34 101 110 95 85 83 34 44 34 101 114 114 111                        114 71 114 111 117 112 34 58 34 82 66 65 67 95 65 85 84 72 79 82 73 90 65 84 73 79 78 95 69 82 82 79 82 34 44 34 115 1                       01 118 101 114 105 116 121 34 58 34 69 82 82 79 82 34 44 34 36 111 98 106 101 99 116 84 121 112 101 34 58 34 112 114 10                       5 115 109 46 118 52 46 101 114 114 111 114 46 65 112 112 77 101 115 115 97 103 101 34 125 93 44 34 36 101 114 114 111 1                       14 73 116 101 109 68 105 115 99 114 105 109 105 110 97 116 111 114 34 58 34 76 105 115 116 60 112 114 105 115 109 46 11                       8 52 46 101 114 114 111 114 46 65 112 112 77 101 115 115 97 103 101 62 34 44 34 36 111 98 106 101 99 116 84 121 112 101                        34 58 34 112 114 105 115 109 46 118 52 46 101 114 114 111 114 46 69 114 114 111 114 82 101 115 112 111 110 115 101 34                        125 44 34 36 100 97 116 97 73 116 101 109 68 105 115 99 114 105 109 105 110 97 116 111 114 34 58 34 112 114 105 115 109                        46 118 52 46 101 114 114 111 114 46 69 114 114 111 114 82 101 115 112 111 110 115 101 34 125] <nil> 403 FORBIDDEN}})
   2024-09-30T22:51:23.98Z categories_idempotent.go:74: [ERROR] Failed to create category key for fqName "KubernetesCluste                       rUUID/k8s-d92bdb3c-dc75-4132-a59d-61d5f3531913" ,err: %!w(string=Max retries done: Failed to get all categories with st                       atus: 403 FORBIDDEN, error: {"data":{"error":[{"message":"Access Denied because no or incorrect permissions were set in                        the request http://172.16.90.75/api/prism/v4.0.b1/config/categories","code":"RBA-10001","locale":"en_US","errorGroup":                       "RBAC_AUTHORIZATION_ERROR","severity":"ERROR","$objectType":"prism.v4.error.AppMessage"}],"$errorItemDiscriminator":"Li                       st<prism.v4.error.AppMessage>","$objectType":"prism.v4.error.ErrorResponse"},"$dataItemDiscriminator":"prism.v4.error.E                       rrorResponse"})
   2024-09-30T22:51:23.98Z main.go:209: [ERROR] Error creating categories, err: %!w(*errors.Error=&{1 0xc0003a2e80 [0xc000                       39e2c0]})
   
   ```

   

4. 調整權限後再跑一次

   ![image-20241001095835526](https://kenkenny.synology.me:5543/images/2024/10/image-20241001095835526.png)

   ![image-20241001100827007](https://kenkenny.synology.me:5543/images/2024/10/image-20241001100827007.png)

   

   ![image-20241001102523546](https://kenkenny.synology.me:5543/images/2024/10/image-20241001102523546.png)

5. 新增View Prism Central、update VM basic config、Volume Group權限

   ![image-20241001102850872](https://kenkenny.synology.me:5543/images/2024/10/image-20241001102850872.png)

   ![image-20241001103337889](https://kenkenny.synology.me:5543/images/2024/10/image-20241001103337889.png)

   ![image-20241001104049556](https://kenkenny.synology.me:5543/images/2024/10/image-20241001104049556.png)

   

6. 完成

   ![image-20241001120931858](https://kenkenny.synology.me:5543/images/2024/10/image-20241001120931858.png)

7. 連線NKP Dashboard

   ```
   [root@ken-rhel9 ~]# nkp get dashboard --kubeconfig="/root/nkp-mgmt.conf"
   
   Username: crazy_boyd
   Password: HYokkgBhIU5WnjBfWoCfxwtBQ4WwP8APuw13fnDKA4MZ9eGz1Gbq3ddyiUkT4OiW
   URL: https://172.16.90.209/dkp/kommander/dashboard
   
   編輯cluster
   nkp --kubeconfig=/root/nkp-mgmt.conf edit cluster nkp-workload -n nkp-workload-9x6hx-2k79n
   ```

   ![image-20241001123425175](https://kenkenny.synology.me:5543/images/2024/10/image-20241001123425175.png)

8. 登入測試

   ![image-20241001123446785](https://kenkenny.synology.me:5543/images/2024/10/image-20241001123446785.png)

### NKP Starter

#### 畫面擷取

1. Dashboard

   可以新增Cluster

   ![image-20241001123601692](https://kenkenny.synology.me:5543/images/2024/10/image-20241001123601692.png)

2. Infrastructure Providers

   這裡不能修改或新增

   ![image-20241001123626080](https://kenkenny.synology.me:5543/images/2024/10/image-20241001123626080.png)

3. Identity Providers

   可以整合Github、LDAP、OIDC、SAML等

   ![image-20241001123843483](https://kenkenny.synology.me:5543/images/2024/10/image-20241001123843483.png)

   ![image-20241001123857034](https://kenkenny.synology.me:5543/images/2024/10/image-20241001123857034.png)

4. Access Control 預設有 Admin 跟 View Role
   Role Bindings是使用跟LDAP或是OIDC等等Group結合

   ![image-20241001124023266](https://kenkenny.synology.me:5543/images/2024/10/image-20241001124023266.png)

   ![image-20241001124114714](https://kenkenny.synology.me:5543/images/2024/10/image-20241001124114714.png)

5. Licensing，安裝在 Nutanix 上都有 NKP Starter ，但功能陽春 

   ![image-20241001124201488](https://kenkenny.synology.me:5543/images/2024/10/image-20241001124201488.png)

6. Resource Alerts 可以設定CPU、Memory、Disk的使用量閥值
   但在Starter看起來只會顯示在 UI 上面，UI 沒有其他可以設定寄送告警的地方
   這功能可以在 PC 上面設定閥值

   ![image-20241001124321203](https://kenkenny.synology.me:5543/images/2024/10/image-20241001124321203.png)

   PC Project畫面

   ![image-20241001124548715](https://kenkenny.synology.me:5543/images/2024/10/image-20241001124548715.png)

   ![image-20241001124614934](https://kenkenny.synology.me:5543/images/2024/10/image-20241001124614934.png)

7. 點進去Management Cluster

   沒有kubernetes Dashboard 只能看Namespaces有哪些，無法編輯

   ![image-20241001125501283](https://kenkenny.synology.me:5543/images/2024/10/image-20241001125501283.png)

   ![image-20241001125518563](https://kenkenny.synology.me:5543/images/2024/10/image-20241001125518563.png)

   ![image-20241001125605416](https://kenkenny.synology.me:5543/images/2024/10/image-20241001125605416.png)

#### Create Workload Cluster

用Control plane + worker node 各一台部署一個測試環境叢集

1. Create Cluster，可以選擇 UI或是 yaml 

   ![image-20241001125738098](https://kenkenny.synology.me:5543/images/2024/10/image-20241001125738098.png)

2. 輸入Cluster name : nkp-workload
   添加Labels 、其他無法選擇

   ![image-20241001125827762](https://kenkenny.synology.me:5543/images/2024/10/image-20241001125827762.png)

3. 數入 SSH Key、 PC Project、PE Subnets 資訊
   這裏PC Project 我有建立但沒抓到，不確定多久會抓一次資訊

   ![image-20241001130524925](https://kenkenny.synology.me:5543/images/2024/10/image-20241001130524925.png)

4. OS Image 、 Control Plan IP 及數量

   ![image-20241001130625362](https://kenkenny.synology.me:5543/images/2024/10/image-20241001130625362.png)

5. Worker Node 資訊、Storage

   ![image-20241001130739672](https://kenkenny.synology.me:5543/images/2024/10/image-20241001130739672.png)

   ![image-20241001130807311](https://kenkenny.synology.me:5543/images/2024/10/image-20241001130807311.png)

6. 容器網路

   ![image-20241001130857996](https://kenkenny.synology.me:5543/images/2024/10/image-20241001130857996.png)

7. Image Registry 先留空白

   ![image-20241001131101538](https://kenkenny.synology.me:5543/images/2024/10/image-20241001131101538.png)

8. 開始部署

   ![image-20241001131157253](https://kenkenny.synology.me:5543/images/2024/10/image-20241001131157253.png)

   ![image-20241001131220624](https://kenkenny.synology.me:5543/images/2024/10/image-20241001131220624.png)

9. 安裝完成

   ![image-20241001133509746](https://kenkenny.synology.me:5543/images/2024/10/image-20241001133509746.png)

   ![image-20241001133534110](https://kenkenny.synology.me:5543/images/2024/10/image-20241001133534110.png)

10. 整體畫面，UI 沒有升級的地方

    要從指令去操作，nkp 指令有提供 update, upgrade

    ![image-20241001133638170](https://kenkenny.synology.me:5543/images/2024/10/image-20241001133638170.png)

    ![image-20241001133953124](https://kenkenny.synology.me:5543/images/2024/10/image-20241001133953124.png)

    Management 整個叢集的pod數量

    ```
    
    [root@ken-rhel9 ~]# kubectl --kubeconfig=nkp-mgmt.conf get nodes
    NAME                              STATUS   ROLES           AGE    VERSION
    nkp-mgmt-hzxnp-28zsd              Ready    control-plane   127m   v1.29.6
    nkp-mgmt-hzxnp-cnjn6              Ready    control-plane   125m   v1.29.6
    nkp-mgmt-hzxnp-gg88v              Ready    control-plane   124m   v1.29.6
    nkp-mgmt-md-0-l965p-fczv8-mspnw   Ready    <none>          125m   v1.29.6
    nkp-mgmt-md-0-l965p-fczv8-n6tlc   Ready    <none>          126m   v1.29.6
    nkp-mgmt-md-0-l965p-fczv8-t5xmd   Ready    <none>          125m   v1.29.6
    nkp-mgmt-md-0-l965p-fczv8-w7sxk   Ready    <none>          125m   v1.29.6
    
    [root@ken-rhel9 ~]# kubectl --kubeconfig=nkp-mgmt.conf get pods -A
    NAMESPACE                           NAME                                                              READY   STATUS      RESTARTS       AGE
    caaph-system                        caaph-controller-manager-7557bfbd76-cdx7d                         1/1     Running     0              120m
    capa-system                         capa-controller-manager-549fff5698-9zdlh                          1/1     Running     0              120m
    capg-system                         capg-controller-manager-54df78c867-7zqlw                          1/1     Running     0              120m
    capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-589f87d995-gqjh6        1/1     Running     0              120m
    capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-869b46bd8d-tqmr5    1/1     Running     0              120m
    capi-system                         capi-controller-manager-7bd8c69994-4c5h9                          1/1     Running     0              120m
    cappp-system                        cappp-controller-manager-6cc595974c-dlj27                         1/1     Running     0              120m
    capv-system                         capv-controller-manager-7c6679579f-sj4vc                          1/1     Running     0              120m
    capvcd-system                       capvcd-controller-manager-647bbc5685-74qnv                        1/1     Running     0              120m
    capx-system                         capx-controller-manager-6b4c9976f6-f29bg                          1/1     Running     0              120m
    capz-system                         azureserviceoperator-controller-manager-697d757c4b-bpwmm          2/2     Running     0              120m
    capz-system                         capz-controller-manager-75c684766c-mp5bb                          1/1     Running     0              120m
    caren-system                        cluster-api-runtime-extensions-nutanix-869f6bc85d-clcgv           1/1     Running     0              120m
    caren-system                        helm-repository-86c695db8f-t7cgs                                  1/1     Running     0              120m
    cert-manager                        cert-manager-c49657b87-kvv2q                                      1/1     Running     0              122m
    cert-manager                        cert-manager-cainjector-7b9b545679-w867w                          1/1     Running     0              122m
    cert-manager                        cert-manager-webhook-647c6946df-qwvnk                             1/1     Running     0              122m
    default                             cluster-autoscaler-01924623-02df-76cb-a348-158120974c65-8ff7rp7   1/1     Running     0              126m
    git-operator-system                 git-operator-controller-manager-585cc87d78-bgl8w                  2/2     Running     0              115m
    git-operator-system                 git-operator-git-0                                                3/3     Running     0              115m
    kommander-default-workspace         karma-traefik-certs-kommander-default-workspace-cert-federj7xjn   1/1     Running     0              107m
    kommander-default-workspace         kubecost-traefik-certs-kommander-default-workspace-cert-fe25f2t   1/1     Running     0              107m
    kommander-default-workspace         prometheus-traefik-certs-kommander-default-workspace-cert-fxpv6   1/1     Running     0              107m
    kommander-flux                      helm-controller-dc4455cd5-frdrw                                   1/1     Running     0              116m
    kommander-flux                      kustomize-controller-755bbdfc55-kd8rh                             1/1     Running     0              116m
    kommander-flux                      notification-controller-86c44d47f9-nzvf7                          1/1     Running     0              116m
    kommander-flux                      source-controller-68bc9cbf4d-fn7x5                                1/1     Running     0              116m
    kommander                           cluster-observer-2360587938-5fc8b86cd6-9kpgn                      1/1     Running     0              106m
    kommander                           dex-5ff688ddc5-bs87d                                              1/1     Running     0              13m
    kommander                           dex-dex-controller-84656677b7-t7jtq                               2/2     Running     0              105m
    kommander                           dex-k8s-authenticator-cd94b9bfb-t6jq5                             1/1     Running     0              13m
    kommander                           gatekeeper-audit-57b899b497-msjzc                                 1/1     Running     0              113m
    kommander                           gatekeeper-controller-manager-84456b75f-5r8h4                     1/1     Running     0              113m
    kommander                           gatekeeper-controller-manager-84456b75f-nf66v                     1/1     Running     0              113m
    kommander                           karma-traefik-certs-kommander-cert-federation-7b7bbdf54f-jdcfw    1/1     Running     0              107m
    kommander                           kommander-appmanagement-677d49d6d4-bwgrw                          2/2     Running     0              111m
    kommander                           kommander-appmanagement-webhook-765c8c99f-cknr5                   1/1     Running     0              111m
    kommander                           kommander-authorizedlister-69586f8664-tl5wq                       1/1     Running     0              108m
    kommander                           kommander-bootstrap-kjrcc                                         0/1     Completed   0              126m
    kommander                           kommander-capimate-68db846987-d9br9                               1/1     Running     0              108m
    kommander                           kommander-capimate-68db846987-kp44s                               1/1     Running     0              107m
    kommander                           kommander-cm-6cdb956ddc-pbqcn                                     2/2     Running     0              108m
    kommander                           kommander-flux-operator-84976cdbd7-l6mkt                          2/2     Running     0              108m
    kommander                           kommander-kommander-ui-594c784fb9-fb5rp                           1/1     Running     0              107m
    kommander                           kommander-licensing-cm-79db86b6f9-hfh9b                           2/2     Running     0              108m
    kommander                           kommander-licensing-webhook-8f48d69f4-n8n2t                       1/1     Running     0              108m
    kommander                           kommander-operator-5ddcf7797f-9mw9k                               1/1     Running     0              113m
    kommander                           kommander-reloader-reloader-68fb77b665-nw854                      1/1     Running     0              109m
    kommander                           kommander-traefik-7bbc46f4f5-6bvxs                                1/1     Running     0              109m
    kommander                           kommander-traefik-7bbc46f4f5-7nxt8                                1/1     Running     0              109m
    kommander                           kommander-webhook-7b758645bd-xkk57                                1/1     Running     0              108m
    kommander                           kube-oidc-proxy-7b8ccdf676-svvqs                                  1/1     Running     0              105m
    kommander                           kubecost-traefik-certs-kommander-cert-federation-c455687b-q8dzq   1/1     Running     0              107m
    kommander                           prometheus-traefik-certs-kommander-cert-federation-677d4b4dlnmz   1/1     Running     0              107m
    kommander                           runtime-extension-kommander-59d9c676b7-vdk76                      1/1     Running     0              120m
    kommander                           traefik-forward-auth-mgmt-7b4848595b-z4kpq                        1/1     Running     0              100m
    kube-federation-system              kubefed-admission-webhook-575c45986d-9c4ff                        1/1     Running     0              109m
    kube-federation-system              kubefed-controller-manager-c6d6989b8-77bwd                        1/1     Running     0              108m
    kube-federation-system              kubefed-controller-manager-c6d6989b8-7m48r                        1/1     Running     0              108m
    kube-system                         cilium-2s6jn                                                      1/1     Running     0              126m
    kube-system                         cilium-5rssh                                                      1/1     Running     0              124m
    kube-system                         cilium-mswcf                                                      1/1     Running     0              124m
    kube-system                         cilium-n7h7f                                                      1/1     Running     0              124m
    kube-system                         cilium-operator-58c5d5d6d-sxxp8                                   1/1     Running     0              126m
    kube-system                         cilium-operator-58c5d5d6d-vknvd                                   1/1     Running     0              126m
    kube-system                         cilium-p89dz                                                      1/1     Running     0              123m
    kube-system                         cilium-ph4px                                                      1/1     Running     0              125m
    kube-system                         cilium-prc67                                                      1/1     Running     0              124m
    kube-system                         coredns-76f75df574-tnk7v                                          1/1     Running     0              126m
    kube-system                         coredns-76f75df574-w84hk                                          1/1     Running     0              126m
    kube-system                         etcd-nkp-mgmt-hzxnp-28zsd                                         1/1     Running     0              126m
    kube-system                         etcd-nkp-mgmt-hzxnp-cnjn6                                         1/1     Running     0              124m
    kube-system                         etcd-nkp-mgmt-hzxnp-gg88v                                         1/1     Running     0              123m
    kube-system                         kube-apiserver-nkp-mgmt-hzxnp-28zsd                               1/1     Running     0              126m
    kube-system                         kube-apiserver-nkp-mgmt-hzxnp-cnjn6                               1/1     Running     0              124m
    kube-system                         kube-apiserver-nkp-mgmt-hzxnp-gg88v                               1/1     Running     0              123m
    kube-system                         kube-controller-manager-nkp-mgmt-hzxnp-28zsd                      1/1     Running     0              126m
    kube-system                         kube-controller-manager-nkp-mgmt-hzxnp-cnjn6                      1/1     Running     0              124m
    kube-system                         kube-controller-manager-nkp-mgmt-hzxnp-gg88v                      1/1     Running     0              123m
    kube-system                         kube-proxy-76jcm                                                  1/1     Running     0              124m
    kube-system                         kube-proxy-96pjt                                                  1/1     Running     0              124m
    kube-system                         kube-proxy-9qrlm                                                  1/1     Running     0              123m
    kube-system                         kube-proxy-cxw6r                                                  1/1     Running     0              125m
    kube-system                         kube-proxy-jkl95                                                  1/1     Running     0              124m
    kube-system                         kube-proxy-kr5wg                                                  1/1     Running     0              126m
    kube-system                         kube-proxy-ljlhh                                                  1/1     Running     0              124m
    kube-system                         kube-scheduler-nkp-mgmt-hzxnp-28zsd                               1/1     Running     0              126m
    kube-system                         kube-scheduler-nkp-mgmt-hzxnp-cnjn6                               1/1     Running     0              124m
    kube-system                         kube-scheduler-nkp-mgmt-hzxnp-gg88v                               1/1     Running     0              123m
    kube-system                         kube-vip-nkp-mgmt-hzxnp-28zsd                                     1/1     Running     1 (126m ago)   126m
    kube-system                         kube-vip-nkp-mgmt-hzxnp-cnjn6                                     1/1     Running     0              124m
    kube-system                         kube-vip-nkp-mgmt-hzxnp-gg88v                                     1/1     Running     0              123m
    kube-system                         nutanix-cloud-controller-manager-598b4c7669-4q8j8                 1/1     Running     0              126m
    kube-system                         snapshot-controller-5c7f9fc58-7zjjg                               1/1     Running     0              126m
    metallb-system                      metallb-controller-94f95d674-48z4s                                1/1     Running     0              126m
    metallb-system                      metallb-speaker-6lclp                                             4/4     Running     0              122m
    metallb-system                      metallb-speaker-c78xt                                             4/4     Running     0              123m
    metallb-system                      metallb-speaker-cnljr                                             4/4     Running     0              123m
    metallb-system                      metallb-speaker-dklnr                                             4/4     Running     0              124m
    metallb-system                      metallb-speaker-fmbzv                                             4/4     Running     0              123m
    metallb-system                      metallb-speaker-nwq8b                                             4/4     Running     0              125m
    metallb-system                      metallb-speaker-nxhz6                                             4/4     Running     0              124m
    nkp-workload-9x6hx-2k79n            cluster-autoscaler-0192467d-9888-7536-8266-ed9e557d46a1-66qkh7z   1/1     Running     1 (15m ago)    27m
    nkp-workload-9x6hx-2k79n            cluster-observer-1774060518-68b9989d54-dzcqt                      1/1     Running     0              27m
    nkp-workload-9x6hx-2k79n            karma-traefik-certs-nkp-workload-9x6hx-2k79n-cert-federatish7kf   1/1     Running     0              28m
    nkp-workload-9x6hx-2k79n            kubecost-traefik-certs-nkp-workload-9x6hx-2k79n-cert-federn6pf9   1/1     Running     0              28m
    nkp-workload-9x6hx-2k79n            prometheus-traefik-certs-nkp-workload-9x6hx-2k79n-cert-fedh9vpc   1/1     Running     0              29m
    node-feature-discovery              node-feature-discovery-gc-7f54d58d99-zvgnb                        1/1     Running     0              126m
    node-feature-discovery              node-feature-discovery-master-ccf75997b-dp7q4                     1/1     Running     0              126m
    node-feature-discovery              node-feature-discovery-worker-42bxq                               1/1     Running     0              124m
    node-feature-discovery              node-feature-discovery-worker-478dl                               1/1     Running     0              124m
    node-feature-discovery              node-feature-discovery-worker-79lmw                               1/1     Running     0              123m
    node-feature-discovery              node-feature-discovery-worker-7kvw6                               1/1     Running     0              123m
    node-feature-discovery              node-feature-discovery-worker-7tvvn                               1/1     Running     0              122m
    node-feature-discovery              node-feature-discovery-worker-dlj8p                               1/1     Running     0              123m
    node-feature-discovery              node-feature-discovery-worker-hpbmp                               1/1     Running     0              125m
    ntnx-system                         nutanix-csi-controller-754fcf5f85-c9vtt                           7/7     Running     3 (119m ago)   122m
    ntnx-system                         nutanix-csi-controller-754fcf5f85-sfrpb                           7/7     Running     3 (119m ago)   122m
    ntnx-system                         nutanix-csi-node-5d9gs                                            3/3     Running     0              122m
    ntnx-system                         nutanix-csi-node-8qhkx                                            3/3     Running     1 (121m ago)   122m
    ntnx-system                         nutanix-csi-node-dbxm8                                            3/3     Running     1 (121m ago)   122m
    ntnx-system                         nutanix-csi-node-qvkv9                                            3/3     Running     0              122m
    ntnx-system                         nutanix-csi-node-s2wn7                                            3/3     Running     1 (122m ago)   122m
    ntnx-system                         nutanix-csi-node-tq5bt                                            3/3     Running     1 (120m ago)   122m
    ntnx-system                         nutanix-csi-node-vz87g                                            3/3     Running     0              122m
    
    ```

    其他NKP , kubectl 指令、加金鑰

    ```
    [root@ken-rhel9 ~]# nkp get workspace --kubeconfig="/root/nkp-mgmt.conf"
    NAME                    NAMESPACE
    default-workspace       kommander-default-workspace
    kommander-workspace     kommander
    nkp-workload-9x6hx      nkp-workload-9x6hx-2k79n
    
    [root@ken-rhel9 ~]# nkp get cluster -w kommander-workspace  --kubeconfig="/root/nkp-mgmt.conf"
    WORKSPACE               NAME            KUBECONFIG                              STATUS
    kommander-workspace     host-cluster    kommander-self-attach-kubeconfig        Joined
    
    [root@ken-rhel9 ~]# nkp get cluster -w default-workspace  --kubeconfig="/root/nkp-mgmt.conf"
    WORKSPACE       NAME    KUBECONFIG      STATUS
    
    [root@ken-rhel9 ~]# nkp get cluster -w nkp-workload-9x6hx  --kubeconfig="/root/nkp-mgmt.conf"
    WORKSPACE               NAME            KUBECONFIG              STATUS
    nkp-workload-9x6hx      nkp-workload    nkp-workload-kubeconfig Joined
    
    編輯cluster
    nkp --kubeconfig=/root/nkp-mgmt.conf edit cluster nkp-workload -n nkp-workload-9x6hx-2k79n
    
    [root@ken-rhel9 ~]# nkp get cluster -w nkp-workload-lab  --kubeconfig="/root/nkp-mgmt.conf"
    WORKSPACE       	NAME         	KUBECONFIG              	STATUS 
    nkp-workload-lab	nkp-cluster01	nkp-cluster01-kubeconfig	Joined	
    
    [root@ken-rhel9 ~]# nkp describe cluster -w nkp-workload-lab  --kubeconfig="/root/nkp-mgmt.conf"
    
    [root@ken-rhel9 ~]# nkp edit cluster nkp-cluster01 -n nkp-workload-lab  --kubeconfig="/root/nkp-mgmt.conf"
    
    
            Users:
              Name:  nkp
              Ssh Authorized Keys:
                ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCVN+q1hhDCFrbW4Gw+wPryGl2BQbLi9NNOv+hXLgI+NXJqhTT0XEL6Uun5WKd7ceNznfiCHee2AkQXm9nrvVyPpFU5zQ91j9zQbMsFylSDv7RALFyEaXO45u6bzKRvVnL4lsKygBcMwxSW8yIrv0qTeDl3rChfkxjhpWR+C7gbn6uIVINRoC/5L1xNGcThbF0CiHgOt5P03ZV4tsm6C7Z65Wj41vczaH460qqq//kaf2uNveOZhi4juIQMc0FcE4kvGwtqRJ5HIs8CRq1N/wX+uIIQEKsqtmQqSujcHOwosYPSNvVXvVgqvy9t0jfEU4Y3bcBkoF5zfq98OCG0978ewjb8yR909bQM2s2QRoQML8Gb6Be+EV7+xtUcqZOpAkyZOmWXN8yoL5EMcelgDZoyK3yH9FgKuz1sbB0f+4b1B8LM/SULTq8Lv7ww6ps3EIvKDwbH5rMQIxf3IEIuk7qvjyQlAZiE2nxiU88fcPB54WUoQBNd3uUjr0YVt1i5GM0=
    
              Sudo:  ALL=(ALL) NOPASSWD:ALL
        Version:     v1.29.6
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
                        Name:  NX1365G6PE
                        Type:  name
                      Image:
                        Name:       nkp-rocky-9.4-release-1.29.6-20240816215147.qcow2
                        Type:       name
                      Memory Size:  32Gi
                      Project:
                        Name:  nkp-management
                        Type:  name
                      Subnets:
                        Name:            Netfos_90_Access_IPAM
                        Type:            name
                      System Disk Size:  80Gi
                      Vcpu Sockets:      8
                      Vcpus Per Socket:  1
    
    [root@ken-rhel9 ~]# cat /home/nkp/.ssh/id_rsa.pub 
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCVN+q1hhDCFrbW4Gw+wPryGl2BQbLi9NNOv+hXLgI+NXJqhTT0XEL6Uun5WKd7ceNznfiCHee2AkQXm9nrvVyPpFU5zQ91j9zQbMsFylSDv7RALFyEaXO45u6bzKRvVnL4lsKygBcMwxSW8yIrv0qTeDl3rChfkxjhpWR+C7gbn6uIVINRoC/5L1xNGcThbF0CiHgOt5P03ZV4tsm6C7Z65Wj41vczaH460qqq//kaf2uNveOZhi4juIQMc0FcE4kvGwtqRJ5HIs8CRq1N/wX+uIIQEKsqtmQqSujcHOwosYPSNvVXvVgqvy9t0jfEU4Y3bcBkoF5zfq98OCG0978ewjb8yR909bQM2s2QRoQML8Gb6Be+EV7+xtUcqZOpAkyZOmWXN8yoL5EMcelgDZoyK3yH9FgKuz1sbB0f+4b1B8LM/SULTq8Lv7ww6ps3EIvKDwbH5rMQIxf3IEIuk7qvjyQlAZiE2nxiU88fcPB54WUoQBNd3uUjr0YVt1i5GM0= nkp@ken-rhel9.nutanixlab.local
    
    
    
    
    
    [root@ken-rhel9 ~]# kubectl --kubeconfig=nkp-mgmt.conf describe cluster nkp-mgmt
    Name:         nkp-mgmt
    Namespace:    default
    Labels:       cluster.x-k8s.io/cluster-name=nkp-mgmt
                  cluster.x-k8s.io/provider=nutanix
                  konvoy.d2iq.io/cluster-name=nkp-mgmt
                  konvoy.d2iq.io/provider=nutanix
                  topology.cluster.x-k8s.io/owned=
    Annotations:  caren.nutanix.com/cluster-uuid: 01924623-02df-76cb-a348-158120974c65
    API Version:  cluster.x-k8s.io/v1beta1
    Kind:         Cluster
    Metadata:
      Creation Timestamp:  2024-10-01T03:44:26Z
      Finalizers:
        cluster.cluster.x-k8s.io
      Generation:        2
      Resource Version:  9910
      UID:               b1e6c594-7132-40cd-ae06-80623ce81de0
    Spec:
      Cluster Network:
        Pods:
          Cidr Blocks:
            192.168.0.0/16
        Services:
          Cidr Blocks:
            10.96.0.0/12
      Control Plane Endpoint:
        Host:  172.16.90.208
        Port:  6443
      Control Plane Ref:
        API Version:  controlplane.cluster.x-k8s.io/v1beta1
        Kind:         KubeadmControlPlane
        Name:         nkp-mgmt-hzxnp
        Namespace:    default
      Infrastructure Ref:
        API Version:  infrastructure.cluster.x-k8s.io/v1beta1
        Kind:         NutanixCluster
        Name:         nkp-mgmt-lbhrg
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
                    Name:  nkp-mgmt-pc-credentials
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
                        Name:  nkp-mgmt-pc-credentials-for-csi
                    Storage Class Configs:
                      Volume:
                        Allow Expansion:  false
                        Parameters:
                          csi.storage.k8s.io/fstype:  ext4
                          Description:                CSI StorageClass nutanix-volume for nkp-mgmt
                          Flash Mode:                 DISABLED
                          Hypervisor Attached:        ENABLED
                          Storage Container:          NX_1365_AHV_cr
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
                    End:    172.16.90.209
                    Start:  172.16.90.209
                Provider:   MetalLB
            Control Plane:
              Nutanix:
                Machine Details:
                  Boot Type:  uefi
                  Cluster:
                    Name:  NX1365G6PE
                    Type:  name
                  Image:
                    Name:       nkp-rocky-9.4-release-1.29.6-20240816215147.qcow2
                    Type:       name
                  Memory Size:  16Gi
                  Project:
                    Name:  nkp-management
                    Type:  name
                  Subnets:
                    Name:            Netfos_90_Access_IPAM
                    Type:            name
                  System Disk Size:  80Gi
                  Vcpu Sockets:      4
                  Vcpus Per Socket:  1
            Encryption At Rest:
              Providers:
                Secretbox:
            Image Registries:
              Credentials:
                Secret Ref:
                  Name:  nkp-mgmt-image-registry-credentials
              URL:       https://registry-1.docker.io
            Nutanix:
              Control Plane Endpoint:
                Host:  172.16.90.208
                Port:  6443
                Virtual IP:
                  Provider:  KubeVIP
              Prism Central Endpoint:
                Credentials:
                  Secret Ref:
                    Name:  nkp-mgmt-pc-credentials
                Insecure:  true
                URL:       https://172.16.90.75:9440
            Users:
              Name:  nkp
              Ssh Authorized Keys:
                ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCVN+q1hhDCFrbW4Gw+wPryGl2BQbLi9NNOv+hXLgI+NXJqhTT0XEL6Uun5WKd7ceNznfiCHee2AkQXm9nrvVyPpFU5zQ 91j9zQbMsFylSDv7RALFyEaXO45u6bzKRvVnL4lsKygBcMwxSW8yIrv0qTeDl3rChfkxjhpWR+C7gbn6uIVINRoC/5L1xNGcThbF0CiHgOt5P03ZV4tsm6C7Z65Wj41vczaH460qqq//ka f2uNveOZhi4juIQMc0FcE4kvGwtqRJ5HIs8CRq1N/wX+uIIQEKsqtmQqSujcHOwosYPSNvVXvVgqvy9t0jfEU4Y3bcBkoF5zfq98OCG0978ewjb8yR909bQM2s2QRoQML8Gb6Be+EV7+xt UcqZOpAkyZOmWXN8yoL5EMcelgDZoyK3yH9FgKuz1sbB0f+4b1B8LM/SULTq8Lv7ww6ps3EIvKDwbH5rMQIxf3IEIuk7qvjyQlAZiE2nxiU88fcPB54WUoQBNd3uUjr0YVt1i5GM0=
    
              Sudo:  ALL=(ALL) NOPASSWD:ALL
        Version:     v1.29.6
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
                        Name:  NX1365G6PE
                        Type:  name
                      Image:
                        Name:       nkp-rocky-9.4-release-1.29.6-20240816215147.qcow2
                        Type:       name
                      Memory Size:  32Gi
                      Project:
                        Name:  nkp-management
                        Type:  name
                      Subnets:
                        Name:            Netfos_90_Access_IPAM
                        Type:            name
                      System Disk Size:  80Gi
                      Vcpu Sockets:      8
                      Vcpus Per Socket:  1
    Status:
      Conditions:
        Last Transition Time:  2024-10-01T03:44:45Z
        Status:                True
        Type:                  Ready
        Last Transition Time:  2024-10-01T03:44:39Z
        Status:                True
        Type:                  ControlPlaneInitialized
        Last Transition Time:  2024-10-01T03:44:45Z
        Status:                True
        Type:                  ControlPlaneReady
        Last Transition Time:  2024-10-01T03:44:42Z
        Status:                True
        Type:                  InfrastructureReady
        Last Transition Time:  2024-10-01T03:46:51Z
        Status:                True
        Type:                  KommanderInitialized
        Last Transition Time:  2024-10-01T03:44:41Z
        Status:                True
        Type:                  TopologyReconciled
      Control Plane Ready:     true
      Infrastructure Ready:    true
      Observed Generation:     2
      Phase:                   Provisioned
    Events:                    <none>
    
    [root@ken-rhel9 ~]# kubectl --kubeconfig=nkp-mgmt.conf describe cluster nkp-mgmt |grep UID
      UID:               b1e6c594-7132-40cd-ae06-80623ce81de0
    ```
    
    

### License

1. 在Support Portal 可以看到有購買的License

   ![image-20241001135632620](https://kenkenny.synology.me:5543/images/2024/10/image-20241001135632620.png)

2. 點選Manage Licenses ，選擇 Nutanix Kubernetes Plaform

   ![image-20241001135718674](https://kenkenny.synology.me:5543/images/2024/10/image-20241001135718674.png)

3. 輸入NKP 叢集名稱、UUID (optional)

   ![image-20241001143111547](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143111547.png)

4. 選擇License NKP Ultimate

   ![image-20241001143154561](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143154561.png)

5. 選擇License的種類，這裡是POC License

   ![image-20241001143221418](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143221418.png)

6. 選擇叢集

   ![image-20241001143255790](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143255790.png)

7. 選擇 vCPU 數量

   ![image-20241001143448697](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143448697.png)

8. 點選Save 然後 Next

   ![image-20241001143511224](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143511224.png)

9. 最後確認完成

   ![image-20241001143551177](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143551177.png)

10. 點選Generate license keys

    ![image-20241001143623309](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143623309.png)

11. 即可產生 License key

    ```
    AEAAQ-AAAQS-L5N22-FU6P2-6NGKV-9QKLU-UF8S7
    renew 
    AEAAQ-AAASU-57YDC-4PVGR-EJ233-N5AKR-3T5XG
    ```

    ![image-20241001143657849](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143657849.png)

    renew

    ![image-20241125174055439](https://kenkenny.synology.me:5543/images/2024/11/image-20241125174055439.png)

12. Support Portal 即可看到使用的License

    ![image-20241001143823066](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143823066.png)

13. 回到NKP Dashboard 的 Licensing 頁面，點選移除License

    ![image-20241001143936967](https://kenkenny.synology.me:5543/images/2024/10/image-20241001143936967.png)

    ![image-20241001144009276](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144009276.png)

    renew

    ![image-20241125174303231](https://kenkenny.synology.me:5543/images/2024/11/image-20241125174303231.png)

14. 未授權狀態，可以看到的畫面與NKP Starter 一樣

    ![image-20241001144031075](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144031075.png)

    renew ，機器人的button會被拿掉，但其他正常

    ![image-20241125174337757](https://kenkenny.synology.me:5543/images/2024/11/image-20241125174337757.png)

15. 點選 Activate License，輸入 License key

    ![image-20241001144210411](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144210411.png)

    renew

    ![image-20241125174644264](https://kenkenny.synology.me:5543/images/2024/11/image-20241125174644264.png)

16. 授權匯入完成

    ![image-20241001144234995](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144234995.png)

    renew 完成，機器人就正常了

    ![image-20241125174657723](https://kenkenny.synology.me:5543/images/2024/11/image-20241125174657723.png)

    ![image-20241125174855765](https://kenkenny.synology.me:5543/images/2024/11/image-20241125174855765.png)

17. License 到期前告警

    ![image-20241121141917439](https://kenkenny.synology.me:5543/images/2024/11/image-20241121141917439.png)

18. Licesne 到期後影響：AI機器人無法使用，其他應用程式看起來正常，按鈕沒被拔掉，Dashboard 也正常

    ![image-20241125092518667](https://kenkenny.synology.me:5543/images/2024/11/image-20241125092518667.png)

19. AI 機器人壞掉，服務應該跟License無關，但修復後有跳出License異常，所以到期後應該是無法使用AI機器人

    ![image-20241125092017318](https://kenkenny.synology.me:5543/images/2024/11/image-20241125092017318.png)

    ![image-20241125092320825](https://kenkenny.synology.me:5543/images/2024/11/image-20241125092320825.png)

    ![image-20241125092746272](https://kenkenny.synology.me:5543/images/2024/11/image-20241125092746272.png)

20. 修復動作

    ```
    # 先將scale 調整成0
    [nkp@ken-rhel9 ~]$ kubectl scale deployment ai-navigator-cluster-info-api -n kommander --replicas=0
    
    # 刪除失敗的pods 用Label來表示所有相關的pods
    [nkp@ken-rhel9 ~]$ kubectl delete pod -l app.kubernetes.io/name=ai-navigator-cluster-info-api -n kommander
    pod "ai-navigator-cluster-info-api-6b78d6449-226cv" deleted
    pod "ai-navigator-cluster-info-api-6b78d6449-249pv" deleted
    pod "ai-navigator-cluster-info-api-6b78d6449-2564c" deleted
    pod "ai-navigator-cluster-info-api-6b78d6449-267s7" deleted
    ...
    最後會卡著要用 Control+C 退出
    
    # 刪除完成
    [nkp@ken-rhel9 ~]$ kubectl get pods -n kommander
    NAME                                                              READY   STATUS      RESTARTS      AGE
    ai-navigator-app-554f86cd46-797tq                                 1/1     Running     0             54d
    ai-navigator-cluster-info-agent-76bfb65f49-qdrqx                  1/1     Running     0             38d
    ai-navigator-cluster-info-api-postgresql-0                        1/1     Running     0             54d
    alertmanager-kube-prometheus-stack-alertmanager-0                 2/2     Running     0             54d
    centralized-grafana-7d7b874996-w658s                              2/2     Running     0             54d
    cluster-observer-2360587938-5fc8b86cd6-9kpgn                      1/1     Running     0             54d
    
    
    # 刪除完將scale 調整回5
    [nkp@ken-rhel9 ~]$ kubectl scale deployment ai-navigator-cluster-info-api -n kommander --replicas=5
    
    [nkp@ken-rhel9 ~]$ kubectl get pods -n kommander
    NAME                                                              READY   STATUS      RESTARTS      AGE
    ai-navigator-app-554f86cd46-797tq                                 1/1     Running     0             54d
    ai-navigator-cluster-info-agent-76bfb65f49-qdrqx                  1/1     Running     0             38d
    ai-navigator-cluster-info-api-557c646c6b-5tkfj                    0/1     Init:0/1    0             4s
    ai-navigator-cluster-info-api-557c646c6b-7sdmk                    0/1     Init:0/1    0             4s
    ai-navigator-cluster-info-api-557c646c6b-htnr7                    0/1     Init:0/1    0             4s
    ai-navigator-cluster-info-api-557c646c6b-wbn4g                    0/1     Init:0/1    0             4s
    ai-navigator-cluster-info-api-557c646c6b-xkb8d                    0/1     Init:0/1    0             4s
    
    # 還是不行，修正錯誤，describe po 之後發現SizeLimit 超用，並且image有更新版本
    [nkp@ken-rhel9 ~]$ kubectl describe pod  ai-navigator-cluster-info-api-6c65f767b8-f5cgl  -n kommander
    
    [nkp@ken-rhel9 ~]$ kubectl edit deployment ai-navigator-cluster-info-api -n kommander
    deployment.apps/ai-navigator-cluster-info-api edited
    image 改為 image: mesosphere/ai-navigator-cluster-info-api:v0.1.0-model-revision-fix
    Volumes SizeLimit改為 8Gi
          - emptyDir:
              sizeLimit: 8Gi
    
    # 修正ＯＫ
    [nkp@ken-rhel9 ~]$ kubectl get pods -n kommander
    NAME                                                              READY   STATUS      RESTARTS      AGE
    ai-navigator-app-554f86cd46-797tq                                 1/1     Running     0             54d
    ai-navigator-cluster-info-agent-76bfb65f49-qdrqx                  1/1     Running     0             38d
    ai-navigator-cluster-info-api-6c65f767b8-f5cgl                    1/1     Running     0             3m4s
    ai-navigator-cluster-info-api-6c65f767b8-jpn96                    1/1     Running     0             3m4s
    ai-navigator-cluster-info-api-6c65f767b8-tx69k                    1/1     Running     0             3m4s
    ai-navigator-cluster-info-api-6c65f767b8-vm926                    0/1     Running     0             3m4s
    ai-navigator-cluster-info-api-6c65f767b8-xwsbx  
    
    
    [nkp@ken-rhel9 ~]$ kubectl logs ai-navigator-cluster-info-api-6c65f767b8-f5cgl  -n kommander
    .
    2024-11-25 02:37:24,745 urllib3.connectionpool DEBUG http://weaviate:80 "GET /v1/meta HTTP/1.1" 200 64
    2024-11-25 02:37:24,752 urllib3.connectionpool DEBUG Starting new HTTPS connection (1): pypi.org:443
    2024-11-25 02:37:24,946 urllib3.connectionpool DEBUG https://pypi.org:443 "GET /pypi/weaviate-client/json HTTP/1.1" 200 61151
    /app/.venv/lib/python3.11/site-packages/weaviate/warnings.py:121: DeprecationWarning: Dep005: You are using weaviate-client version 3.25.3. The latest version is 4.9.4.
                Please consider upgrading to the latest version. See https://weaviate.io/developers/weaviate/client-libraries/python for details.
      warnings.warn(
    INFO:     Started server process [1]
    INFO:     Waiting for application startup.
    INFO:     Application startup complete.
    INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
    
    
    kubectl rollout restart deployment ai-navigator-cluster-info-agent  -n kommander
    kubectl rollout restart deployment ai-navigator-app  -n kommander
    kubectl rollout restart deployment ai-navigator-cluster-info-api  -n kommander
    
    ```

    

21. AI 服務正常，但有跳出License Error (有使用就會多一筆)

      kubectl logs ai-navigator-app-5945846b57-5c6gl -n kommander |grep licesne

    <img src="https://kenkenny.synology.me:5543/images/2024/11/image-20241125124904258.png" alt="image-20241125124904258" style="zoom:50%;" />

    ![image-20241125114846644](https://kenkenny.synology.me:5543/images/2024/11/image-20241125114846644.png)

### NKP Ultimate

#### 畫面擷取

1. Dashboard

   ![image-20241001144342650](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144342650.png)

   ![image-20241001144415725](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144415725.png)

2. Clusters

   ![image-20241001144438786](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144438786.png)

   ![image-20241001144455447](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144455447.png)

3. Workspaces

   ![image-20241001144513817](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144513817.png)

4. Infrastructure Providers 可以新增其他的 （原先的會被移除，要重新加入）

   ![image-20241001144642139](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144642139.png)

   ![image-20241001144654156](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144654156.png)

5. Access Control 多了一些預設角色，也可以新增其他的角色
   Identity Providers 與 NKP Starter 相同

   ![image-20241001144807105](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144807105.png)

6. Banners

   ![image-20241001144848455](https://kenkenny.synology.me:5543/images/2024/10/image-20241001144848455.png)

7. 點進去Management Cluster workspaces

   ![image-20241001145016633](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145016633.png)

   ![image-20241001145034200](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145034200.png)

8. 各元件也都有Dashboard可查看

   ![image-20241001145117659](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145117659.png)

9. Kubernetes Dashboard 除了看到比較詳細的畫面外，也可以直接從 UI 部署新資源

   ![image-20241001145203867](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145203867.png)

   ![image-20241001145233001](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145233001.png)

10. Grafana

    ![image-20241001145336112](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145336112.png)

11. Traefik

    ![image-20241001145408776](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145408776.png)

12. Prometheus ，這邊就可以設定告警值等等

    ![image-20241001145457519](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145457519.png)

    ![image-20241001145510044](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145510044.png)

13. AI 測試

    ![image-20241001145946920](https://kenkenny.synology.me:5543/images/2024/10/image-20241001145946920.png)

    ![image-20241001150004325](https://kenkenny.synology.me:5543/images/2024/10/image-20241001150004325.png)

14. Application

    ![image-20241001150104478](https://kenkenny.synology.me:5543/images/2024/10/image-20241001150104478.png)

    ![image-20241001150119975](https://kenkenny.synology.me:5543/images/2024/10/image-20241001150119975.png)

    ![image-20241001150135142](https://kenkenny.synology.me:5543/images/2024/10/image-20241001150135142.png)

    ![image-20241001150147622](https://kenkenny.synology.me:5543/images/2024/10/image-20241001150147622.png)

    ![image-20241001150201548](https://kenkenny.synology.me:5543/images/2024/10/image-20241001150201548.png)

15. kubecost

    ![image-20241001152432840](https://kenkenny.synology.me:5543/images/2024/10/image-20241001152432840.png)

16. Ceph

    ![image-20241001153157559](https://kenkenny.synology.me:5543/images/2024/10/image-20241001153157559.png)

17. 所有Pod 資訊 ，相較於 NKP Starter  多了很多東西 AI 也是匯入授權後自動建立 

```
[root@ken-rhel9 ~]# kubectl --kubeconfig=nkp-mgmt.conf get pods -A
NAMESPACE                           NAME                                                              READY   STATUS              RESTARTS        AGE
caaph-system                        caaph-controller-manager-7557bfbd76-cdx7d                         1/1     Running             0               3h15m
capa-system                         capa-controller-manager-549fff5698-9zdlh                          1/1     Running             0               3h15m
capg-system                         capg-controller-manager-54df78c867-7zqlw                          1/1     Running             0               3h15m
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-589f87d995-gqjh6        1/1     Running             0               3h15m
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-869b46bd8d-tqmr5    1/1     Running             0               3h15m
capi-system                         capi-controller-manager-7bd8c69994-4c5h9                          1/1     Running             0               3h15m
cappp-system                        cappp-controller-manager-6cc595974c-dlj27                         1/1     Running             0               3h15m
capv-system                         capv-controller-manager-7c6679579f-sj4vc                          1/1     Running             0               3h15m
capvcd-system                       capvcd-controller-manager-647bbc5685-74qnv                        1/1     Running             0               3h15m
capx-system                         capx-controller-manager-6b4c9976f6-f29bg                          1/1     Running             0               3h15m
capz-system                         azureserviceoperator-controller-manager-697d757c4b-bpwmm          2/2     Running             0               3h15m
capz-system                         capz-controller-manager-75c684766c-mp5bb                          1/1     Running             0               3h15m
caren-system                        cluster-api-runtime-extensions-nutanix-869f6bc85d-clcgv           1/1     Running             0               3h15m
caren-system                        helm-repository-86c695db8f-t7cgs                                  1/1     Running             0               3h15m
cert-manager                        cert-manager-c49657b87-kvv2q                                      1/1     Running             0               3h17m
cert-manager                        cert-manager-cainjector-7b9b545679-w867w                          1/1     Running             0               3h17m
cert-manager                        cert-manager-webhook-647c6946df-qwvnk                             1/1     Running             0               3h17m
default                             cluster-autoscaler-01924623-02df-76cb-a348-158120974c65-8ff7rp7   1/1     Running             0               3h21m
git-operator-system                 git-operator-controller-manager-585cc87d78-bgl8w                  2/2     Running             0               3h10m
git-operator-system                 git-operator-git-0                                                3/3     Running             0               3h10m
kommander-default-workspace         karma-traefik-certs-kommander-default-workspace-cert-federj7xjn   1/1     Running             0               3h2m
kommander-default-workspace         kubecost-traefik-certs-kommander-default-workspace-cert-fe25f2t   1/1     Running             0               3h2m
kommander-default-workspace         prometheus-traefik-certs-kommander-default-workspace-cert-fxpv6   1/1     Running             0               3h2m
kommander-flux                      helm-controller-dc4455cd5-frdrw                                   1/1     Running             0               3h11m
kommander-flux                      kustomize-controller-755bbdfc55-kd8rh                             1/1     Running             0               3h11m
kommander-flux                      notification-controller-86c44d47f9-nzvf7                          1/1     Running             0               3h11m
kommander-flux                      source-controller-68bc9cbf4d-fn7x5                                1/1     Running             0               3h11m
kommander                           ai-navigator-app-554f86cd46-797tq                                 1/1     Running             0               13m
kommander                           ai-navigator-cluster-info-api-6b78d6449-fh756                     1/1     Running             0               11m
kommander                           ai-navigator-cluster-info-api-6b78d6449-fn6pn                     1/1     Running             0               11m
kommander                           ai-navigator-cluster-info-api-postgresql-0                        1/1     Running             0               11m
kommander                           alertmanager-kube-prometheus-stack-alertmanager-0                 2/2     Running             0               9m
kommander                           centralized-grafana-7d7b874996-w658s                              2/2     Running             0               9m30s
kommander                           cluster-observer-2360587938-5fc8b86cd6-9kpgn                      1/1     Running             0               3h1m
kommander                           create-kommander-thanos-query-stores-configmap-mblkt              0/1     Completed           0               11m
kommander                           dex-5ff688ddc5-bs87d                                              1/1     Running             0               88m
kommander                           dex-dex-controller-84656677b7-t7jtq                               2/2     Running             0               3h
kommander                           dex-k8s-authenticator-cd94b9bfb-t6jq5                             1/1     Running             0               88m
kommander                           dkp-ceph-prereq-job-f7wqt                                         0/2     Completed           0               11m
kommander                           etcd-metrics-proxy-4hm42                                          1/1     Running             0               12m
kommander                           etcd-metrics-proxy-4p8r8                                          1/1     Running             0               12m
kommander                           etcd-metrics-proxy-mxnlz                                          1/1     Running             0               12m
kommander                           gatekeeper-audit-57b899b497-msjzc                                 1/1     Running             0               3h8m
kommander                           gatekeeper-controller-manager-84456b75f-5r8h4                     1/1     Running             0               3h8m
kommander                           gatekeeper-controller-manager-84456b75f-nf66v                     1/1     Running             0               3h8m
kommander                           grafana-logging-7b655598b9-99gqn                                  2/2     Running             0               12m
kommander                           grafana-loki-pre-install-x56fg                                    1/1     Running             0               12m
kommander                           karma-5f8cf7cb76-hllxr                                            1/1     Running             0               11m
kommander                           karma-traefik-certs-kommander-cert-federation-7b7bbdf54f-jdcfw    1/1     Running             0               3h2m
kommander                           kommander-appmanagement-677d49d6d4-bwgrw                          2/2     Running             0               3h6m
kommander                           kommander-appmanagement-webhook-765c8c99f-cknr5                   1/1     Running             0               3h6m
kommander                           kommander-authorizedlister-69586f8664-tl5wq                       1/1     Running             0               3h3m
kommander                           kommander-bootstrap-kjrcc                                         0/1     Completed           0               3h21m
kommander                           kommander-capimate-68db846987-d9br9                               1/1     Running             0               3h3m
kommander                           kommander-capimate-68db846987-kp44s                               1/1     Running             0               3h3m
kommander                           kommander-cm-6cdb956ddc-pbqcn                                     2/2     Running             0               3h3m
kommander                           kommander-flux-operator-84976cdbd7-l6mkt                          2/2     Running             0               3h3m
kommander                           kommander-kommander-ui-594c784fb9-fb5rp                           1/1     Running             0               3h2m
kommander                           kommander-licensing-cm-79db86b6f9-hfh9b                           2/2     Running             0               3h3m
kommander                           kommander-licensing-webhook-8f48d69f4-n8n2t                       1/1     Running             0               3h3m
kommander                           kommander-operator-5ddcf7797f-9mw9k                               1/1     Running             0               3h9m
kommander                           kommander-reloader-reloader-68fb77b665-nw854                      1/1     Running             0               3h4m
kommander                           kommander-traefik-7bbc46f4f5-6bvxs                                1/1     Running             0               3h4m
kommander                           kommander-traefik-7bbc46f4f5-7nxt8                                1/1     Running             0               3h4m
kommander                           kommander-webhook-7b758645bd-xkk57                                1/1     Running             0               3h3m
kommander                           kube-oidc-proxy-7b8ccdf676-svvqs                                  1/1     Running             0               3h
kommander                           kube-prometheus-stack-grafana-5c965665bf-zgvft                    2/2     Running             0               9m24s
kommander                           kube-prometheus-stack-kube-state-metrics-8c5d9876b-wnf9s          1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-operator-8458bfb4df-jstdm                   1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-prometheus-node-exporter-42dg4              1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-prometheus-node-exporter-5k45l              1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-prometheus-node-exporter-87q9v              1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-prometheus-node-exporter-fbpj9              1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-prometheus-node-exporter-hpkzs              1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-prometheus-node-exporter-jdnq4              1/1     Running             0               9m24s
kommander                           kube-prometheus-stack-prometheus-node-exporter-n8cdq              1/1     Running             0               9m24s
kommander                           kubecost-cost-analyzer-5874fc47d5-5587w                           2/2     Running             0               12m
kommander                           kubecost-grafana-96d5947cd-kllf4                                  3/3     Running             0               12m
kommander                           kubecost-kube-state-metrics-6645c45576-d2lmv                      1/1     Running             0               12m
kommander                           kubecost-prometheus-alertmanager-55f9cf456b-89w8z                 2/2     Running             0               12m
kommander                           kubecost-prometheus-server-776b57c895-8nbmn                       3/3     Running             0               12m
kommander                           kubecost-traefik-certs-kommander-cert-federation-c455687b-q8dzq   1/1     Running             0               3h2m
kommander                           kubernetes-dashboard-api-5f5b7994c-kzbgt                          1/1     Running             0               9m7s
kommander                           kubernetes-dashboard-auth-8c7546cfc-hcfdn                         1/1     Running             0               9m7s
kommander                           kubernetes-dashboard-kong-699fffc989-znmdv                        1/1     Running             0               9m7s
kommander                           kubernetes-dashboard-metrics-scraper-5998c4d7d-pshrt              1/1     Running             0               9m7s
kommander                           kubernetes-dashboard-web-6f4466f486-ww949                         1/1     Running             0               9m7s
kommander                           kubetunnel-59598865c5-7q97c                                       1/1     Running             0               12m
kommander                           kubetunnel-webhook-55d9974f85-nlv86                               1/1     Running             0               12m
kommander                           logging-operator-5497588957-2x4hs                                 1/1     Running             0               11m
kommander                           logging-operator-logging-fluentbit-52p52                          1/1     Running             0               3m23s
kommander                           logging-operator-logging-fluentbit-88mbm                          1/1     Running             0               3m23s
kommander                           logging-operator-logging-fluentbit-8pszh                          0/1     ContainerCreating   0               3m23s
kommander                           logging-operator-logging-fluentbit-drgdp                          1/1     Running             0               3m23s
kommander                           logging-operator-logging-fluentbit-dsbct                          1/1     Running             0               3m23s
kommander                           logging-operator-logging-fluentbit-prhkn                          1/1     Running             0               3m23s
kommander                           logging-operator-logging-fluentbit-vnrjl                          1/1     Running             0               3m23s
kommander                           logging-operator-logging-fluentd-0                                0/3     ContainerCreating   0               3m24s
kommander                           logging-operator-logging-fluentd-configcheck-82fdd7bc             0/1     Completed           0               7m35s
kommander                           nkp-insights-management-mgmt-cm-7d4dddd996-l8gr8                  1/1     Running             0               9m56s
kommander                           prometheus-adapter-cd86fdf97-8fppm                                1/1     Running             0               6m
kommander                           prometheus-kube-prometheus-stack-prometheus-0                     3/3     Running             0               8m59s
kommander                           prometheus-traefik-certs-kommander-cert-federation-677d4b4dlnmz   1/1     Running             0               3h2m
kommander                           rook-ceph-detect-version-v5mk2                                    0/1     PodInitializing     0               4m2s
kommander                           rook-ceph-operator-7bb9c5bb9f-gvgjs                               1/1     Running             0               9m4s
kommander                           rook-ceph-tools-5995cd86bd-6b9xk                                  0/1     ContainerCreating   0               4m3s
kommander                           runtime-extension-kommander-59d9c676b7-vdk76                      1/1     Running             0               3h15m
kommander                           thanos-query-84b7859896-5fmc4                                     1/1     Running             0               10m
kommander                           traefik-forward-auth-mgmt-7b4848595b-z4kpq                        1/1     Running             0               175m
kommander                           velero-pre-install-bqxxz                                          1/1     Running             0               11m
kommander                           weaviate-0                                                        1/1     Running             0               11m
kube-federation-system              kubefed-admission-webhook-575c45986d-9c4ff                        1/1     Running             0               3h4m
kube-federation-system              kubefed-controller-manager-c6d6989b8-77bwd                        1/1     Running             0               3h3m
kube-federation-system              kubefed-controller-manager-c6d6989b8-7m48r                        1/1     Running             0               3h3m
kube-system                         cilium-2s6jn                                                      1/1     Running             0               3h21m
kube-system                         cilium-5rssh                                                      1/1     Running             0               3h19m
kube-system                         cilium-mswcf                                                      1/1     Running             0               3h20m
kube-system                         cilium-n7h7f                                                      1/1     Running             0               3h19m
kube-system                         cilium-operator-58c5d5d6d-sxxp8                                   1/1     Running             0               3h21m
kube-system                         cilium-operator-58c5d5d6d-vknvd                                   1/1     Running             0               3h21m
kube-system                         cilium-p89dz                                                      1/1     Running             0               3h18m
kube-system                         cilium-ph4px                                                      1/1     Running             0               3h20m
kube-system                         cilium-prc67                                                      1/1     Running             0               3h19m
kube-system                         coredns-76f75df574-tnk7v                                          1/1     Running             0               3h21m
kube-system                         coredns-76f75df574-w84hk                                          1/1     Running             0               3h21m
kube-system                         etcd-nkp-mgmt-hzxnp-28zsd                                         1/1     Running             0               3h21m
kube-system                         etcd-nkp-mgmt-hzxnp-cnjn6                                         1/1     Running             0               3h20m
kube-system                         etcd-nkp-mgmt-hzxnp-gg88v                                         1/1     Running             0               3h18m
kube-system                         kube-apiserver-nkp-mgmt-hzxnp-28zsd                               1/1     Running             0               3h21m
kube-system                         kube-apiserver-nkp-mgmt-hzxnp-cnjn6                               1/1     Running             0               3h20m
kube-system                         kube-apiserver-nkp-mgmt-hzxnp-gg88v                               1/1     Running             0               3h18m
kube-system                         kube-controller-manager-nkp-mgmt-hzxnp-28zsd                      1/1     Running             0               3h21m
kube-system                         kube-controller-manager-nkp-mgmt-hzxnp-cnjn6                      1/1     Running             0               3h20m
kube-system                         kube-controller-manager-nkp-mgmt-hzxnp-gg88v                      1/1     Running             0               3h18m
kube-system                         kube-proxy-76jcm                                                  1/1     Running             0               3h19m
kube-system                         kube-proxy-96pjt                                                  1/1     Running             0               3h19m
kube-system                         kube-proxy-9qrlm                                                  1/1     Running             0               3h18m
kube-system                         kube-proxy-cxw6r                                                  1/1     Running             0               3h20m
kube-system                         kube-proxy-jkl95                                                  1/1     Running             0               3h20m
kube-system                         kube-proxy-kr5wg                                                  1/1     Running             0               3h21m
kube-system                         kube-proxy-ljlhh                                                  1/1     Running             0               3h19m
kube-system                         kube-scheduler-nkp-mgmt-hzxnp-28zsd                               1/1     Running             0               3h21m
kube-system                         kube-scheduler-nkp-mgmt-hzxnp-cnjn6                               1/1     Running             0               3h20m
kube-system                         kube-scheduler-nkp-mgmt-hzxnp-gg88v                               1/1     Running             0               3h18m
kube-system                         kube-vip-nkp-mgmt-hzxnp-28zsd                                     1/1     Running             1 (3h21m ago)   3h21m
kube-system                         kube-vip-nkp-mgmt-hzxnp-cnjn6                                     1/1     Running             0               3h19m
kube-system                         kube-vip-nkp-mgmt-hzxnp-gg88v                                     1/1     Running             0               3h18m
kube-system                         nutanix-cloud-controller-manager-598b4c7669-4q8j8                 1/1     Running             0               3h21m
kube-system                         snapshot-controller-5c7f9fc58-7zjjg                               1/1     Running             0               3h21m
kubecost                            copy-kubecost-grafana-datasource-cm-mv7cm                         0/1     Completed           0               9m33s
kubecost                            create-kubecost-thanos-query-stores-configmap-j84tx               0/1     Completed           0               12m
kubecost                            kommander-kubecost-cost-analyzer-dbbfd778c-pfs97                  2/2     Running             0               10m
kubecost                            kommander-kubecost-thanos-query-656956b46b-xmltk                  1/1     Running             0               10m
kubecost                            kommander-kubecost-thanos-query-frontend-8f5774f6f-xsznt          1/1     Running             0               10m
metallb-system                      metallb-controller-94f95d674-48z4s                                1/1     Running             0               3h21m
metallb-system                      metallb-speaker-6lclp                                             4/4     Running             0               3h18m
metallb-system                      metallb-speaker-c78xt                                             4/4     Running             0               3h18m
metallb-system                      metallb-speaker-cnljr                                             4/4     Running             0               3h18m
metallb-system                      metallb-speaker-dklnr                                             4/4     Running             0               3h20m
metallb-system                      metallb-speaker-fmbzv                                             4/4     Running             0               3h18m
metallb-system                      metallb-speaker-nwq8b                                             4/4     Running             0               3h20m
metallb-system                      metallb-speaker-nxhz6                                             4/4     Running             0               3h19m
nkp-workload-9x6hx-2k79n            cluster-autoscaler-0192467d-9888-7536-8266-ed9e557d46a1-66qkh7z   1/1     Running             1 (91m ago)     102m
nkp-workload-9x6hx-2k79n            cluster-observer-1774060518-68b9989d54-dzcqt                      1/1     Running             0               103m
nkp-workload-9x6hx-2k79n            karma-traefik-certs-nkp-workload-9x6hx-2k79n-cert-federatish7kf   1/1     Running             0               104m
nkp-workload-9x6hx-2k79n            kubecost-traefik-certs-nkp-workload-9x6hx-2k79n-cert-federn6pf9   1/1     Running             0               104m
nkp-workload-9x6hx-2k79n            prometheus-traefik-certs-nkp-workload-9x6hx-2k79n-cert-fedh9vpc   1/1     Running             0               104m
node-feature-discovery              node-feature-discovery-gc-7f54d58d99-zvgnb                        1/1     Running             0               3h21m
node-feature-discovery              node-feature-discovery-master-ccf75997b-dp7q4                     1/1     Running             0               3h21m
node-feature-discovery              node-feature-discovery-worker-42bxq                               1/1     Running             0               3h19m
node-feature-discovery              node-feature-discovery-worker-478dl                               1/1     Running             0               3h20m
node-feature-discovery              node-feature-discovery-worker-79lmw                               1/1     Running             0               3h18m
node-feature-discovery              node-feature-discovery-worker-7kvw6                               1/1     Running             0               3h18m
node-feature-discovery              node-feature-discovery-worker-7tvvn                               1/1     Running             0               3h18m
node-feature-discovery              node-feature-discovery-worker-dlj8p                               1/1     Running             0               3h18m
node-feature-discovery              node-feature-discovery-worker-hpbmp                               1/1     Running             0               3h20m
ntnx-system                         nutanix-csi-controller-754fcf5f85-c9vtt                           7/7     Running             3 (3h14m ago)   3h18m
ntnx-system                         nutanix-csi-controller-754fcf5f85-sfrpb                           7/7     Running             3 (3h14m ago)   3h18m
ntnx-system                         nutanix-csi-node-5d9gs                                            3/3     Running             0               3h18m
ntnx-system                         nutanix-csi-node-8qhkx                                            3/3     Running             1 (3h17m ago)   3h18m
ntnx-system                         nutanix-csi-node-dbxm8                                            3/3     Running             1 (3h16m ago)   3h18m
ntnx-system                         nutanix-csi-node-qvkv9                                            3/3     Running             0               3h18m
ntnx-system                         nutanix-csi-node-s2wn7                                            3/3     Running             1 (3h17m ago)   3h18m
ntnx-system                         nutanix-csi-node-tq5bt                                            3/3     Running             1 (3h16m ago)   3h18m
ntnx-system                         nutanix-csi-node-vz87g                                            3/3     Running             0               3h18m
```

## Infrastructure Providers

### Nutanix

![image-20241001153625463](https://kenkenny.synology.me:5543/images/2024/10/image-20241001153625463.png)

![image-20241001153942117](https://kenkenny.synology.me:5543/images/2024/10/image-20241001153942117.png)

![image-20241001154001678](https://kenkenny.synology.me:5543/images/2024/10/image-20241001154001678.png)

![image-20241001154014865](https://kenkenny.synology.me:5543/images/2024/10/image-20241001154014865.png)



## Add Cluster

### Attach OCP

![image-20241001154845204](https://kenkenny.synology.me:5543/images/2024/10/image-20241001154845204.png)

![image-20241001154903514](https://kenkenny.synology.me:5543/images/2024/10/image-20241001154903514.png)

![image-20241001154938137](https://kenkenny.synology.me:5543/images/2024/10/image-20241001154938137.png)

![image-20241001155058938](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155058938.png)

![image-20241001155118242](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155118242.png)

![image-20241001155215935](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155215935.png)

![image-20241001155311069](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155311069.png)

Enable Dashboard

![image-20241001155331923](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155331923.png)

![image-20241001155440515](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155440515.png)

![image-20241001155522350](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155522350.png)

![image-20241001155550632](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155550632.png)

![image-20241001155642007](https://kenkenny.synology.me:5543/images/2024/10/image-20241001155642007.png)

Enable Kubecost

![image-20241017143816425](https://kenkenny.synology.me:5543/images/2024/10/image-20241017143816425.png)

![image-20241017143836312](https://kenkenny.synology.me:5543/images/2024/10/image-20241017143836312.png)

## LDAP

[KB-16668](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA0VO0000001mcT0AQ)

連線 AD 設定RBAC

1. 新增 Identitiy Providers ，選擇 LDAP

   ![image-20241004133412971](https://kenkenny.synology.me:5543/images/2024/10/image-20241004133412971.png)

   ![image-20241004133438174](https://kenkenny.synology.me:5543/images/2024/10/image-20241004133438174.png)

2. 輸入資訊

   ```
   Workspaces: All Workspaces # 也可以針對不同workspaces去設定
   Name: dc01.nutanixlab.local # 可辨識名稱即可
   Host: 192.168.102.1:389 # IP + Port
   Bind DN: CN=Administrator,CN=Users,DC=nutanixlab,DC=local # 綁定的管理員帳號
   Bind Password: ****
   ```

   ![image-20241004133902167](https://kenkenny.synology.me:5543/images/2024/10/image-20241004133902167.png)

   ![image-20241004144717369](https://kenkenny.synology.me:5543/images/2024/10/image-20241004144717369.png)

3. 輸入資訊

   ```
   Root CA: # 如果有的話才需要
   User Search Base DN: DC=nutanixlab,DC=local # 搜尋整個目錄
   User Search Username: sAMAccountName # 登入會輸入的帳號格式
   User Search Filter: (objectClass=person)
   User Search Scope: sub # 整個樹系或是一個Level
   User Search ID Attribute: DN
   ```

   ![image-20241004150808142](https://kenkenny.synology.me:5543/images/2024/10/image-20241004150808142.png)

4. 輸入資訊

   ```
   User Search E-Mail: mail # 或是UPN
   User Search Name: name # 登入後右上角顯示的名稱
   User Search E-mail Suffix:   # 留空
   Group Search Base DN: CN=Users,DC=nutanixlab,DC=local # 搜尋Group的地方
   Group Search Filter: (objectClass=group) # Group的object class
   Group Search Scope: sub
   Group Search Name Attribute: cn # 顯示的group名稱
   ```

   ![image-20241004143715589](https://kenkenny.synology.me:5543/images/2024/10/image-20241004143715589.png)

   ![image-20241004140340501](https://kenkenny.synology.me:5543/images/2024/10/image-20241004140340501.png)

   ![image-20241004150431207](https://kenkenny.synology.me:5543/images/2024/10/image-20241004150431207.png)

5. 設定完成 Save

   ![image-20241004143741938](https://kenkenny.synology.me:5543/images/2024/10/image-20241004143741938.png)

6. 新增Group

   ![image-20241004143813995](https://kenkenny.synology.me:5543/images/2024/10/image-20241004143813995.png)

   ![image-20241004144350520](https://kenkenny.synology.me:5543/images/2024/10/image-20241004144350520.png)

   ![image-20241004144415237](https://kenkenny.synology.me:5543/images/2024/10/image-20241004144415237.png)

7. Access Control 新增 Role Bindings

   ![image-20241004144451254](https://kenkenny.synology.me:5543/images/2024/10/image-20241004144451254.png)

   ![image-20241004144513748](https://kenkenny.synology.me:5543/images/2024/10/image-20241004144513748.png)

8. 登入測試 dc01.nutanixlab.local

   ![image-20241004144532799](https://kenkenny.synology.me:5543/images/2024/10/image-20241004144532799.png)

   ![image-20241004150544942](https://kenkenny.synology.me:5543/images/2024/10/image-20241004150544942.png)

   ![image-20241004150525532](https://kenkenny.synology.me:5543/images/2024/10/image-20241004150525532.png)

9. 其他使用者登入驗證OK

   ![image-20241017115248113](https://kenkenny.synology.me:5543/images/2024/10/image-20241017115248113.png)

```
[nkp@ken-rhel9 ~]$ kubectl get Connector.dex.mesosphere.io -n kommander  -o yaml
apiVersion: v1
items:
- apiVersion: dex.mesosphere.io/v1alpha1
  kind: Connector
  metadata:
    creationTimestamp: "2024-10-04T06:37:26Z"
    generateName: ldap-identity-provider-
    generation: 4
    name: ldap-identity-provider-hg4bx
    namespace: kommander
    resourceVersion: "8127529"
    uid: 709bdf88-b4d5-44be-98ba-0f3b5ee70db3
  spec:
    displayName: dc01.nutanixlab.local
    enabled: true
    ldap:
      bindDN: CN=Administrator,CN=Users,DC=nutanixlab,DC=local
      bindPW: Nutanix/Lab123
      bindSecretRef:
        name: connector-ldap-bindsecret-lf6zq
      groupSearch:
        baseDN: CN=Users,DC=nutanixlab,DC=local
        filter: (objectClass=group)
        nameAttr: cn
        scope: ""
        userMatchers:
        - groupAttr: member
          userAttr: DN
      host: 192.168.102.1:389
      insecureNoSSL: true
      insecureSkipVerify: true
      startTLS: false
      userSearch:
        baseDN: DC=nutanixlab,DC=local
        emailAttr: mail
        emailSuffix: ""
        filter: (objectClass=person)
        idAttr: DN
        nameAttr: name
        scope: sub
        username: sAMAccountName
    type: ldap
kind: List
metadata:
  resourceVersion: ""

```

Check Logs

```
[nkp@ken-rhel9 ~]$ kubectl logs dex-55978d4bcc-ksgsg  -n kommander
time="2024-10-04T07:06:36Z" level=info msg="Dex Version: v2.37.0-d2iq.5-dirty, Go Version: go1.20.4, Go OS/ARCH: linux amd64"
time="2024-10-04T07:06:36Z" level=info msg="config using log level: debug"
time="2024-10-04T07:06:36Z" level=info msg="config issuer: https://172.16.90.209/dex"
...
time="2024-10-17T03:34:21Z" level=info msg="username \"ken.wang\" mapped to entry CN=Ken Wang,OU=NutanixTeam,OU=NetfosTaipei,DC=nutanixlab,DC=local"
time="2024-10-17T03:34:21Z" level=info msg="performing ldap search CN=Users,DC=nutanixlab,DC=local sub (&(objectClass=group)(member=CN=Ken Wang,OU=NutanixTeam,OU=NetfosTaipei,DC=nutanixlab,DC=local))"
time="2024-10-17T03:34:21Z" level=info msg="login successful: connector \"dex-controller_kommander_ldap-identity-provider-hg4bx\", username=\"Ken Wang\", preferred_username=\"\", email=\"ken.wang@nutanixlab.local\", groups=[\"Schema Admins\" \"Enterprise Admins\" \"Domain Admins\" \"Group Policy Creator Owners\" \"NKPAdmins\"]"

```



## AI Navigator

啟用 NKP Ultimate License 後，會自動安裝 AI 聊天機器人，主要功能為 直覺的查詢處理、即時問題處理協助、整合NKP

另外還有NKP AI Navigator Cluster Info Agent，描述的功能與NKP AI Navigator 相同，主要可以偵測整個叢集的版本等資訊

AI 助理回答如下：



<img src="https://kenkenny.synology.me:5543/images/2024/10/image-20241017140922240.png" alt="image-20241017140922240" style="zoom:50%;" />



<img src="https://kenkenny.synology.me:5543/images/2024/10/image-20241017141627809.png" alt="image-20241017141627809" style="zoom:50%;" />



```
Key Features

Intuitive Query Processing：
Empowered by Natural Language Processing (NLP), the AI Navigator interprets user queries, even if they’re not phrased technically, making it a go-to tool for both novices and experts.

Real-time Troubleshooting Assistance：
With its vast database of common NKP issues and solutions, the AI Navigator can provide instant solutions, dramatically reducing downtime and enhancing productivity.

Seamless NKP Integration：
Being native to the NKP platform, the AI Navigator ensures users get the most out of their NKP environments by offering insights, best practices, and performance optimization tips.

Privacy First Approach
Data privacy and security remain paramount, ensuring user trust and compliance.
```

AI Navigator 所安裝的 pod

```
[nkp@ken-rhel9 ~]$ kubectl get pods -n kommander |grep ai-navigator
ai-navigator-app-554f86cd46-797tq                                 1/1     Running     0             15d
ai-navigator-cluster-info-api-6b78d6449-fh756                     1/1     Running     0             15d
ai-navigator-cluster-info-api-6b78d6449-fn6pn                     1/1     Running     0             15d
ai-navigator-cluster-info-api-postgresql-0                        1/1     Running     0             15d

[nkp@ken-rhel9 ~]$ kubectl -n kommander describe pod ai-navigator-app-554f86cd46-797tq
Name:                 ai-navigator-app-554f86cd46-797tq
Namespace:            kommander
Priority:             100001000
Priority Class Name:  dkp-high-priority
Service Account:      ai-navigator-app
Node:                 nkp-mgmt-md-0-l965p-fczv8-n6tlc/172.16.90.171
Start Time:           Tue, 01 Oct 2024 14:42:44 +0800
Labels:               app=ai-navigator-app
                      pod-template-hash=554f86cd46
Annotations:          <none>
Status:               Running
IP:                   192.168.1.226
IPs:
  IP:           192.168.1.226
Controlled By:  ReplicaSet/ai-navigator-app-554f86cd46
Containers:
  ai-navigator-app:
    Container ID:    containerd://32691c6af7bcb6942c7d11de6a0c331c6aa75faf27d8cd9c98bc8e692b8a055c
    Image:           mesosphere/ai-navigator-app:v0.1.1
    Image ID:        docker.io/mesosphere/ai-navigator-app@sha256:2026ed7aabb265eead86917ad95de623aef7c13a775c7901a4629909460f5bce
    Port:            8080/TCP
    Host Port:       0/TCP
    SeccompProfile:  RuntimeDefault
    State:           Running
      Started:       Tue, 01 Oct 2024 14:42:56 +0800
    Ready:           True
  
```

![image-20241017120511998](https://kenkenny.synology.me:5543/images/2024/10/image-20241017120511998.png)

![image-20241017120050371](https://kenkenny.synology.me:5543/images/2024/10/image-20241017120050371.png)

啟用NKP AI Navigator Cluster Info Agent 前的回答

<img src="https://kenkenny.synology.me:5543/images/2024/10/image-20241017141711072.png" alt="image-20241017141711072" style="zoom:50%;" />

啟用Agent

![image-20241017141740569](https://kenkenny.synology.me:5543/images/2024/10/image-20241017141740569.png)

![image-20241017141750580](https://kenkenny.synology.me:5543/images/2024/10/image-20241017141750580.png)

大約等3分鐘後 ， Pod 新增一個 agent 

```
[nkp@ken-rhel9 ~]$ kubectl get pods -n kommander |grep ai-navigator
ai-navigator-app-554f86cd46-797tq                                 1/1     Running     0             15d
ai-navigator-cluster-info-agent-76bfb65f49-qdrqx                  1/1     Running     0             77s
ai-navigator-cluster-info-api-6b78d6449-fh756                     1/1     Running     0             15d
ai-navigator-cluster-info-api-6b78d6449-fn6pn                     1/1     Running     0             15d
ai-navigator-cluster-info-api-postgresql-0                        1/1     Running     0             15d
```

同樣問題已可回答

![image-20241017142131014](https://kenkenny.synology.me:5543/images/2024/10/image-20241017142131014.png)

驗證版本稍微有點不一樣，而且多問幾次的回答也不同，應該還要多點時間學習

![image-20241017143333852](https://kenkenny.synology.me:5543/images/2024/10/image-20241017143333852.png)

```
[nkp@ken-rhel9 ~]$ kubectl version
Client Version: v1.28.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.29.6

[nkp@ken-rhel9 ~]$ kubectl get nodes
NAME                              STATUS   ROLES           AGE   VERSION
nkp-mgmt-hzxnp-28zsd              Ready    control-plane   16d   v1.29.6
nkp-mgmt-hzxnp-cnjn6              Ready    control-plane   16d   v1.29.6
nkp-mgmt-hzxnp-gg88v              Ready    control-plane   16d   v1.29.6
nkp-mgmt-md-0-l965p-fczv8-mspnw   Ready    <none>          16d   v1.29.6
nkp-mgmt-md-0-l965p-fczv8-n6tlc   Ready    <none>          16d   v1.29.6
nkp-mgmt-md-0-l965p-fczv8-t5xmd   Ready    <none>          16d   v1.29.6
nkp-mgmt-md-0-l965p-fczv8-w7sxk   Ready    <none>          16d   v1.29.6

[nkp@ken-rhel9 ~]$ kubectl get pods -n kommander |grep ai-navigator
ai-navigator-app-554f86cd46-797tq                                 1/1     Running     0             15d
ai-navigator-cluster-info-agent-76bfb65f49-qdrqx                  1/1     Running     0             9m25s
ai-navigator-cluster-info-api-6b78d6449-24v4r                     1/1     Running     0             7m51s
ai-navigator-cluster-info-api-6b78d6449-bmxk2                     1/1     Running     0             7m51s
ai-navigator-cluster-info-api-6b78d6449-c5z4s                     1/1     Running     0             7m51s
ai-navigator-cluster-info-api-6b78d6449-fh756                     1/1     Running     0             15d
ai-navigator-cluster-info-api-6b78d6449-fn6pn                     1/1     Running     0             15d
ai-navigator-cluster-info-api-postgresql-0                        1/1     Running     0             15d
```

![image-20241017142451188](https://kenkenny.synology.me:5543/images/2024/10/image-20241017142451188.png)



## NDB Operator

https://github.com/nutanix-cloud-native/ndb-operator?tab=readme-ov-file

### Pre-install

#### 安裝 Operator-SDK

( Helm Install 可略過 )

1. 設定OS 版本

   ```
   [nkp@ken-rhel9 ~]$ export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
   
   [nkp@ken-rhel9 ~]$ export OS=$(uname | awk '{print tolower($0)}')
   ```

2. 下載Binary

   ```
   [nkp@ken-rhel9 ~]$ export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.37.0
   
   [nkp@ken-rhel9 ~]$ curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
     0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
   100 91.6M  100 91.6M    0     0  1994k      0  0:00:47  0:00:47 --:--:-- 2059k
   ```

   

3. 確認Binary

   ```
   [nkp@ken-rhel9 ~]$ gpg --keyserver keyserver.ubuntu.com --recv-keys 052996E2A20B5C7E
   gpg: directory '/home/nkp/.gnupg' created
   gpg: keybox '/home/nkp/.gnupg/pubring.kbx' created
   gpg: /home/nkp/.gnupg/trustdb.gpg: trustdb created
   gpg: key 052996E2A20B5C7E: public key "Operator SDK (release) <cncf-operator-sdk@cncf.io>" imported
   gpg: Total number processed: 1
   gpg:               imported: 1
   
   [nkp@ken-rhel9 ~]$ curl -LO ${OPERATOR_SDK_DL_URL}/checksums.txt
   
   [nkp@ken-rhel9 ~]$ curl -LO ${OPERATOR_SDK_DL_URL}/checksums.txt.asc
   
   [nkp@ken-rhel9 ~]$ gpg -u "Operator SDK (release) <cncf-operator-sdk@cncf.io>" --verify checksums.txt.asc
   
   [nkp@ken-rhel9 ~]$ grep operator-sdk_${OS}_${ARCH} checksums.txt | sha256sum -c -
   operator-sdk_linux_amd64: OK
   
   ```

4. 安裝並複製到/usr/loca/bin

   ```
   [nkp@ken-rhel9 ~]$ chmod +x operator-sdk_${OS}_${ARCH} && sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
   [sudo] password for nkp: 
   ```

   

5. 下載 git 跟 go

   https://git-scm.com/downloads

   https://go.dev/dl/

   ```
   [nkp@ken-rhel9 ~]$ git --version
   git version 2.31.1
   
   [nkp@ken-rhel9 ~]$ go version
   go version go1.22.2 linux/amd64
   ```

#### 確認 cert-manager 安裝

```
[nkp@ken-rhel9 ~]$ kubectl get pods -n cert-manager --kubeconfig=nkp-cluster01-kubeconfig.yaml
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-556c9f47d-c4d6q               1/1     Running   0          11d
cert-manager-cainjector-57d8d7ff64-cbzfg   1/1     Running   0          11d
cert-manager-webhook-844cfb8858-g8ftz      1/1     Running   0          11d
```

#### 確認 NDB 連線

```
NDB: 172.16.90.196

[nkp@ken-rhel9 ~]$ kubectl get nodes -o wide --kubeconfig=nkp-cluster01-kubeconfig.yaml
NAME                                   STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                      KERNEL-VERSION                 CONTAINER-RUNTIME
nkp-cluster01-9swsx-zwgx9              Ready    control-plane   11d   v1.29.6   172.16.90.126   <none>        Rocky Linux 9.4 (Blue Onyx)   5.14.0-427.31.1.el9_4.x86_64   containerd://1.6.33-d2iq.1
nkp-cluster01-md-0-ztljf-l4nbq-5xnsc   Ready    <none>          11d   v1.29.6   172.16.90.127   <none>        Rocky Linux 9.4 (Blue Onyx)   5.14.0-427.31.1.el9_4.x86_64   containerd://1.6.33-d2iq.1
nkp-cluster01-md-0-ztljf-l4nbq-vhfvq   Ready    <none>          11d   v1.29.6   172.16.90.128   <none>        Rocky Linux 9.4 (Blue Onyx)   5.14.0-427.31.1.el9_4.x86_64   containerd://1.6.33-d2iq.1

[nkp@ken-rhel9 ~]$ ssh nkp@172.16.90.126

[nkp@nkp-cluster01-9swsx-zwgx9 ~]$ ping 172.16.90.196
PING 172.16.90.196 (172.16.90.196) 56(84) bytes of data.
64 bytes from 172.16.90.196: icmp_seq=1 ttl=64 time=2.92 ms
^C
--- 172.16.90.196 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.921/2.921/2.921/0.000 ms

```



### Install

#### Github Repo

1. git clone

   ```
   [nkp@ken-rhel9 ~]$ git clone https://github.com/nutanix-cloud-native/ndb-operator.git
   Cloning into 'ndb-operator'...
   remote: Enumerating objects: 3345, done.
   remote: Counting objects: 100% (1264/1264), done.
   remote: Compressing objects: 100% (520/520), done.
   remote: Total 3345 (delta 974), reused 793 (delta 743), pack-reused 2081 (from 1)
   Receiving objects: 100% (3345/3345), 772.31 KiB | 1.58 MiB/s, done.
   Resolving deltas: 100% (2073/2073), done.
   
   [nkp@ken-rhel9 ~]$ ls -l |grep ndb
   drwxr-xr-x. 13 nkp nkp      4096 Nov 11 09:55 ndb-operator
   ```

   

2. cd ndb-operator 後安裝於 kubernetes 中

   ```
   [nkp@ken-rhel9 ~]$ cd ndb-operator/
   [nkp@ken-rhel9 ndb-operator]$ make deploy
   ```



#### Helm Chart 

1. Add Nutanix Repository

   ```
   [nkp@ken-rhel9 ~]$ helm repo add nutanix https://nutanix.github.io/helm/ --kubeconfig=nkp-cluster01-kubeconfig.yaml
   "nutanix" has been added to your repositories
   
   [nkp@ken-rhel9 ~]$ helm repo list --kubeconfig=nkp-cluster01-kubeconfig.yaml
   NAME   	URL                            
   nutanix	https://nutanix.github.io/helm/
   ```

   

2. Install Chart

   ```
   [nkp@ken-rhel9 ~]$ kubectl create ns ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   namespace/ndb-operator created
   
   [nkp@ken-rhel9 ~]$ helm install netfos-ndb-operator -n ndb-operator nutanix/ndb-operator --version 0.5.3 --kubeconfig=nkp-cluster01-kubeconfig.yaml
   
   NAME: netfos-ndb-operator
   LAST DEPLOYED: Mon Nov 11 10:20:27 2024
   NAMESPACE: ndb-operator
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   
   ```

#### Create Secret and Connet

1. 建立 ndb-secret 連線用的secret

   ```
   [nkp@ken-rhel9 ndb]$ vim netfos-ndb-secret.yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: netfos-ndb-secret
   type: Opaque
   stringData:
     username: admin
     password: Nutanix/Lab123 
     
   [nkp@ken-rhel9 ndb]$ kubectl apply -f netfos-ndb-secret.yaml -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   ```

   

2. 連線 ndb-server

   ```
   [nkp@ken-rhel9 ndb]$ vim ndb-server.yaml
   apiVersion: ndb.nutanix.com/v1alpha1
   kind: NDBServer
   metadata:
     labels:
       app.kubernetes.io/name: ndbserver
       app.kubernetes.io/instance: ndbserver
       app.kubernetes.io/part-of: ndb-operator
       app.kubernetes.io/managed-by: kustomize
       app.kubernetes.io/created-by: ndb-operator
     name: ndb
   spec:
       # Name of the secret that holds the credentials for NDB: username, password and ca_certificate created earlier
       credentialSecret: netfos-ndb-secret
       # NDB Server's API URL
       server: https://172.16.90.196/era/v0.9
       # Set to true to skip SSL certificate validation, should be false if ca_certificate is provided in the credential secret.
       skipCertificateVerification: true
       
   [nkp@ken-rhel9 ndb]$ kubectl apply -f ndb-server.yaml -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.
   
   [nkp@ken-rhel9 ndb]$ kubectl get pods -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   NAME                                                      READY   STATUS    RESTARTS   AGE
   netfos-ndb-operator-controller-manager-5c77798ddf-7xz49   2/2     Running   0          13m
   
   [nkp@ken-rhel9 ndb]$ kubectl get pods -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   NAME                                                      READY   STATUS    RESTARTS   AGE
   netfos-ndb-operator-controller-manager-5c77798ddf-7xz49   2/2     Running   0          13m
   
   [nkp@ken-rhel9 ndb]$ kubectl describe pod netfos-ndb-operator-controller-manager-5c77798ddf-7xz49 -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yamlName:             netfos-ndb-operator-controller-manager-5c77798ddf-7xz49
   Namespace:        ndb-operator
   Priority:         0
   Service Account:  netfos-ndb-operator-service-account
   Node:             nkp-cluster01-md-0-ztljf-l4nbq-vhfvq/172.16.90.128
   Start Time:       Mon, 11 Nov 2024 10:21:00 +0800
   Labels:           control-plane=controller-manager
                     pod-template-hash=5c77798ddf
   Annotations:      kubectl.kubernetes.io/default-container: manager
   Status:           Running
   IP:               192.168.2.57
   IPs:
     IP:           192.168.2.57
   Controlled By:  ReplicaSet/netfos-ndb-operator-controller-manager-5c77798ddf
   Containers:
     manager:
       Container ID:  containerd://3a8754ee1381f2bb87fdffe202ae91770f19642d690f552086e1b451d3869fcf
       Image:         ghcr.io/nutanix-cloud-native/ndb-operator/controller:v0.5.1
       Image ID:      ghcr.io/nutanix-cloud-native/ndb-operator/controller@sha256:5579d0349f5dc8b7f45d0eaddfe6ad087aa92ad925c799d4fe83eda7beba4ac5
       Port:          9443/TCP
       Host Port:     0/TCP
       Args:
         --health-probe-bind-address=:8081
         --metrics-bind-address=127.0.0.1:8080
         --leader-elect
       State:          Running
         Started:      Mon, 11 Nov 2024 10:21:17 +0800
       Ready:          True
       Restart Count:  0
       Limits:
         cpu:     500m
         memory:  128Mi
       Requests:
         cpu:        10m
         memory:     64Mi
       Liveness:     http-get http://:8081/healthz delay=15s timeout=1s period=20s #success=1 #failure=3
       Readiness:    http-get http://:8081/readyz delay=5s timeout=1s period=10s #success=1 #failure=3
       Environment:  <none>
       Mounts:
         /tmp/k8s-webhook-server/serving-certs from cert (ro)
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hmp6h (ro)
     kube-rbac-proxy:
       Container ID:  containerd://20c40b65d22a766217ce8ead5db6ed7d6d3316a676c733f916e859cdf6f87557
       Image:         gcr.io/kubebuilder/kube-rbac-proxy:v0.16.0
       Image ID:      gcr.io/kubebuilder/kube-rbac-proxy@sha256:771a9a173e033a3ad8b46f5c00a7036eaa88c8d8d1fbd89217325168998113ea
       Port:          8443/TCP
       Host Port:     0/TCP
       Args:
         --secure-listen-address=0.0.0.0:8443
         --upstream=http://127.0.0.1:8080/
         --logtostderr=true
         --v=0
       State:          Running
         Started:      Mon, 11 Nov 2024 10:21:22 +0800
       Ready:          True
       Restart Count:  0
       Limits:
         cpu:     500m
         memory:  128Mi
       Requests:
         cpu:        5m
         memory:     64Mi
       Environment:  <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hmp6h (ro)
   Conditions:
     Type                        Status
     PodReadyToStartContainers   True 
     Initialized                 True 
     Ready                       True 
     ContainersReady             True 
     PodScheduled                True 
   
   [nkp@ken-rhel9 ndb]$ kubectl get all -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   NAME                                                          READY   STATUS    RESTARTS   AGE
   pod/netfos-ndb-operator-controller-manager-5c77798ddf-7xz49   2/2     Running   0          53m
   
   NAME                                                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   service/netfos-ndb-operator-controller-manager-metrics-service   ClusterIP   10.107.170.244   <none>        8443/TCP   53m
   service/netfos-ndb-operator-webhook-service                      ClusterIP   10.109.133.195   <none>        443/TCP    53m
   
   NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/netfos-ndb-operator-controller-manager   1/1     1            1           53m
   
   NAME                                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/netfos-ndb-operator-controller-manager-5c77798ddf   1         1         1       53m
   ```



### Post-install

#### Create postgres

1. Create db-instance-secret

   ```
   [nkp@ken-rhel9 ndb]$ cat db-postgres-secret.yaml 
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: ndb-postgres-secret
   type: Opaque
   stringData:
     password: P@55w.rd
     ssh_public_key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCVN+q1hhDCFrbW4Gw+wPryGl2BQbLi9NNOv+hXLgI+NXJqhTT0XEL6Uun5WKd7ceNznfiCHee2AkQXm9nrvVyPpFU5zQ91j9zQbMsFylSDv7RALFyEaXO45u6bzKRvVnL4lsKygBcMwxSW8yIrv0qTeDl3rChfkxjhpWR+C7gbn6uIVINRoC/5L1xNGcThbF0CiHgOt5P03ZV4tsm6C7Z65Wj41vczaH460qqq//kaf2uNveOZhi4juIQMc0FcE4kvGwtqRJ5HIs8CRq1N/wX+uIIQEKsqtmQqSujcHOwosYPSNvVXvVgqvy9t0jfEU4Y3bcBkoF5zfq98OCG0978ewjb8yR909bQM2s2QRoQML8Gb6Be+EV7+xtUcqZOpAkyZOmWXN8yoL5EMcelgDZoyK3yH9FgKuz1sbB0f+4b1B8LM/SULTq8Lv7ww6ps3EIvKDwbH5rMQIxf3IEIuk7qvjyQlAZiE2nxiU88fcPB54WUoQBNd3uUjr0YVt1i5GM0= nkp@ken-rhel9.nutanixlab.local
     
   [nkp@ken-rhel9 ndb]$ kubectl apply -f db-postgres-secret.yaml -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   
   [nkp@ken-rhel9 ndb]$ kubectl get secret -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   NAME                                        TYPE                 DATA   AGE
   db-postgres-secret                          Opaque               2      5s
   netfos-ndb-secret                           Opaque               2      38m
   sh.helm.release.v1.netfos-ndb-operator.v1   helm.sh/release.v1   1      50m
   webhook-server-cert                         kubernetes.io/tls    3      50m
   ```

   

2. Cluster ID ( Nutanix PE Cluster 有註冊在NDB上的 )

   ![image-20241111112958254](https://kenkenny.synology.me:5543/images/2024/11/image-20241111112958254.png)

3. Create Postgres DB

   ```
   [nkp@ken-rhel9 ndb]$ cat db-postgres-deploy.yaml 
   apiVersion: ndb.nutanix.com/v1alpha1
   kind: Database
   metadata:
     name: nkp-postgres
   spec:
     ndbRef: ndb
     isClone: false
     databaseInstance:
       clusterId: "b9ab1352-1673-4ce1-a9d8-174771f3dcbb"
       name: "nkp-postgres"
       description: This for testing NDB Operator in NKP Cluster
       databaseNames:
         - nkp_database
       credentialSecret: db-postgres-secret
       size: 100
       timezone: "UTC"
       type: postgres
       # You can specify any (or none) of these types of profiles: compute, software, network, dbParam
       # If not specified, the corresponding Out-of-Box (OOB) profile will be used wherever applicable
       # Name is case-sensitive. ID is the UUID of the profile. Profile should be in the "READY" state
       # "id" & "name" are optional. If none provided, OOB may be resolved to any profile of that type
       profiles:
         compute:
           id: "8adb0c25-4136-4aa9-bf48-d0801d01e3e4"
           name: "DEFAULT_OOB_SMALL_COMPUTE"
         # A Software profile is a mandatory input for closed-source engines: SQL Server & Oracle
         software:
           name: "POSTGRES_15.5_Rocky8.10"
           id: "71466901-fba3-4391-a9b3-b77d8933de46"
         network:
           id: "731cf689-e55a-4b05-a8ac-98ff8a18af13"
           name: "DEFAULT_OOB_POSTGRESQL_NETWORK"
         dbParam:
           name: "DEFAULT_POSTGRES_PARAMS"
           id: "c6c8517a-3697-4090-b5b7-08632c77bdce"
       
   [nkp@ken-rhel9 ndb]$ kubectl apply -f db-postgres-deploy.yaml -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   
   看logs
   [nkp@ken-rhel9 ndb]$ kubectl logs -f netfos-ndb-operator-controller-manager-5c77798ddf-7xz49 -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   ```

   

4. 從NDB查看進度

   ![image-20241122150444883](https://kenkenny.synology.me:5543/images/2024/11/image-20241122150444883.png)

   ![image-20241122151313720](https://kenkenny.synology.me:5543/images/2024/11/image-20241122151313720.png)

5. 查看Logs

   ![image-20241122150620515](https://kenkenny.synology.me:5543/images/2024/11/image-20241122150620515.png)

6. 部署完成即可看到此Service

   ![image-20241122152959990](https://kenkenny.synology.me:5543/images/2024/11/image-20241122152959990.png)

   ```
   [nkp@ken-rhel9 ndb]$ kubectl --kubeconfig=nkp-cluster01-kubeconfig.yaml get all -n ndb-operator
   NAME                                                          READY   STATUS    RESTARTS        AGE
   pod/netfos-ndb-operator-controller-manager-5c77798ddf-7xz49   2/2     Running   1 (6d22h ago)   11d
   
   NAME                                                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   service/netfos-ndb-operator-controller-manager-metrics-service   ClusterIP   10.107.170.244   <none>        8443/TCP   11d
   service/netfos-ndb-operator-webhook-service                      ClusterIP   10.109.133.195   <none>        443/TCP    11d
   service/netfos-nkp-mssql-svc                                     ClusterIP   10.108.221.85    <none>        80/TCP     10d
   service/nkp-postgres-svc                                         ClusterIP   10.96.183.27     <none>        80/TCP     5m40s
   
   NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/netfos-ndb-operator-controller-manager   1/1     1            1           11d
   
   NAME                                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/netfos-ndb-operator-controller-manager-5c77798ddf   1         1         1       11d
   
   [nkp@ken-rhel9 ndb]$ kubectl --kubeconfig=nkp-cluster01-kubeconfig.yaml describe service/nkp-postgres-svc  -n ndb-operator
   Name:              nkp-postgres-svc
   Namespace:         ndb-operator
   Labels:            <none>
   Annotations:       <none>
   Selector:          <none>
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                10.96.183.27
   IPs:               10.96.183.27
   Port:              <unset>  80/TCP
   TargetPort:        5432/TCP
   Endpoints:         172.16.90.164:5432
   Session Affinity:  None
   Events:            <none>
   
   # 列出 database
   [nkp@ken-rhel9 ndb]$ kubectl --kubeconfig=nkp-cluster01-kubeconfig.yaml get database -n ndb-operator
   NAME               IP ADDRESS      STATUS   TYPE
   netfos-nkp-mssql   172.16.90.163   READY    mssql
   nkp-postgres       172.16.90.164   READY    postgres
   
   # 描述 database (包含Cloen會用到的database ID)
   [nkp@ken-rhel9 ndb]$ kubectl --kubeconfig=nkp-cluster01-kubeconfig.yaml describe database nkp-postgres -n ndb-operator
   Name:         nkp-postgres
   Namespace:    ndb-operator
   Labels:       <none>
   Annotations:  <none>
   API Version:  ndb.nutanix.com/v1alpha1
   Kind:         Database
   Metadata:
     Creation Timestamp:  2024-11-22T06:58:19Z
     Finalizers:
       ndb.nutanix.com/finalizerinstance
       ndb.nutanix.com/finalizerserver
     Generation:        1
     Resource Version:  18657950
     UID:               c9ae56f5-a86e-44ba-a6be-b96e3ac89767
   Spec:
     Clone:
       Additional Arguments:
       Cluster Id:         
       Credential Secret:  
       Description:        
       Name:               
       Profiles:
         Compute:
           Id:    
           Name:  
         Db Param:
           Id:    
           Name:  
         Db Param Instance:
           Id:    
           Name:  
         Network:
           Id:    
           Name:  
         Software:
           Id:              
           Name:            
       Snapshot Id:         
       Source Database Id:  
       Timezone:            
       Type:                
     Database Instance:
       Additional Arguments:
       Cluster Id:         b9ab1352-1673-4ce1-a9d8-174771f3dcbb
       Credential Secret:  db-postgres-secret
       Database Names:
         nkp_database
       Description:  This for testing NDB Operator in NKP Cluster
       Name:         nkp-postgres
       Profiles:
         Compute:
           Id:    8adb0c25-4136-4aa9-bf48-d0801d01e3e4
           Name:  DEFAULT_OOB_SMALL_COMPUTE
         Db Param:
           Id:    c6c8517a-3697-4090-b5b7-08632c77bdce
           Name:  DEFAULT_POSTGRES_PARAMS
         Db Param Instance:
           Id:    
           Name:  
         Network:
           Id:    731cf689-e55a-4b05-a8ac-98ff8a18af13
           Name:  DEFAULT_OOB_POSTGRESQL_NETWORK
         Software:
           Id:    71466901-fba3-4391-a9b3-b77d8933de46
           Name:  POSTGRES_15.5_Rocky8.10
       Size:      100
       Time Machine:
         Daily Snapshot Time:       04:00:00
         Description:               
         Log Catch Up Frequency:    30
         Monthly Snapshot Day:      15
         Name:                      
         Quarterly Snapshot Month:  Jan
         Sla:                       NONE
         Snapshots Per Day:         1
         Weekly Snapshot Day:       FRIDAY
       Timezone:                    UTC
       Type:                        postgres
     Is Clone:                      false
     Ndb Ref:                       ndb
   Status:
     Creation Operation Id:        c60d8a4a-7c20-4a22-8006-f48fa28b9ff7
     Db Server Id:                 4ef4c7d0-c891-460d-9c9a-4b71032a3d5a
     Deregistration Operation Id:  
     Id:                           7e9f5189-adbf-4228-abb1-b6f69cb13d3e ## Clone 會用到此Database Id
     Ip Address:                   172.16.90.164
     Status:                       READY
     Type:                         postgres
   Events:                         <none>
   
   ```

   

7. 確認Endpoints

   ![image-20241122153104705](https://kenkenny.synology.me:5543/images/2024/11/image-20241122153104705.png)

8. 測試連線 用kubernetes service 方式

   ```
   # 啟動測試容器
   [nkp@ken-rhel9 ndb]$ kubectl --kubeconfig=nkp-cluster01-kubeconfig.yaml -n ndb-operator run -i --tty postgres --image=postgres --restart=Never -- sh
   
   # 連線 kubernetes service （預設帶80 port）
   # psql -U postgres -p 80 -h nkp-postgres-svc -W
   Password: 
   psql (17.1 (Debian 17.1-1.pgdg120+1), server 15.5)
   Type "help" for help.
   
   # 列出 database
   postgres=# \l
                                                          List of databases
        Name     |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
   --------------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
    nkp_database | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
    postgres     | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
    template0    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
                 |          |          |                 |             |             |        |           | postgres=CTc/postgres
    template1    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
                 |          |          |                 |             |             |        |           | postgres=CTc/postgres
   (4 rows)
   
   # 新增 test database
   postgres=# create database test;
   CREATE DATABASE
   
   # 列出 database
   postgres=# \l
                                                          List of databases
        Name     |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
   --------------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
    nkp_database | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
    postgres     | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
    template0    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
                 |          |          |                 |             |             |        |           | postgres=CTc/postgres
    template1    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
                 |          |          |                 |             |             |        |           | postgres=CTc/postgres
    test         | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
   (5 rows)
   
   # 移除 test database
   postgres=# drop database test;
   DROP DATABASE
   
   # 退出
   \q
   ```

   

9. 測試連線 直連 VM IP 方式

   ```
   # 啟動測試容器
   [nkp@ken-rhel9 ndb]$ kubectl --kubeconfig=nkp-cluster01-kubeconfig.yaml -n ndb-operator run -i --tty postgres --image=postgres --restart=Never -- sh
   
   # 連線 VM 的 IP 以及 p 5432 
   # psql -U postgres -p 5432 -h 172.16.90.164 -W
   Password: 
   psql (17.1 (Debian 17.1-1.pgdg120+1), server 15.5)
   Type "help" for help.
   
   # 列出database
   postgres=# \l
                                                          List of databases
        Name     |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
   --------------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
    nkp_database | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
    postgres     | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
    template0    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
                 |          |          |                 |             |             |        |           | postgres=CTc/postgres
    template1    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
                 |          |          |                 |             |             |        |           | postgres=CTc/postgres
   (4 rows)
   ```

   ![image-20241122172515700](https://kenkenny.synology.me:5543/images/2024/11/image-20241122172515700.png)

   

   ![image-20241122172535411](https://kenkenny.synology.me:5543/images/2024/11/image-20241122172535411.png)

##### Clone Database

1. 建立Clone manifest 
   Snapshot Id 從 NDB API Exporter上查看

   ```
   apiVersion: ndb.nutanix.com/v1alpha1
   kind: Database
   metadata:
     name: nkp-postgres-clone
   spec:
     ndbRef: ndb
     isClone: true
     clone:
       clusterId: "b9ab1352-1673-4ce1-a9d8-174771f3dcbb"
       name: "nkp-postgres-clone"
       description: This for testing NDB Operator Cloning in NKP Cluster
       credentialSecret: db-postgres-secret
       timezone: "UTC"
       type: postgres
       # You can specify any (or none) of these types of profiles: compute, software, network, dbParam
       # If not specified, the corresponding Out-of-Box (OOB) profile will be used wherever applicable
       # Name is case-sensitive. ID is the UUID of the profile. Profile should be in the "READY" state
       # "id" & "name" are optional. If none provided, OOB may be resolved to any profile of that type
       profiles:
         compute:
           id: "8adb0c25-4136-4aa9-bf48-d0801d01e3e4"
           name: "DEFAULT_OOB_SMALL_COMPUTE"
         # A Software profile is a mandatory input for closed-source engines: SQL Server & Oracle
         software:
           name: "POSTGRES_15.5_Rocky8.10"
           id: "71466901-fba3-4391-a9b3-b77d8933de46"
         network:
           id: "731cf689-e55a-4b05-a8ac-98ff8a18af13"
           name: "DEFAULT_OOB_POSTGRESQL_NETWORK"
         dbParam:
           name: "DEFAULT_POSTGRES_PARAMS"
           id: "c6c8517a-3697-4090-b5b7-08632c77bdce"
       sourceDatabaseId: "7e9f5189-adbf-4228-abb1-b6f69cb13d3e"
       snapshotId: "942b39b7-ca20-461e-8313-baf17148a8ac"
   
   ```

   

2. Clone Database

   ```
   [nkp@ken-rhel9 ndb]$ kubectl --kubeconfig=nkp-cluster01-kubeconfig.yaml -n ndb-operator apply -f db-postgres-clone.yaml 
   database.ndb.nutanix.com/nkp-postgres-clone created
   
   
   ```

   

3. NDB 查看狀態

   ![image-20241122180336412](https://kenkenny.synology.me:5543/images/2024/11/image-20241122180336412.png)

4. Log 狀態

   ![image-20241122180350421](https://kenkenny.synology.me:5543/images/2024/11/image-20241122180350421.png)

5. Clone 完成

   ![image-20241204152518617](https://kenkenny.synology.me:5543/images/2024/12/image-20241204152518617.png)

   ![image-20241204152555070](https://kenkenny.synology.me:5543/images/2024/12/image-20241204152555070.png)

6. NKP 可認得此Service

   ![image-20241204152841083](https://kenkenny.synology.me:5543/images/2024/12/image-20241204152841083.png)



#### Create MSSQL

1. Create mssql secret

   ```
   [nkp@ken-rhel9 ndb]$ vim db-mssql-secret.yaml
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: db-mssql-secret
   type: Opaque
   stringData:
     password: P@ssw0rd123
     ssh_public_key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCVN+q1hhDCFrbW4Gw+wPryGl2BQbLi9NNOv+hXLgI+NXJqhTT0XEL6Uun5WKd7ceNznfiCHee2AkQXm9nrvVyPpFU5zQ91j9zQbMsFylSDv7RALFyEaXO45u6bzKRvVnL4lsKygBcMwxSW8yIrv0qTeDl3rChfkxjhpWR+C7gbn6uIVINRoC/5L1xNGcThbF0CiHgOt5P03ZV4tsm6C7Z65Wj41vczaH460qqq//kaf2uNveOZhi4juIQMc0FcE4kvGwtqRJ5HIs8CRq1N/wX+uIIQEKsqtmQqSujcHOwosYPSNvVXvVgqvy9t0jfEU4Y3bcBkoF5zfq98OCG0978ewjb8yR909bQM2s2QRoQML8Gb6Be+EV7+xtUcqZOpAkyZOmWXN8yoL5EMcelgDZoyK3yH9FgKuz1sbB0f+4b1B8LM/SULTq8Lv7ww6ps3EIvKDwbH5rMQIxf3IEIuk7qvjyQlAZiE2nxiU88fcPB54WUoQBNd3uUjr0YVt1i5GM0= nkp@ken-rhel9.nutanixlab.local
     
   [nkp@ken-rhel9 ndb]$ kubectl apply -f db-mssql-secret.yaml -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   secret/db-mssql-secret created
   ```

   

2. Create mssql manifests

   ```
   [nkp@ken-rhel9 ndb]$ vim db-mssql-deploy.yaml
   apiVersion: ndb.nutanix.com/v1alpha1
   kind: Database
   metadata:
     name: netfos-nkp-mssql
   spec:
     ndbRef: ndb
     isClone: false
     databaseInstance:
       clusterId: "b9ab1352-1673-4ce1-a9d8-174771f3dcbb"
       name: "netfos-nkp-mssql"
       description: This for testing NDB Operator in NKP Cluster
       databaseNames:
         - netfos_database
       credentialSecret: db-mssql-secret
       size: 100
       timezone: "UTC"
       type: mssql
       # You can specify any (or none) of these types of profiles: compute, software, network, dbParam
       # If not specified, the corresponding Out-of-Box (OOB) profile will be used wherever applicable
       # Name is case-sensitive. ID is the UUID of the profile. Profile should be in the "READY" state
       # "id" & "name" are optional. If none provided, OOB may be resolved to any profile of that type
       profiles:
         compute:
           id: "79b6fd91-4d3d-4a30-9e7b-180bc4265922"
           name: "SQL_Default_Compute"
         # A Software profile is a mandatory input for closed-source engines: SQL Server & Oracle
         software:
           name: "Win2022_SQL2019_NKP"
           id: "3e2cb21d-073d-4b06-9815-07ebc85746f9"
         network:
           id: "7248aa73-46d7-438f-bb42-0afb9fb4f996"
           name: "DEFAULT_OOB_SQLSERVER_NETWORK"
         dbParam:
           name: "DEFAULT_SQLSERVER_DATABASE_PARAMS"
           id: "530dc232-b664-4d01-b542-81b4c2befb7c"
         # Only applicable for MSSQL databases
         dbParamInstance:
           name: "DEFAULT_SQLSERVER_INSTANCE_PARAMS"
           id: "90aa7e49-0ade-4122-b3ea-ba7c6fa4854c"
       timeMachine:                        # Optional block, if removed the SLA defaults to NONE
         sla : "DEFAULT_OOB_BRASS_SLA"
         dailySnapshotTime:   "12:00:00"   # Time for daily snapshot in hh:mm:ss format
         snapshotsPerDay:     1            # Number of snapshots per day
         logCatchUpFrequency: 30           # Frequency (in minutes)
         weeklySnapshotDay:   "MONDAY"  # Day of the week for weekly snapshot
         monthlySnapshotDay:  11           # Day of the month for monthly snapshot
       additionalArguments:
         sql_user_name: "nkpuser"
         authentication_mode: "mixed" 
         sql_user_password: "P@ssw0rd123"
   ```

3. Create MSSQL 

   ```
   [nkp@ken-rhel9 ndb]$ kubectl apply -f db-mssql-deploy.yaml -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   database.ndb.nutanix.com/netfos-nkp-mssql created
   
   看logs
   [nkp@ken-rhel9 ndb]$ kubectl logs -f netfos-ndb-operator-controller-manager-5c77798ddf-7xz49 -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   
   2024-11-12T00:28:18Z	INFO	Database CR Status: {"ipAddress":"172.16.90.163","id":"2582b1e6-f6c2-4eca-85f0-da5837845bff","status":"READY","dbServerId":"649e5d21-0c2b-4
   139-9a14-a617473ebfd3","type":"mssql","creationOperationId":"73c5a0d6-d46d-47b2-9e5f-ec972158efd3","deregistrationOperationId":""}	{"controller": "database", "control
   lerGroup": "ndb.nutanix.com", "controllerKind": "Database", "Database": {"name":"netfos-nkp-mssql","namespace":"ndb-operator"}, "namespace": "ndb-operator", "name": "netfo
   s-nkp-mssql", "reconcileID": "89bae2ea-bdad-40c8-b9cb-137f56a8f99e"}
   ```

4. NDB 查看進度

   ![image-20241111135100239](https://kenkenny.synology.me:5543/images/2024/11/image-20241111135100239.png)

5. 部署完成

   ![image-20241111152326244](https://kenkenny.synology.me:5543/images/2024/11/image-20241111152326244.png)

   ![image-20241111154004240](https://kenkenny.synology.me:5543/images/2024/11/image-20241111154004240.png)

6. 從 NKP Cluster 可以看到此 Database 的 Service

   ```
   [nkp@ken-rhel9 ndb]$ kubectl get svc -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   NAME                                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   netfos-ndb-operator-controller-manager-metrics-service   ClusterIP   10.107.170.244   <none>        8443/TCP   5h2m
   netfos-ndb-operator-webhook-service                      ClusterIP   10.109.133.195   <none>        443/TCP    5h2m
   netfos-nkp-mssql-svc                                     ClusterIP   10.100.14.241    <none>        80/TCP     47m
   
   [nkp@ken-rhel9 ndb]$ kubectl describe svc netfos-nkp-mssql-svc -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
   Name:              netfos-nkp-mssql-svc
   Namespace:         ndb-operator
   Labels:            <none>
   Annotations:       <none>
   Selector:          <none>
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                10.100.14.241
   IPs:               10.100.14.241
   Port:              <unset>  80/TCP
   TargetPort:        1433/TCP
   Endpoints:         172.16.90.163:1433
   Session Affinity:  None
   Events:            <none>
   ```

   ![image-20241111152436685](https://kenkenny.synology.me:5543/images/2024/11/image-20241111152436685.png)

7. 測試連線

   ```
   [nkp@ken-rhel9 ndb]$ nc -zv 172.16.90.163 1433
   Ncat: Version 7.92 ( https://nmap.org/ncat )
   Ncat: Connected to 172.16.90.163:1433.
   Ncat: 0 bytes sent, 0 bytes received in 0.25 seconds.
   ```

   

8. 安裝sqlcmd 來確認連線

   ```
   [nkp@ken-rhel9 ndb]$ curl https://packages.microsoft.com/config/rhel/9/prod.repo | sudo tee /etc/yum.repos.d/mssql-release.repo
   
   [nkp@ken-rhel9 ndb]$ sudo yum install -y mssql-tools18 unixODBC-devel
   
   Installed products updated.
   
   Installed:
     libtool-ltdl-2.4.6-45.el9.x86_64   msodbcsql18-18.4.1.1-1.x86_64   mssql-tools18-18.4.1.1-1.x86_64   unixODBC-2.3.11-1.rh.x86_64   unixODBC-devel-2.3.11-1.rh.x86_64  
   
   Complete!
   
   [nkp@ken-rhel9 ndb]$ echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bash_profile
   [nkp@ken-rhel9 ndb]$ source ~/.bash_profile
   ```

   

9. sqlcmd連線測試

   ```
   [nkp@ken-rhel9 ~]$ sqlcmd -S 172.16.90.163  -U nkpuser -p -Q "SELECT @@VERSION" -C
   Password: 
                                                                                                                                                                                                                                                                                                               
   ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   Microsoft SQL Server 2022 (RTM) - 16.0.1000.6 (X64) 
   	Oct  8 2022 05:58:25 
   	Copyright (C) 2022 Microsoft Corporation
   	Enterprise Edition (64-bit) on Windows Server 2019 Standard 10.0 <X64> (Build 17763: ) (Hypervisor)
                                                                                    
   
   (1 rows affected)
   
   Network packet size (bytes): 4096
   1 xact[s]:
   Clock Time (ms.): total         1  avg   1.0 (1000.0 xacts per sec.)
   
   
   [nkp@ken-rhel9 ~]$ sqlcmd -S 172.16.90.163  -U nkpuser -p -Q "SELECT name from sys.databases" -C
   Password: 
   name                                                                                                                            
   --------------------------------------------------------------------------------------------------------------------------------
   master                                                                                                                          
   tempdb                                                                                                                          
   model                                                                                                                           
   msdb                                                                                                                            
   netfos_database                                                                                                                 
   
   (5 rows affected)
   
   Network packet size (bytes): 4096
   1 xact[s]:
   Clock Time (ms.): total        37  avg   37.0 (27.0 xacts per sec.)
   
   ```

   

10. Create Database

    ```
    [nkp@ken-rhel9 ~]$ sqlcmd -S 172.16.90.163  -U nkpuser -p -C -Q "CREATE DATABASE nkpdev"
    Password: 
    
    Network packet size (bytes): 4096
    1 xact[s]:
    Clock Time (ms.): total       379  avg   379.0 (2.6 xacts per sec.)
    
    [nkp@ken-rhel9 ~]$ sqlcmd -S 172.16.90.163  -U nkpuser -p -Q "SELECT name from sys.databases" -C
    Password: 
    name                                                                                                                            
    --------------------------------------------------------------------------------------------------------------------------------
    master                                                                                                                          
    tempdb                                                                                                                          
    model                                                                                                                           
    msdb                                                                                                                            
    netfos_database                                                                                                                 
    nkpdev                                                                                                                          
    
    (6 rows affected)
    
    Network packet size (bytes): 4096
    1 xact[s]:
    Clock Time (ms.): total         3  avg   3.0 (333.3 xacts per sec.)
    ```

    

11. Delete Database VM

    ```
    [nkp@ken-rhel9 ndb]$ kubectl delete -f db-mssql-deploy.yaml -n ndb-operator --kubeconfig=nkp-cluster01-kubeconfig.yaml
    database.ndb.nutanix.com "netfos-nkp-mssql" deleted
    ```

    ![image-20241111173936985](https://kenkenny.synology.me:5543/images/2024/11/image-20241111173936985.png)

    ![image-20241111174412927](https://kenkenny.synology.me:5543/images/2024/11/image-20241111174412927.png)

    



## Workspace & Project



建立Project 設定權限



![image-20241209163958365](https://kenkenny.synology.me:5543/images/2024/12/image-20241209163958365.png)

![image-20241209164310858](https://kenkenny.synology.me:5543/images/2024/12/image-20241209164310858.png)

![image-20241209164041292](https://kenkenny.synology.me:5543/images/2024/12/image-20241209164041292.png)

![image-20241209164123945](https://kenkenny.synology.me:5543/images/2024/12/image-20241209164123945.png)

![image-20241209164138754](https://kenkenny.synology.me:5543/images/2024/12/image-20241209164138754.png)

![image-20241209164026129](https://kenkenny.synology.me:5543/images/2024/12/image-20241209164026129.png)



```
[nkp@ken-rhel9 ~]$ kubectl get workspace --kubeconfig=/home/nkp/.kube/adminconfig
NAME                  DISPLAY NAME                   WORKSPACE NAMESPACE           AGE
default-workspace     Default Workspace              kommander-default-workspace   69d
kommander-workspace   Management Cluster Workspace   kommander                     69d
nkp-workload-lab      nkp-workload-lab               nkp-workload-lab              40d

export WORKSPACE_NAME=<name_target_workspace>
export WORKSPACE_NAME=nkp-workload-lab

echo https://$(kubectl get kommandercluster -n kommander host-cluster -o jsonpath='{ .status.ingress.address }')/token/landing/${WORKSPACE_NAME}

[nkp@ken-rhel9 ~]$ export WORKSPACE_NAME=nkp-workload-lab
[nkp@ken-rhel9 ~]$ echo https://$(kubectl --kubeconfig=/home/nkp/.kube/adminconfig get kommandercluster -n kommander host-cluster -o jsonpath='{ .status.ingress.address }')/token/landing/${WORKSPACE_NAME}

https://172.16.90.209/token/landing/nkp-workload-lab

```

![image-20241209162859507](https://kenkenny.synology.me:5543/images/2024/12/image-20241209162859507.png)

![image-20241209163415226](https://kenkenny.synology.me:5543/images/2024/12/image-20241209163415226.png)

![image-20241209163426947](https://kenkenny.synology.me:5543/images/2024/12/image-20241209163426947.png)

![image-20241209163439110](https://kenkenny.synology.me:5543/images/2024/12/image-20241209163439110.png)

```
[nkpuser@ken-rhel9 ~]$ cat ~/.kube/config 
apiVersion: v1
clusters:
- cluster:
    certificate-authority: certs/vpnuser12-172.16.90.211/k8s-ca.crt
    server: https://172.16.90.211/dkp/api-server
  name: vpnuser12-172.16.90.211
contexts:
- context:
    cluster: vpnuser12-172.16.90.211
    user: vpnuser12-172.16.90.211
  name: vpnuser12-172.16.90.211
current-context: vpnuser12-172.16.90.211
kind: Config
preferences: {}
users:
- name: vpnuser12-172.16.90.211
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IjRjOWY5NGZjM2I0ZThmYzJmN2YyN2I5NmUwMzc2YzIyNGViMzVhNjMifQ.eyJhdF9oYXNoIjoidERPS2Z2cG5fRXY0T2xfVFpLSDBNUSIsImF1ZCI6WyJkZXgtY29udHJvbGxlci1kZXh0ZmEtY2xpZW50LW5rcC1jbHVzdGVyMDEtMjZrYmgiLCJkZXgtY29udHJvbGxlci1kZXgtZGthLWNsaWVudC1rdWJlLWFwaXNlcnZlciJdLCJhenAiOiJkZXgtY29udHJvbGxlci1kZXgtZGthLWNsaWVudC1rdWJlLWFwaXNlcnZlciIsImNfaGFzaCI6Ik5Rb0haSUFrekNJblpZcEJCdXRUdlEiLCJlbWFpbCI6InZwbnVzZXIxMkBudXRhbml4bGFiLmxvY2FsIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImV4cCI6MTczMzgxOTUwMCwiZmVkZXJhdGVkX2NsYWltcyI6bnVsbCwiZ3JvdXBzIjpbIk5LUFVzZXJzIl0sImlhdCI6MTczMzczMzEwMCwiaXNzIjoiaHR0cHM6Ly8xNzIuMTYuOTAuMjA5L2RleCIsIm5hbWUiOiJ2cG51c2VyMTIiLCJub25jZSI6IiIsInByZWZlcnJlZF91c2VybmFtZSI6IiIsInN1YiI6IkNrQkRUajEyY0c1MWMyVnlNVElzVDFVOVEzVnpkRzl0WlhKekxFOVZQVTVsZEdadmMxUmhhWEJsYVN4RVF6MXVkWFJoYm1sNGJHRmlMRVJEUFd4dlkyRnNFalZrWlhndFkyOXVkSEp2Ykd4bGNsOXJiMjF0WVc1a1pYSmZiR1JoY0MxcFpHVnVkR2wwZVMxd2NtOTJhV1JsY2kxb1p6UmllQSJ9.a7NGtbrz2cHHxXYSIvXjQwUXQubOPQWCHVjPpFrREK0uuW8B7__t0Nc-v10uhRsHNCPIAmHwo-bnGkyjk8fGwcGDRFg9iDTTKy35Zsx_nc-OuHblKOG8tU42aUiGKG5fftd0vfHgzGaBolQ-dKbk45hn_h7y9l60eYYU5yUCIdI7kX5OgNV2eWksm3qhHathkF1vzC022Yo6Q2QmZceMSFQp9ckn9K033EHAIsSK3CEuyZcTqejA23XItE6Sfh3UOdwjzqRi7H810XSaQVnEMrctgIDnpbSUZMpLE2UCt0jAFXB841Ce7oNQvfzteCcHDvd9wiMfUY1gHIR5mbZL7g


[nkpuser@ken-rhel9 ~]$ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "vpnuser12@nutanixlab.local" cannot list resource "nodes" in API group "" at the cluster scope

[nkpuser@ken-rhel9 ~]$ kubectl get pods -n dev-team
No resources found in dev-team namespace.
[nkpuser@ken-rhel9 ~]$ kubectl get ns
NAME                     STATUS   AGE
cert-manager             Active   40d
default                  Active   40d
dev-team-npbwm           Active   128m
kommander-flux           Active   40d
kube-federation-system   Active   40d
kube-node-lease          Active   40d
kube-public              Active   40d
kube-system              Active   40d
metallb-system           Active   40d
ndb-operator             Active   28d
nkp-workload-lab         Active   40d
node-feature-discovery   Active   40d
ntnx-system              Active   40d

# 在別的namespace無法新增pod
[nkpuser@ken-rhel9 ~]$ kubectl run nginx --image=nginx
Error from server (Forbidden): pods is forbidden: User "vpnuser12@nutanixlab.local" cannot create resource "pods" in API group "" in the namespace "default"

# 可在指定的namespace新增pod
[nkpuser@ken-rhel9 ~]$ kubectl run nginx --image=nginx -n dev-team-npbwm
pod/nginx created

[nkpuser@ken-rhel9 ~]$ kubectl expose po nginx --type=NodePort --name=nginx-svc --port=80 -n dev-team-npbwm
service/nginx-svc exposed

# 確認pod與服務並連線
[nkpuser@ken-rhel9 ~]$ kubectl get all -n dev-team-npbwm
NAME        READY   STATUS    RESTARTS   AGE
pod/nginx   1/1     Running   0          44s

NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/nginx-svc   NodePort   10.98.244.223   <none>        80:31403/TCP   20s

[nkpuser@ken-rhel9 ~]$ kubectl describe service/nginx-svc -n dev-team-npbwm
Name:                     nginx-svc
Namespace:                dev-team-npbwm
Labels:                   run=nginx
Annotations:              <none>
Selector:                 run=nginx
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.98.244.223
IPs:                      10.98.244.223
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31403/TCP
Endpoints:                192.168.1.163:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

![image-20241209171829715](https://kenkenny.synology.me:5543/images/2024/12/image-20241209171829715.png)



## Metallb-system

新增 loadbalancer ip 到 ip pool 裡面

```
[nkp@ken-rhel9 ~]$ kubectl get ipaddresspools -n metallb-system
NAME      AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
metallb   true          false             ["172.16.90.209-172.16.90.209"]

[nkp@ken-rhel9 ~]$ kubectl edit ipaddresspools -n metallb-system
ipaddresspool.metallb.io/metallb edited

[nkp@ken-rhel9 ~]$ kubectl describe ipaddresspools -n metallb-system
Name:         metallb
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>
API Version:  metallb.io/v1beta1
Kind:         IPAddressPool
Metadata:
  Creation Timestamp:  2024-10-01T03:37:31Z
  Generation:          2
  Resource Version:    318510970
  UID:                 3cba6b6f-06c6-4002-a039-12774eedda72
Spec:
  Addresses:
    172.16.90.209-172.16.90.211
  Auto Assign:       true
  Avoid Buggy I Ps:  false
Events:              <none>
```

![image-20250220095215314](https://kenkenny.synology.me:5543/images/2025/02/image-20250220095215314.png)

```
[nkp@ken-rhel9 ~]$ kubectl get svc -n ken-ns
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)             AGE
carts          ClusterIP      10.99.139.74     <none>          80/TCP              18h
carts-db       ClusterIP      10.99.20.222     <none>          27017/TCP           18h
catalogue      ClusterIP      10.106.141.207   <none>          80/TCP              18h
catalogue-db   ClusterIP      10.102.80.220    <none>          3306/TCP            18h
front-end      LoadBalancer   10.103.0.112     172.16.90.211   80:30239/TCP        18h
orders         ClusterIP      10.106.57.160    <none>          80/TCP              18h
orders-db      ClusterIP      10.110.189.178   <none>          27017/TCP           18h
payment        ClusterIP      10.102.165.251   <none>          80/TCP              18h
queue-master   ClusterIP      10.96.248.113    <none>          80/TCP              18h
rabbitmq       ClusterIP      10.109.56.231    <none>          5672/TCP,9090/TCP   18h
session-db     ClusterIP      10.98.29.122     <none>          6379/TCP            18h
shipping       ClusterIP      10.97.155.46     <none>          80/TCP              18h
user           ClusterIP      10.99.13.238     <none>          80/TCP              18h
user-db        ClusterIP      10.105.150.252   <none>          27017/TCP           18h


```

![image-20250220095316530](https://kenkenny.synology.me:5543/images/2025/02/image-20250220095316530.png)
