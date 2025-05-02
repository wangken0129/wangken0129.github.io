---
title: Nutanix_NKP-2.14_AHV_Object
description: Nutanix_NKP-2.14_AHV_Object
slug: Nutanix_NKP-2.14_AHV_Object
date: 2025-05-02T09:44:33+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - NKP
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Nutanix NKP 2.14 on AHV with Nutanix Object

此 LAB 會透過 airgapped 方式建立 NKP v2.14 on AHV 

並且將Velero、Grafana Loki、Harbor 會用到的儲存從 rook ceph 改為 Nutanix Object Storage

https://portal.nutanix.com/page/documents/kbs/details?targetId=kA0VO0000001mU30AI



## 安裝部分大致步驟

1. 確認安裝環境

2. 連線 bootstrap host (Bastion Host)

3. 確認 Container Service

4. 確認 Kubernetes CLI、NKP CLI

5. 設定 Local Registry (air-gapped)

6. 推送 NKP Image 到 Local Registry (air-gapped)

7. 建立 machine image (optional)

8. 建立 deploy.yaml

9. 建立 management cluster (Nutanix環境預設裝起來就是Starter授權，會是最小安裝)

   

## 環境資訊

### 版本

Prism Central: pc.2024.1.0.1

AOS: 6.8.1

Nutanix Files: 5.0.0.2

Nutanix Object: 5.0

NKP: 2.14

Kubernetes: 1.31.4

OS Image: Ubuntu-22.04

KIB: v2.18.0

Bastion OS Version: RHEL 9.3

Bastion docker Version:  28.0.1

Local Registry Mirror (Harbor): v2.12.2

### IP 與名稱

Prism Central: https://172.16.90.75:9440

UUID: acb94940-7f03-416e-9dcd-6b442d3fe081

Prism Element (NX1365G6PE): https://172.16.90.74:9440/

UUID: 00061972-aeba-fee2-0000-000000028e95

bastion ip : 172.16.90.206

Storage Container: Kubernetes-Container

Subnet: Nutanix_Lab_Public_1002_IPAM

kubeVIP: 192.168.102.91

kube Ingress ip Range:  192.168.102.92-192.168.102.94

Subnet Mask: 255.255.255.0

Gateway IP Address: 192.168.102.254



DR Prism Central: https://172.16.90.72:9440

UUID: 12ecbf3b-352d-4369-a7bb-ef71b7262f47

DR Prism Element (NX3500PE): https://172.16.90.71:9440

UUID: 00061f20-3fc1-adc4-7dda-002590c84b9a

### 環境變數

env.sh

```
export MGMT_CLUSTER_NAME="netfos-nkp"
export MGMT_CP_ENDPOINT="192.168.102.91"
export MGMT_LB_IP_RANGE="192.168.102.92-192.168.102.94"
export PC_ENDPOINT="172.16.90.75"
export PE_CLUSTER_NAME="NX1365G6PE"
export SUBNET="Nutanix_Lab_Public_1002_IPAM"
export VM_IMAGE="nkp-ubuntu-22.04-1.31.4-20250409073323"
export STORAGE_CONTAINER="Kubernetes-Container"
export NUTANIX_USER="k8sadmin"
export NUTANIX_PASSWORD="P@55w.rd"
export CSI_HYPERVISOR_ATTACHED="true"
export CSI_FILESYSTEM="ext4"
export SSH_USER="nkp"
export SSH_PUBLIC_KEY="/home/nkp/.ssh/id_rsa.pub"
export BOOTSTRAP_HOST_IP="172.16.90.206"
export REGISTRY_URL="http://172.16.90.206/nkp-2.14"
export REGISTRY_URL_APP="http://172.16.90.206/app"
export REGISTRY_USERNAME="admin"
export REGISTRY_PASSWORD="Harbor12345"
export BASTION_HOST="172.16.90.206"
export BASTION_SSH_USER="nkp"
export BASTION_KEY="/home/nkp/.ssh/id_rsa"
export BASTION_PORT="22"
export BASE_IMAGE="ubuntu-22.04-server-cloudimg-amd64.img"
export OS_BUNDLE_DIR="/home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/kib/artifacts"
export OS="ubuntu-22.04"

# source env.sh
```

### Nutanix Object Storage

Object Storage 建置過程省略，詳細可參考原廠文件。

#### Create Object Bucket

從現有 object store ntnxobject  建立 Bucket

![image-20250410133149755](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133149755.png)

![image-20250410133338241](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133338241.png)

nkp-loki

![image-20250410133431833](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133431833.png)

![image-20250410133612900](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133612900.png)

nkp-velero

![image-20250410133501235](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133501235.png)

![image-20250410133520935](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133520935.png)



#### Create Access Key

![image-20250410133714308](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133714308.png)

![image-20250410133945152](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133945152.png)

Tag 建議不要加，Velero 會有異常

![image-20250410133957175](https://kenkenny.synology.me:5543/images/2025/04/image-20250410133957175.png)

![image-20250410134034989](https://kenkenny.synology.me:5543/images/2025/04/image-20250410134034989.png)

```
Username: nkpsa@nutanixlab.local
Access Key: Grw2SVIH6ZxSMOkjSI8pQBMtzqhjZ9fY
Secret Key: FSAWoandsVhTcGMvwHSK_Wd-Ntwy2ZP0
Display Name: nkpsa
Tag: nkp

---

Username: nkpsa@nutanixlab.local
Access Key: _y1gdwax1EwopQXQo1EXM1egoBYijgF0
Secret Key: 5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5
Display Name: nkpsa
```

把 nkpsa user 權限加入到兩個 bucket

![image-20250410134223676](https://kenkenny.synology.me:5543/images/2025/04/image-20250410134223676.png)

![image-20250410134250732](https://kenkenny.synology.me:5543/images/2025/04/image-20250410134250732.png)

![image-20250410134321816](https://kenkenny.synology.me:5543/images/2025/04/image-20250410134321816.png)

測試連線

```
vi ~/.aws/credentials
[default]
aws_access_key_id=Grw2SVIH6ZxSMOkjSI8pQBMtzqhjZ9fY
aws_secret_access_key=FSAWoandsVhTcGMvwHSK_Wd-Ntwy2ZP0
---
aws s3 ls --endpoint-url=https://ntnxobject.nutanixlab.local --no-verify-ssl

aws s3 ls --endpoint-url=http://ntnxobject.nutanixlab.local
2025-04-14 17:31:53 ntnx-object-nkp
2025-04-10 13:34:34 nkp-loki
2025-04-10 13:35:00 nkp-velero

aws s3 ls s3://nkp-loki --endpoint-url=http://ntnxobject.nutanixlab.local


aws s3api get-bucket-location --bucket nkp-loki --endpoint-url=http://ntnxobject.nutanixlab.local
{
    "LocationConstraint": null
}

```

![image-20250410142215105](https://kenkenny.synology.me:5543/images/2025/04/image-20250410142215105.png)



## Bastion Host

### 軟體下載

nkp-airgapped, nkp cli , kib 從 Portal 上直接用 copy link 方式下載

```
[nkp@ken-rhel9 nkp-download]$ curl -o nkp-air-gapped-bundle_v2.14.0_linux_amd64.tar.gz
"https://download.nutanix.com/downloads/nkp/v2.14.0/nkp-air-gapped-bundle_v2.14.0_linux_amd64.tar.gz?Expires=1744200896&Key-Pair-Id=APKAJxxxmNHXR3Mg__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 15.4G  100 15.4G    0     0  14.9M      0  0:17:39  0:17:39 --:--:-- 18.1M

[nkp@ken-rhel9 nkp-download]$ curl -o nkp_v2.14.0_linux_amd64.tar.gz
"https://download.nutanix.com/downloads/nkp/v2.14.0/nkp_v2.14.0_linux_amd64.tar.gz?Expires=1744201963&Key-Pair-Id=APKAJxxxNpDGyQbUxlgGAKmQ__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 90.5M  100 90.5M    0     0  12.9M      0  0:00:06  0:00:06 --:--:-- 15.6M

[nkp@ken-rhel9 nkp-download]$ curl -o konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz 
https://download.nutanix.com/downloads/nkp/v2.14.0/konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz?Expires=1744202054&Key-Pair-Id=APKAJTTxxxQSJtjHRQ__"

[nkp@ken-rhel9 nkp-download]$ ll
total 16674272
-rw-r--r-- 1 nkp nkp   346426397 Apr  9 10:34 konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz
-rw-r--r-- 1 nkp nkp 16633123312 Apr  9 10:33 nkp-air-gapped-bundle_v2.14.0_linux_amd64.tar.gz
-rw-r--r-- 1 nkp nkp    94896342 Apr  9 10:33 nkp_v2.14.0_linux_amd64.tar.gz
```

### 軟體安裝

```
[nkp@nkp-bastion ~]$ sudo yum install docker yum-utils bzip2 wget tar -y
[nkp@nkp-bastion ~]$ sudo yum install git-all -y
[nkp@nkp-bastion ~]$ wget https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz
[nkp@nkp-bastion ~]$ sudo cp linux-amd64/helm /usr/local/bin

[nkp@nkp-bastion ~]$ mkdir -p nkptools/nkp
[nkp@nkp-bastion ~]$ mkdir -p nkptools/kib
[nkp@nkp-bastion ~]$ mkdir -p nkptools/nkp-airgap
[nkp@ken-rhel9 nkp-download]$ tar xvf nkp_v2.14.0_linux_amd64.tar.gz -C ../nkptools/nkp
[nkp@ken-rhel9 nkp-download]$ tar xvf konvoy-image-bundle-v2.22.2_linux_amd64.tar.gz -C ../nkptools/kib
[nkp@ken-rhel9 nkp-download]$ tar xvf nkp-air-gapped-bundle_v2.14.0_linux_amd64.tar.gz -C ../nkptools/nkp-airgap

[nkp@nkp-bastion ~]$ sudo cp nkptools/nkp/nkp /usr/local/bin/
[nkp@nkp-bastion ~]$ sudo cp nkptools/kib/konvoy-image /usr/local/bin/
[nkp@nkp-bastion ~]$ sudo cp nkptools/nkp-airgap/nkp-v2.14.0/kubectl /usr/local/bin/

[nkp@ken-rhel9 ~]$ nkp version
diagnose: v0.10.1
imagebuilder: v0.22.3
kommander: v2.14.0
konvoy: v2.14.0
mindthegap: v1.16.0
nkp: v2.14.0

[nkp@ken-rhel9 ~]$ kubectl version
Client Version: v1.31.4
Kustomize Version: v5.4.2
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

### Harbor

詳細安裝步驟

https://goharbor.io/docs/1.10/install-config/installation-prereqs/

1. 下載並解壓縮 Harbor offline installer

https://github.com/goharbor/harbor/releases

```
[root@ken-rhel9 harbor]# ll
total 632400
drwxr-xr-x. 3 nkp nkp       181 Apr  9 10:20 harbor
-rw-r--r--. 1 nkp nkp 647577270 Jan 17 01:05 harbor-offline-installer-v2.12.2.tgz

[root@ken-rhel9 harbor]# cd harbor/
[root@ken-rhel9 harbor]# ll
total 636520
drwxr-xr-x. 3 root root        20 Mar 12 10:10 common
-rw-r--r--. 1 nkp  nkp       3646 Jan 16 22:10 common.sh
-rw-r--r--. 1 root root      5828 Mar 12 10:46 docker-compose.yml
-rw-r--r--. 1 nkp  nkp  651727378 Jan 16 22:11 harbor.v2.12.2.tar.gz
-rw-r--r--. 1 nkp  nkp      14295 Mar 12 10:04 harbor.yml
-rw-r--r--. 1 nkp  nkp      14288 Jan 16 22:10 harbor.yml.tmpl
-rwxr-xr-x. 1 nkp  nkp       1975 Jan 16 22:10 install.sh
-rw-r--r--. 1 nkp  nkp      11347 Jan 16 22:10 LICENSE
-rwxr-xr-x. 1 nkp  nkp       2211 Jan 16 22:10 prepare
```

2. 修改 harbor.yml

```
hostname: 172.16.90.206
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80
harbor_admin_password: Harbor12345
```

3. 安裝 Harbor

   ```
   sudo ./install.sh
   ```

4. 調整為 HTTP 連線（建議還是用 HTTPS）

   ```
   # 修改 docker daemon
   $ vi /etc/docker/daemon.json
   {
   "insecure-registries" : ["myregistrydomain.com:5000", "0.0.0.0"]
   }
   
   # 找到 docker service
   $ find / -name docker.service -type f
   /usr/lib/systemd/system/docker.service
   
   # 修改 docker.service配置文件 https 改為 http
   $ vim /usr/lib/systemd/system/docker.service
   [Unit]
   Documentation=http://docs.docker.io 
   ...
   [Service]
   Type=notify
   ...
   ```

5. 重啟 Harbor

   ```
   $ docker-compose down -v
   $ systemctl daemon-reload
   $ systemctl restart docker
   $ docker-compose up -d
   ```

6. 登入測試

   ![image-20250409113115658](https://kenkenny.synology.me:5543/images/2025/04/image-20250409113115658.png)

   ![image-20250409113137161](https://kenkenny.synology.me:5543/images/2025/04/image-20250409113137161.png)

7. 新增專案 For nkp-2.14

   ![image-20250409113247071](https://kenkenny.synology.me:5543/images/2025/04/image-20250409113247071.png)

   ![image-20250409113315027](https://kenkenny.synology.me:5543/images/2025/04/image-20250409113315027.png)




## Seed Image

推送 Container Image 到 Harbor 上 172.16.90.206/nkp-2.14

 切換成 root 帳號，要先用 docker login 172.16.90.206/nkp-2.14 ，否則 push 的時候必須指定帳號密碼 
 --to-registry-username= , --to-registry-password= , --to-registry-ca-cert-file= 

```
# Kommander
nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/kommander-image-bundle-v2.14.0.tar --to-registry=http://172.16.90.206/nkp-2.14 --to-registry-insecure-skip-tls-verify

# Konvoy-image-bundle
nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/konvoy-image-bundle-v2.14.0.tar --to-registry=http://172.16.90.206/nkp-2.14 --to-registry-insecure-skip-tls-verify

# Catalog-application
nkp push bundle --bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/container-images/nkp-catalog-applications-image-bundle-v2.14.0.tar --to-registry=http://172.16.90.206/nkp-2.14 --to-registry-insecure-skip-tls-verify

```

![image-20250409115229036](https://kenkenny.synology.me:5543/images/2025/04/image-20250409115229036.png)

![image-20250409115432811](https://kenkenny.synology.me:5543/images/2025/04/image-20250409115432811.png)

![image-20250409115452047](https://kenkenny.synology.me:5543/images/2025/04/image-20250409115452047.png)

![image-20250409115505218](https://kenkenny.synology.me:5543/images/2025/04/image-20250409115505218.png)

![image-20250409115530074](https://kenkenny.synology.me:5543/images/2025/04/image-20250409115530074.png)

Bastion Load nkp bootstrap image、image builder image

```
docker load -i /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/konvoy-bootstrap-image-v2.14.0.tar

docker load -i /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/nkp-image-builder-image-v0.22.3.tar
```

![image-20250409120337540](https://kenkenny.synology.me:5543/images/2025/04/image-20250409120337540.png)

![image-20250409120349384](https://kenkenny.synology.me:5543/images/2025/04/image-20250409120349384.png)

## Create Machine Image

先從 https://cloud-images.ubuntu.com/releases/releases/22.04/release/  下載 ubuntu-22.04-server-cloudimg-amd64.img  

並上傳到 Prism Central

![image-20250409133423208](https://kenkenny.synology.me:5543/images/2025/04/image-20250409133423208.png)

![image-20250409133616066](https://kenkenny.synology.me:5543/images/2025/04/image-20250409133616066.png)

### Create OS Bundle

```
export OS_BUNDLE_DIR=kib/artifacts
export OS=ubuntu-22.04

nkp create package-bundle --artifacts-directory ${OS_BUNDLE_DIR} ${OS}
```

![image-20250409154303111](https://kenkenny.synology.me:5543/images/2025/04/image-20250409154303111.png)

![image-20250409154333193](https://kenkenny.synology.me:5543/images/2025/04/image-20250409154333193.png)

### Create Machine Image

```
# normal（建立的 Machine 可以連線到 create image 的主機）
export NUTANIX_USER=<user>
export NUTANIX_PASSWORD=<password>
export BASE_IMAGE=ubuntu-22.04-server-cloudimg-amd64.img
export SUBNET=<VLAN Subnet>
export PC_ENDPOINT=<PC Hostname>
export PE_CLUSTER_NAME=<PE Cluster name>

nkp create image nutanix ubuntu-22.04 --endpoint ${PC_ENDPOINT} --cluster ${PE_CLUSTER_NAME} --subnet ${SUBNET} --source-image ${BASE_IMAGE} --artifacts-directory ${OS_BUNDLE_DIR} --insecure=true

# restricted network (透過 bastion 去安裝相關的套件)
# bastion 的 sshd_config 要設定 PubkeyAuthentication yes ，然後要將public key 放到 .ssh/authorized_keys
export NUTANIX_USER=admin
export NUTANIX_PASSWORD=<password>
export BASE_IMAGE=ubuntu-22.04-server-cloudimg-amd64.img
export SUBNET=<VLAN Subnet>
export PC_ENDPOINT=<PC Hostname>
export PE_CLUSTER_NAME=<PE Cluster name>
export BASTION_HOST=<Reachable Bastion host IP>
export BASTION_SSH_USER=<SSH User to connect to the Bastion>
export BASTION_KEY=<SSH Private Key to connect to the Bastion>
export BASTION_PORT=22

nkp create image nutanix ${OS} --endpoint ${PC_ENDPOINT} --cluster ${PE_CLUSTER_NAME} --subnet ${SUBNET} --source-image ${BASE_IMAGE} --artifacts-directory=${OS_BUNDLE_DIR} --bastion-host=${BASTION_HOST} --bastion-private-key-file=${BASTION_KEY} --bastion-username=${BASTION_SSH_USER} --bastion-port ${BASTION_PORT} --insecure=true
```

![image-20250409154349915](https://kenkenny.synology.me:5543/images/2025/04/image-20250409154349915.png)

![image-20250409154403690](https://kenkenny.synology.me:5543/images/2025/04/image-20250409154403690.png)

![image-20250409154421108](https://kenkenny.synology.me:5543/images/2025/04/image-20250409154421108.png)

## Create&Modify Cluster

用 NKP CLI 方式建立 NKP 叢集，有客製化部分所以將步驟拆解，也可以直接使用 nkp create cluster 後面帶入參數來建立

### Create Cluster

#### Dry run

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
--worker-memory=24 \
--worker-replicas=3 \
--csi-storage-container=$STORAGE_CONTAINER \
--kubernetes-service-load-balancer-ip-range=$MGMT_LB_IP_RANGE \
--endpoint="https://$PC_ENDPOINT:9440" \
--insecure=true \
--registry-mirror-url=$REGISTRY_URL \
--registry-mirror-username=${REGISTRY_USERNAME} \
--registry-mirror-password=${REGISTRY_PASSWORD} \
--registry-url=$REGISTRY_URL_APP \
--registry-username=$REGISTRY_USERNAME \
--registry-password=$REGISTRY_PASSWORD \
--ssh-username=$SSH_USER \
--ssh-public-key-file=$SSH_PUBLIC_KEY \
--dry-run \
--airgapped \
--self-managed \
--output=yaml > deploy-nkp-$MGMT_CLUSTER_NAME.yaml
```

#### Create bootstrap

```
nkp create bootstrap
```

#### Create Cluster

```
kubectl apply -f deploy-nkp-netfos-nkp-edited.yaml 

watch -n 5 nkp describe cluster --cluster-name $MGMT_CLUSTER_NAME

nkp get kubeconfig --cluster-name $MGMT_CLUSTER_NAME > $MGMT_CLUSTER_NAME.conf
chmod 600 $MGMT_CLUSTER_NAME.conf

kubectl get nodes --kubeconfig=$MGMT_CLUSTER_NAME.conf

"建立 capi-components"
nkp create capi-components --kubeconfig=$MGMT_CLUSTER_NAME.conf

 
"從 bootstratp 移轉 capi-resources"
nkp move capi-resources --to-kubeconfig $MGMT_CLUSTER_NAME.conf

nkp delete bootstrap

watch kubectl get pods -n kommander

nkp get dashboard 
```

#### License 匯入



#### Velero、Grafana Loki

##### Secret for object

```
Username: nkpsa@nutanixlab.local
Access Key: Grw2SVIH6ZxSMOkjSI8pQBMtzqhjZ9fY
Secret Key: FSAWoandsVhTcGMvwHSK_Wd-Ntwy2ZP0
Display Name: nkpsa
Tag: nkp
-- （有tag實測velero會有異常）
Username: nkpsa@nutanixlab.local
Access Key: _y1gdwax1EwopQXQo1EXM1egoBYijgF0
Secret Key: 5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5
Display Name: nkpsa
--

secret ( 需要base64,grafana loki 文件建議要加 -n )
Linux echo 指令預設會加上換行符號 \n，如果想要 echo 不加上換行符號的話可以加上 -n 的選項

echo -n _y1gdwax1EwopQXQo1EXM1egoBYijgF0 |base64
X3kxZ2R3YXgxRXdvcFFYUW8xRVhNMWVnb0JZaWpnRjA=

echo -n 5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5 |base64
NWo4eEdDSzE2LTFfM0hIcmJtbVhWcDM5ZTFDaTk2YjU=

echo -n Grw2SVIH6ZxSMOkjSI8pQBMtzqhjZ9fY|base64
R3J3MlNWSUg2WnhTTU9ralNJOHBRQk10enFoalo5Zlk=

cho -n FSAWoandsVhTcGMvwHSK_Wd-Ntwy2ZP0|base64
RlNBV29hbmRzVmhUY0dNdndIU0tfV2QtTnR3eTJaUDA=


echo -n "[default]
aws_access_key_id=_y1gdwax1EwopQXQo1EXM1egoBYijgF0
aws_secret_access_key=5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5" | base64 -w0
W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkPV95MWdkd2F4MUV3b3BRWFFvMUVYTTFlZ29CWWlqZ0YwCmF3c19zZWNyZXRfYWNjZXNzX2tleT01ajh4R0NLMTYtMV8zSEhyYm1tWFZwMzllMUNpOTZiNQ==

Secrets yaml 檔，名稱為 dkp-loki、dkp-velero 第一個 secret 不能改名字，後面可以再加其他的
---
apiVersion: v1
kind: Secret
metadata:
  name: dkp-loki
  namespace: kommander
data:
  AWS_ACCESS_KEY_ID: R3J3MlNWSUg2WnhTTU9ralNJOHBRQk10enFoalo5Zlk=
  AWS_SECRET_ACCESS_KEY: RlNBV29hbmRzVmhUY0dNdndIU0tfV2QtTnR3eTJaUDA=

---
apiVersion: v1
kind: Secret
metadata:
  name: dkp-velero
  namespace: kommander
type: Opaque
data:
  AWS_ACCESS_KEY_ID: X3kxZ2R3YXgxRXdvcFFYUW8xRVhNMWVnb0JZaWpnRjA=
  AWS_SECRET_ACCESS_KEY: NWo4eEdDSzE2LTFfM0hIcmJtbVhWcDM5ZTFDaTk2YjU=
  cloud: W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkPV95MWdkd2F4MUV3b3BRWFFvMUVYTTFlZ29CWWlqZ0YwCmF3c19zZWNyZXRfYWNjZXNzX2tleT01ajh4R0NLMTYtMV8zSEhyYm1tWFZwMzllMUNpOTZiNQ==
---
             
# 部署 secret 
kubectl apply -f dkp-loki-secret.yaml 
kubectl apply -f dkp-velero-secret.yaml

# 驗證 secret 內的 key 是否正確
export AWS_ACCESS_KEY_ID=`kubectl get secret -n kommander dkp-velero -o 'jsonpath={.data.AWS_ACCESS_KEY_ID}' | base64 --decode;echo`

export AWS_SECRET_ACCESS_KEY=`kubectl get secret -n kommander dkp-velero -o 'jsonpath={.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode;echo`


# 手動新增secret的方法
kubectl -n kommander create secret generic dkp-velero \
                      --from-literal=AWS_ACCESS_KEY_ID=GRW2SVIH6ZxSMOkjSI8pQBmtzqhjZ9fY\
                      --from-literal=AWS_SECRET_ACCESS_KEY=FSAWoandsVhTcGMvvHSK_Wd-Ntwy2ZP0
        


```

##### Install kommander

利用 kommander.yaml 來安裝設定

調整 grafana-loki, velero config 來串接外部 S3 Object Storage ，ex. Nutanix Object Storage 

```
kommander-install.yaml
---
apiVersion: config.kommander.mesosphere.io/v1alpha1
kind: Installation
namespace: kommander
apps:
  dex:
    enabled: true
  dex-k8s-authenticator:
    enabled: true
  nkp-insights-management:
    enabled: false
  gatekeeper:
    enabled: true
  git-operator:
    enabled: true
  grafana-logging:
    enabled: false
  grafana-loki:
    enabled: true
    values: |
      loki:
        structuredConfig:
          storage_config:
            aws:
              s3: s3://ntnxobject.nutanixlab.local/nkp-loki
              bucketnames: nkp-loki
              endpoint: http://ntnxobject.nutanixlab.local
              insecure: true
              sse_encryption: false
              s3forcepathstyle: true
              region: "us-east-1"
          table_manager:
            retention_deletes_enabled: true
            retention_period: 7d
          limits_config:
            retention_period: 168h  # 7 days in hours
        extraEnvFrom:
          - secretRef:
              name: dkp-loki
  kommander:
    enabled: true
  kommander-ui:
    enabled: true
  kube-prometheus-stack:
    enabled: false
  kubernetes-dashboard:
    enabled: false
  kubefed:
    enabled: true
  kubetunnel:
    enabled: false
  logging-operator:
    enabled: false
  prometheus-adapter:
    enabled: false
  reloader:
    enabled: true
  rook-ceph:
    enabled: false
  rook-ceph-cluster:
    enabled: false
  traefik:
    enabled: true
  traefik-forward-auth-mgmt:
    enabled: true
  velero:
    enabled: true
    values: |
      configuration:
        backupStorageLocation:
        - name: "nkp-velero"
          bucket: "nkp-velero"
          provider: "aws"
          default: true
          config:
            s3Url: "http://ntnxobject.nutanixlab.local"
            insecureSkipTLSVerify: "true"
            s3ForcePathStyle: "true"
            region: "us-east-1"
      deployNodeAgent: true # 為了要透過 kopia 來備份 pv的Daemonset
      nodeAgent:
        uploaderType: kopia
        extraEnvFrom:
        - secretRef:
            name: dkp-velero
      defaultVolumesToFsBackup: true  # 是讓 Velero 預設對所有沒有快照的 PVC 使用 Kopia 備份
      extraEnvFrom:
        - secretRef:
            name: dkp-velero
ageEncryptionSecretName: sops-age
clusterHostname: ""
airgapped:
  enabled: true        
---

nkp install kommander --installer-config=kommander-install.yaml
```

![image-20250419135103514](https://kenkenny.synology.me:5543/images/2025/04/image-20250419135103514.png)

![image-20250419135221779](https://kenkenny.synology.me:5543/images/2025/04/image-20250419135221779.png)



驗證 velero backupstoragelocaltion 以及 velero 的 pod 是否正常

```
kubectl get pods -n kommander |grep velero

kubectl get configmaps -n kommander |grep velero

kubectl describe configmaps velero-overrides -n kommander
 
kubectl get bsl -n kommander

NAME         PHASE       LAST VALIDATED   AGE   DEFAULT
nkp-velero   Available   18s              18h   true
 
kubectl describe bsl nkp-velero -n kommander
```

![image-20250417112609804](https://kenkenny.synology.me:5543/images/2025/04/image-20250417112609804.png)

![image-20250417112723243](https://kenkenny.synology.me:5543/images/2025/04/image-20250417112723243.png)

![image-20250417112543669](https://kenkenny.synology.me:5543/images/2025/04/image-20250417112543669.png)

##### 啟用 Logging Stack

Logging Stack 架構：

NKP 的 Logging Stack 主要用於**收集、傳輸、儲存與視覺化 K8s 的日誌資料**，支援多租戶架構與 RBAC 控制

BanzaiCloud Logging Operator：負責管理 Fluent Bit、Fluentd 等日誌代理程式的部署與設定

Fluent Bit：**輕量級** agent，部署在每個 Node，負責收集容器與系統 log。

Fluentd：**強大擴充性**的 log 處理器，處理來自 Fluent Bit 的資料（轉換、標籤、轉發）。

Grafana Loki：Log 儲存與查詢系統，儲存經過處理後的 log 資料。

Grafana：提供 UI 界面供使用者查詢 Loki 中的日誌資料

![image-20250416142052497](https://kenkenny.synology.me:5543/images/2025/04/image-20250416142052497.png)



啟用後如下，並確認資料Grafana Loki 有收到：

![image-20250416141900100](https://kenkenny.synology.me:5543/images/2025/04/image-20250416141900100.png)

![image-20250419134912927](https://kenkenny.synology.me:5543/images/2025/04/image-20250419134912927.png)

![image-20250419134934742](https://kenkenny.synology.me:5543/images/2025/04/image-20250419134934742.png)

![image-20250419135022107](https://kenkenny.synology.me:5543/images/2025/04/image-20250419135022107.png)

![image-20250419135408487](https://kenkenny.synology.me:5543/images/2025/04/image-20250419135408487.png)



##### 驗證Velero備份還原

建立測試服務 wordpress，初始化後新增一篇文章

```
kubectl apply -f wordpress.yaml

kubectl get all -n wordpress

NAME                             READY   STATUS    RESTARTS   AGE
pod/mysql-set-0                  1/1     Running   0          50s
pod/wordpress-699d5d76f9-qvhrt   1/1     Running   0          50s

NAME                TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
service/mysql       ClusterIP      10.99.63.17    <none>           3306/TCP       50s
service/wordpress   LoadBalancer   10.103.140.5   192.168.102.94   80:32608/TCP   50s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wordpress   1/1     1            1           50s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/wordpress-699d5d76f9   1         1         1       50s

NAME                         READY   AGE
statefulset.apps/mysql-set   1/1     50s


kubectl get pvc -n wordpress

NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     VOLUMEATTRIBUTESCLASS   AGE
mysql-data-1-mysql-set-0   Bound    pvc-e8b52c52-b545-4608-93aa-584a5d861613   3Gi        RWO            nutanix-volume   <unset>                 9m42s
mysql-store-mysql-set-0    Bound    pvc-75006256-32ca-40fe-bc38-58b30711be6b   5Gi        RWO            nutanix-volume   <unset>                 9m42s
wordpress-pv-claim         Bound    pvc-2074c06c-06b8-43a4-8750-c284d48ce634   1Gi        RWO            nutanix-volume   <unset>                 9m42s
```

![image-20250417154415866](https://kenkenny.synology.me:5543/images/2025/04/image-20250417154415866.png)

![image-20250417161530780](https://kenkenny.synology.me:5543/images/2025/04/image-20250417161530780.png)

測試備份

velero-backup-test.yaml

```
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: velero-backup-wordpress
  namespace: kommander
spec:
  includedNamespaces:
    - wordpress  # 改成你想備份的 Namespace，例如 kube-system、platform 等
  ttl: 72h0m0s  # 備份保留 72 小時
  storageLocation: nkp-velero
  snapshotVolumes: false
  defaultVolumesToFsBackup: true # 透過檔案層級來備份，但會比volume snapshot還慢
  hooks: {}
```

執行備份並確認

```
kubectl apply -f velero-backup-wordpress.yaml

kubectl get backup -n kommander

kubectl get backuprepository -n kommander

velero backup get -n kommander

velero backup describe velero-backup-wordpress -n kommander
```

![image-20250418135629473](https://kenkenny.synology.me:5543/images/2025/04/image-20250418135629473.png)

![image-20250418135647189](https://kenkenny.synology.me:5543/images/2025/04/image-20250418135647189.png)

![image-20250418135659451](https://kenkenny.synology.me:5543/images/2025/04/image-20250418135659451.png)

![image-20250418135730595](https://kenkenny.synology.me:5543/images/2025/04/image-20250418135730595.png)

![image-20250417155111513](https://kenkenny.synology.me:5543/images/2025/04/image-20250417155111513.png)

![image-20250418135826208](https://kenkenny.synology.me:5543/images/2025/04/image-20250418135826208.png)

![image-20250418135807410](https://kenkenny.synology.me:5543/images/2025/04/image-20250418135807410.png)

![image-20250418135924806](https://kenkenny.synology.me:5543/images/2025/04/image-20250418135924806.png)

刪除 wordpress namespace 的資源

```
kubectl delete ns wordpress

kubectl get pvc -n wordpress

kubectl get all -n wordpress
```

![image-20250417155443108](https://kenkenny.synology.me:5543/images/2025/04/image-20250417155443108.png)

![image-20250418140028703](https://kenkenny.synology.me:5543/images/2025/04/image-20250418140028703.png)

還原備份

```
vi velero-restore-wordpress.yaml 

apiVersion: velero.io/v1
kind: Restore
metadata:
  name: velero-restore-wordpress
  namespace: kommander
spec:
  backupName: velero-backup-wordpress
  restorePVs: true
  includedNamespaces:
    - wordpress
   #- '*' 全部
  # 指定要還原的資源種類（例如 deployment, service, pvc 等）
  #includedResources:
  #  - deployment
  #  - service
  #  - persistentvolumeclaim
  # 可以選擇排除某些資源（可選）
  excludedResources:
    - events
    - events.events.k8s.io
  # 可選：防止因重複還原導致名稱衝突
  namespaceMapping:
    wordpress: wordpress-restored
        
```

```
kubectl apply -f velero-restore-wordpress.yaml 

kubectl get restore -n kommander

velero get restore -n kommander

kubectl get ns |grep wordpress

kubectl get all -n wordpress

刪除備份
kubectl get backuprepository -n kommander

kubectl get restore -n kommander

kubectl get backup -n kommander
```

![image-20250418140202849](https://kenkenny.synology.me:5543/images/2025/04/image-20250418140202849.png)

![image-20250418140520582](https://kenkenny.synology.me:5543/images/2025/04/image-20250418140520582.png)

![image-20250418140252326](https://kenkenny.synology.me:5543/images/2025/04/image-20250418140252326.png)

![image-20250419134808615](https://kenkenny.synology.me:5543/images/2025/04/image-20250419134808615.png)

##### Velero 獨立安裝 (option)

額外手動安裝來測試備份還原，此方法與 NKP Operator 無關

用 minio 測試

https://learn.microsoft.com/en-us/azure/aks/aksarc/backup-workload-cluster

```
# 設定連線
[root@ken-rhel9 minio]# mc alias set minio http://192.168.102.93:9000 "minioadmin" "miniokey" --api s3v4
mc: Configuration written to `/root/.mc/config.json`. Please update your access credentials.
mc: Successfully created `/root/.mc/share`.
mc: Initialized share uploads `/root/.mc/share/uploads.json` file.
mc: Initialized share downloads `/root/.mc/share/downloads.json` file.
Added `minio` successfully.

[root@ken-rhel9 minio]# mc ls minio
[root@ken-rhel9 minio]# mc mb minio/velero-backup
Bucket created successfully `minio/velero-backup`.
[root@ken-rhel9 minio]# mc ls minio
[2025-04-16 12:00:30 CST]     0B velero-backup/

# 安裝指定 minio object
velero install --provider aws --bucket velero-backup --secret-file ./minio.credentials --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://192.168.102.93:9000 --plugins velero/velero-plugin-for-aws:v1.1.0

kubectl get bsl -n velero
NAME      PHASE       LAST VALIDATED   AGE    DEFAULT
default   Available   16s              2m4s   true

# 安裝指定 nutanix object
velero install --name nkp-velero --provider aws --bucket ntnx-object-nkp --secret-file ./ntnxobject.credentials --backup-location-config region=us-east-1,s3ForcePathStyle=true,s3Url=http://172.16.90.148/ntnx-object-nkp --plugins velero/velero-plugin-for-aws:v1.1.0

# 移除
[root@ken-rhel9 minio]# velero uninstall
You are about to uninstall Velero.
Are you sure you want to continue (Y/N)? Y
Waiting for resource with attached finalizer to be deleted

Waiting for velero namespace "velero" to be deleted
..................................................................................................................
Velero namespace "velero" deleted
Velero uninstalled ⛵
```



手動執行備份驗證

```
velero backup create nutanix-velero-testbackup -n kommander --storage-location nkp-velero --snapshot-volumes=false

velero backup describe nutanix-velero-testbackup -n kommander
```



#### Harbor

透過 NKP Operator 安裝 Harbor

啟用 COSI Driver For Nutanix、CloudNativePG、Harbor

![image-20250423134512608](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134512608.png)

COSI

![image-20250423134601622](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134601622.png)

![image-20250423134618086](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134618086.png)

Harbor 啟用後會自己開始安裝，並且透過 Nutanix COSI 自行產生一個 Bucket

![image-20250423134646652](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134646652.png)

![image-20250423134727889](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134727889.png)

![image-20250423134743977](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134743977.png)

取得密碼來連線 Harbor

```
kubectl get secrets -n ncr-system harbor-admin-password -o jsonpath='{.data.HARBOR_ADMIN_PASSWORD}' | base64 -d

Hq2r5?rliC9k0pdUoknZ
```

![image-20250423134901179](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134901179.png)

![image-20250423134922422](https://kenkenny.synology.me:5543/images/2025/04/image-20250423134922422.png)

![image-20250423135138124](https://kenkenny.synology.me:5543/images/2025/04/image-20250423135138124.png)

預設有開啟 Trivy 掃描映像檔

![image-20250423142413740](https://kenkenny.synology.me:5543/images/2025/04/image-20250423142413740.png)

新增 Registry 這樣 Harbor 就可以作為 Docker Hub 的快取站

![image-20250423135246587](https://kenkenny.synology.me:5543/images/2025/04/image-20250423135246587.png)

新增專案，勾選代理快取

![image-20250423135340904](https://kenkenny.synology.me:5543/images/2025/04/image-20250423135340904.png)

新增管理員使用者

![image-20250423135644812](https://kenkenny.synology.me:5543/images/2025/04/image-20250423135644812.png)

新增 Harbor User 的 Secret

```
REGISTRY_USERNAME="nkpuser"
REGISTRY_PASSWORD="Nutanix/4u"

kubectl create secret generic harbor-registry-credentials \
 --from-literal username=nkpuser \
 --from-literal password=Nutanix/4u \
 --from-file=ca.crt=<(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.caBundle}' | base64 -d)
```

確認 Harbor URL

```
echo "https://$(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.address}'):5000"

https://192.168.102.92:5000
```

修改 Image Registry

```
kubectl edit cluster netfos-nkp

apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: <NAME>
spec:
  topology:
    variables:
      - name: clusterConfig
        value:
          imageRegistries:
          - url: <HARBOR_ADDRESS>
            credentials:
              secretRef:
                name: harbor-registry-credentials

```

![image-20250423140107149](https://kenkenny.synology.me:5543/images/2025/04/image-20250423140107149.png)

測試 nginx pod 看是否會透過 Harbor 來 pull

```
nginx-test.yaml 

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: 192.168.102.92:5000/app/nginx:alpine
    ports:
    - containerPort: 80
---

kubectl apply -f nginx-test.yaml

kubectl get pods

kubectl describe pod nginx

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  86s   default-scheduler  Successfully assigned default/nginx to netfos-nkp-md-0-27bzg-9tz7l-2mgh6
  Normal  Pulling    85s   kubelet            Pulling image "192.168.102.92:5000/app/nginx:alpine"
  Normal  Pulled     55s   kubelet            Successfully pulled image "192.168.102.92:5000/app/nginx:alpine" in 29.85s (29.85s including waiting). Image size: 20984244 bytes.
  Normal  Created    55s   kubelet            Created container nginx
  Normal  Started    54s   kubelet            Started container nginx
```

![image-20250423141916852](https://kenkenny.synology.me:5543/images/2025/04/image-20250423141916852.png)

![image-20250423141940027](https://kenkenny.synology.me:5543/images/2025/04/image-20250423141940027.png)

![image-20250423142008143](https://kenkenny.synology.me:5543/images/2025/04/image-20250423142008143.png)

![image-20250423142020772](https://kenkenny.synology.me:5543/images/2025/04/image-20250423142020772.png)

### kommander.yaml

```
nkp install kommander --init --airgapped > kommander.yaml

apiVersion: config.kommander.mesosphere.io/v1alpha1
kind: Installation
apps:
  dex:
    enabled: true
  dex-k8s-authenticator:
    enabled: true
  gatekeeper:
    enabled: true
  git-operator:
    enabled: true
  grafana-logging:
    enabled: true
  grafana-loki:
    enabled: true
  kommander:
    enabled: true
  kommander-ui:
    enabled: true
  kube-prometheus-stack:
    enabled: true
  kubefed:
    enabled: true
  kubernetes-dashboard:
    enabled: true
  kubetunnel:
    enabled: true
  logging-operator:
    enabled: true
  nkp-insights-management:
    enabled: true
  prometheus-adapter:
    enabled: true
  reloader:
    enabled: true
  rook-ceph:
    enabled: true
  rook-ceph-cluster:
    enabled: true
  traefik:
    enabled: true
    values: |
      service:
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-internal: "true"
  traefik-forward-auth-mgmt:
    enabled: true
  velero:
    enabled: true
ageEncryptionSecretName: sops-age
clusterHostname: ""
airgapped:
  enabled: true
```

kommander-minimal.yaml 最小安裝

```
apiVersion: config.kommander.mesosphere.io/v1alpha1
kind: Installation
apps:
  dex:
    enabled: true
  dex-k8s-authenticator:
    enabled: true
  nkp-insights-management:
    enabled: false
  gatekeeper:
    enabled: true
  git-operator:
    enabled: true
  grafana-logging:
    enabled: false
  grafana-loki:
    enabled: false
  kommander:
    enabled: true
  kommander-ui:
    enabled: true
  kube-prometheus-stack:
    enabled: false
  kubernetes-dashboard:
    enabled: false
  kubefed:
    enabled: true
  kubetunnel:
    enabled: false
  logging-operator:
    enabled: false
  prometheus-adapter:
    enabled: false
  reloader:
    enabled: true
  rook-ceph:
    enabled: false
  rook-ceph-cluster:
    enabled: false
  traefik:
    enabled: true
  traefik-forward-auth-mgmt:
    enabled: true
  velero:
    enabled: false
ageEncryptionSecretName: sops-age
clusterHostname: ""
```

安裝參照

```
nkp install kommander \
--installer-config kommander-install.yaml --kubeconfig=${CLUSTER_NAME}.conf \
--kommander-applications-repository /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/application-repositories/kommander-applications-nkp-v2.14.0.tar.gz \
--charts-bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/application-charts/nkp-kommander-charts-bundle-nkp-v2.14.0.tar.gz \
--charts-bundle /home/nkp/nkptools/nkp-airgap/nkp-v2.14.0/application-charts/nkp-catalog-applications-charts-bundle-nkp-v2.14.0.tar.gz --kubeconfig=$MGMT_CLUSTER_NAME.conf
```





Volume Snpashot

```
[root@ken-rhel9 ~]# kubectl get VolumeSnapshotClass
NAME                     DRIVER            DELETIONPOLICY   AGE
nutanix-snapshot-class   csi.nutanix.com   Retain           7d17h
[root@ken-rhel9 ~]# kubectl describe VolumeSnapshotClass nutanix-snapshot-class
Name:             nutanix-snapshot-class
Namespace:        
Labels:           app.kubernetes.io/managed-by=Helm
Annotations:      meta.helm.sh/release-name: nutanix-csi
                  meta.helm.sh/release-namespace: ntnx-system
API Version:      snapshot.storage.k8s.io/v1
Deletion Policy:  Retain
Driver:           csi.nutanix.com
Kind:             VolumeSnapshotClass
Metadata:
  Creation Timestamp:  2025-04-10T08:46:19Z
  Generation:          1
  Resource Version:    2229
  UID:                 80308da9-38e6-418d-97d4-af6a8f507d5d
Parameters:
  Storage Type:  NutanixVolumes
Events:          <none>

kubectl annotate volumesnapshotclass nutanix-snapshot-class snapshot.storage.kubernetes.io/is-default-class="true"

[root@ken-rhel9 ~]# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     VOLUMEATTRIBUTESCLASS   AGE
minio-pv-claim   Bound    pvc-891efd37-285d-4fda-b0dd-9a63e3fa63e3   100Gi      RWO            nutanix-volume   <unset>                 47h

[root@ken-rhel9 ~]# vi volumesnapshot-minio.yaml 

apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: snapshot-minio
  namespace: default
spec:
  volumeSnapshotClassName: nutanix-snapshot-class
  source:
    persistentVolumeClaimName: minio-pv-claim

[root@ken-rhel9 ~]# kubectl apply -f volumesnapshot-minio.yaml 
volumesnapshot.snapshot.storage.k8s.io/snapshot-minio created

[root@ken-rhel9 ~]# kubectl get volumesnapshot
NAME             READYTOUSE   SOURCEPVC        SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS            SNAPSHOTCONTENT                                    CREATIONTIME   AGE
snapshot-minio   false        minio-pv-claim                                         nutanix-snapshot-class   snapcontent-b4c279fa-5b38-4465-9e3f-58a37b9806c4 3s

[root@ken-rhel9 ~]# kubectl get volumesnapshot
NAME             READYTOUSE   SOURCEPVC        SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS            SNAPSHOTCONTENT                                    CREATIONTIME   AGE
snapshot-minio   true         minio-pv-claim                           100Gi         nutanix-snapshot-class   snapcontent-b4c279fa-5b38-4465-9e3f-58a37b9806c4   9s             11s


# restore
[root@ken-rhel9 ~]# vi volumesnapshot-createpv-minio.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pv-claim2
spec:
  storageClassName: nutanix-volume
  dataSource:
    name: snapshot-minio
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
      

```

![image-20250418095856956](https://kenkenny.synology.me:5543/images/2025/04/image-20250418095856956.png)

![image-20250418100027699](https://kenkenny.synology.me:5543/images/2025/04/image-20250418100027699.png)

![image-20250418100408948](https://kenkenny.synology.me:5543/images/2025/04/image-20250418100408948.png)



## DR Cluster

DR Cluster 用 Nutanix 原生的 Rocky linux os

control plane 1 台、worker 2台

env

```
export MGMT_CLUSTER_NAME="netfos-nkp-dr"
export MGMT_CP_ENDPOINT="172.16.90.208"
export MGMT_LB_IP_RANGE="172.16.90.209-172.16.90.211"
export PC_ENDPOINT="172.16.90.72"
export PE_CLUSTER_NAME="NX3500PE"
export SUBNET="Netfos_90_Access_IPAM"
export VM_IMAGE="nkp-rocky-9.5-release-1.31.4-20250214003015.qcow2"
export STORAGE_CONTAINER="Kubernetes-Container"
export NUTANIX_USER="admin"
export NUTANIX_PASSWORD="adminpassword"
export CSI_HYPERVISOR_ATTACHED="true"
export CSI_FILESYSTEM="ext4"
export SSH_USER="nkp"
export SSH_PUBLIC_KEY="/home/nkp/.ssh/id_rsa.pub"
export BOOTSTRAP_HOST_IP="172.16.90.212"
export REGISTRY_URL="http://172.16.90.206/nkp-2.14"
export REGISTRY_USERNAME="admin"
export REGISTRY_PASSWORD="Harbor12345"
export BASTION_HOST="172.16.90.212"
export BASTION_SSH_USER="nkp"
export BASTION_KEY="/home/nkp/.ssh/id_rsa"
export BASTION_PORT="22"

# source env-dr.sh
```



```
nkp create cluster nutanix \
--cluster-name=$MGMT_CLUSTER_NAME \
--control-plane-prism-element-cluster=$PE_CLUSTER_NAME \
--control-plane-subnets=$SUBNET \
--control-plane-endpoint-ip=$MGMT_CP_ENDPOINT \
--control-plane-vm-image=$VM_IMAGE \
--control-plane-replicas=1 \
--control-plane-memory=8 \
--worker-prism-element-cluster=$PE_CLUSTER_NAME \
--worker-subnets=$SUBNET \
--worker-vm-image=$VM_IMAGE \
--worker-memory=8 \
--worker-replicas=2 \
--csi-storage-container=$STORAGE_CONTAINER \
--kubernetes-service-load-balancer-ip-range=$MGMT_LB_IP_RANGE \
--endpoint="https://$PC_ENDPOINT:9440" \
--insecure=true \
--registry-mirror-url=$REGISTRY_URL \
--registry-mirror-username=${REGISTRY_USERNAME} \
--registry-mirror-password=${REGISTRY_PASSWORD} \
--ssh-username=$SSH_USER \
--ssh-public-key-file=$SSH_PUBLIC_KEY \
--airgapped \
--self-managed
```

過程中有 timeout ，就要手動安裝 kommander

![image-20250422130430249](https://kenkenny.synology.me:5543/images/2025/04/image-20250422130430249.png)

![image-20250422130451375](https://kenkenny.synology.me:5543/images/2025/04/image-20250422130451375.png)

![image-20250422130337037](https://kenkenny.synology.me:5543/images/2025/04/image-20250422130337037.png)

![image-20250422130300764](https://kenkenny.synology.me:5543/images/2025/04/image-20250422130300764.png)

![image-20250422130515406](https://kenkenny.synology.me:5543/images/2025/04/image-20250422130515406.png)

## NDK

Nutanix Data Service for Kubernetes

### Requirement

1. DR及主要的叢集中 Prism Central 與 Prism Element 的 UUID 以及 Data Service IP

2. Prism Central 啟用 Kubernetes Management 並更新到最新，兩地的 Prism Central 要做Availability Zones串連

3. 兩地的 Kubernetes 必須同一個供應商 ex. NKP 對 NKP or OCP 對 OCP

4. Kubenetes Cluster  安裝 Nutanix CSI 3.0 以上、 Cert-manager、Helm ( NKP預設皆有安裝 ) 

5. Onboard Kubenetes Cluster 到 Prism Central

6. 保留一組 Loadbalancer IP 給 NDK 使用
7. 確認 Image Registry 有 k8s-agent 以及 ndk 相關的 image

### 下載&安裝

DR Site 也是一模一樣的步驟

#### 下載

![image-20250421100319536](https://kenkenny.synology.me:5543/images/2025/04/image-20250421100319536.png)

```
[root@ken-rhel9 ndk]# curl -o ndk-1.2.0.tar "https://download.nutanix.com/downloads/ndk/1.2.0/ndk-1.2.0.tar?Expires=1745237015&Key-Pair-Id=APKAJTTNCWPEI42q5efZ3aJGjekGgoW5w__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  526M  100  526M    0     0  18.3M      0  0:00:28  0:00:28 --:--:-- 23.5M

[root@ken-rhel9 ndk]# curl -o k8s-agent-1.1.688.tar "https://download.nutanix.com/downloads/ndk/1.2.0/k8s-agent-1.1.688.tar?Expires=1745237080&Key-Pair-Id=APKAJTTNCWPEI42QKMSA&Signature=PB74DhIdBozRvpm-wEgkzVZQU5N2YksxO3T6ew1WAwt7itJwx-swYqrDuw__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 20.4M  100 20.4M    0     0  5921k      0  0:00:03  0:00:03 --:--:-- 5921k

[root@ken-rhel9 ndk]# ll
total 559692
-rw-r--r-- 1 root root  21428736 Apr 21 10:04 k8s-agent-1.1.688.tar
-rw-r--r-- 1 root root 551690240 Apr 21 10:04 ndk-1.2.0.tar

[root@ken-rhel9 ndk]# tar xvf ndk-1.2.0.tar 
[root@ken-rhel9 ndk]# tar xvf k8s-agent-1.1.688.tar 
```

#### 推送到 Image Registry

##### k8s-agent

k8s-agent 可以讓使用者從 Prism Central 看到 Kubernetes 的資源使用情況

```
docker image load -i k8s-agent-1-1-688/k8s-agent-1.1.688.tar

外部

docker image tag k8s-agent:1.1.688 kenkennyinfo/k8s-agent:688

docker push kenkennyinfo/k8s-agent:688


私倉
docker image tag k8s-agent:1.1.688 172.16.90.206/nkp-2.14/k8s-agent:688

docker push 172.16.90.206/nkp-2.14/k8s-agent:688

rm k8s-agent-1-1-688/k8s-agent-1.1.688.tar
```

外部

![image-20250421120433971](https://kenkenny.synology.me:5543/images/2025/04/image-20250421120433971.png)

私倉

![image-20250421105052026](https://kenkenny.synology.me:5543/images/2025/04/image-20250421105052026.png)

![image-20250421105033776](https://kenkenny.synology.me:5543/images/2025/04/image-20250421105033776.png)

##### NDK

```
docker image load -i ndk-1.2.0/ndk-1.2.0.tar


    ndk/manager:<version>
    ndk/infra-manager:<version>
    ndk/job-scheduler:<version>
    ndk/kube-rbac-proxy:<version>
    ndk/bitnami-kubectl:<version>

docker images |grep ndk
ndk/manager                        1.2.0     ce1fad021e29   6 months ago    67.9MB
ndk/infra-manager                  1.2.0     862126c207bb   6 months ago    57.9MB
ndk/bitnami-kubectl                1.30.3    9c2184d8f9cc   8 months ago    293MB
ndk/job-scheduler                  1.2.0     452e4e230a33   11 months ago   57MB
ndk/kube-rbac-proxy                v0.17.0   2d1f79a3b6dd   12 months ago   67.3MB

外部

docker tag ndk/manager:1.2.0 kenkennyinfo/ndk-manager:1.2.0
docker tag ndk/infra-manager:1.2.0 kenkennyinfo/ndk-infra-manager:1.2.0
docker tag ndk/job-scheduler:1.2.0 kenkennyinfo/ndk-job-scheduler:1.2.0
docker tag ndk/kube-rbac-proxy:v0.17.0 kenkennyinfo/ndk-kube-rbac-proxy:v0.17.0
docker tag ndk/bitnami-kubectl:1.30.3 kenkennyinfo/ndk-bitnami-kubectl:1.30.3


也可以用 for 迴圈
for img in ndk/manager:1.2.0 ndk/infra-manager:1.2.0 ndk/job-scheduler:1.2.0 ndk/kube-rbac-proxy:v0.17.0 ndk/bitnami-kubectl:1.30.3; do docker tag $img kenkennyinfo/${img}; done

docker images |grep ndk

kenkennyinfo/ndk-manager           1.2.0     ce1fad021e29   6 months ago    67.9MB
ndk/manager                        1.2.0     ce1fad021e29   6 months ago    67.9MB
kenkennyinfo/ndk-infra-manager     1.2.0     862126c207bb   6 months ago    57.9MB
ndk/infra-manager                  1.2.0     862126c207bb   6 months ago    57.9MB
kenkennyinfo/ndk-bitnami-kubectl   1.30.3    9c2184d8f9cc   8 months ago    293MB
ndk/bitnami-kubectl                1.30.3    9c2184d8f9cc   8 months ago    293MB
kenkennyinfo/ndk-job-scheduler     1.2.0     452e4e230a33   11 months ago   57MB
ndk/job-scheduler                  1.2.0     452e4e230a33   11 months ago   57MB
kenkennyinfo/ndk-kube-rbac-proxy   v0.17.0   2d1f79a3b6dd   12 months ago   67.3MB
ndk/kube-rbac-proxy                v0.17.0   2d1f79a3b6dd   12 months ago   67.3MB


docker push kenkennyinfo/ndk-manager:1.2.0
docker push kenkennyinfo/ndk-kube-rbac-proxy:v0.17.0
docker push kenkennyinfo/ndk-infra-manager:1.2.0
docker push kenkennyinfo/ndk-bitnami-kubectl:1.30.3
docker push kenkennyinfo/ndk-job-scheduler:1.2.0
```

![image-20250421134944759](https://kenkenny.synology.me:5543/images/2025/04/image-20250421134944759.png)

![image-20250422092158030](https://kenkenny.synology.me:5543/images/2025/04/image-20250422092158030.png)

![image-20250422101926581](https://kenkenny.synology.me:5543/images/2025/04/image-20250422101926581.png)



#### Helm 安裝

image registry dockerconfig

```
# 私倉範例
cat ~/.docker/config.json
{
	"auths": {
		"172.16.90.206": {
			"auth": "YWRtaW46SGFyYm9yMTIzNDU="
		}
	}


kubectl create secret docker-registry nutanix-mirror-image-pull-secret \
  --docker-server=172.16.90.206/nkp-2.14 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --docker-email=nkp@nutanixlab.local \
  --namespace=ntnx-system

kubectl get secret nutanix-mirror-image-pull-secret -n ntnx-system -o jsonpath='{.data.\.dockerconfigjson}'


# 外部 docker.io
echo -n 'username:password' | base64

dockerconfig.json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "username": "kenkennyinfo",
      "auth": "a2VuassxxxxxxxxxsxA2"
    }
  }
}

cat dockerconfig.json |base64 -w 0

ewogICJhdXRocyI6IHsKICAxxxxxxxxxxxxxxUnhkWEF6YlhBMiIKICAgIH0KICB9Cn0KCg=

or 

kubectl create secret docker-registry nutanix-docker-image-pull-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=kenkennyinfo \
  --docker-password=xxxxxxxxx \
  --docker-email=kenkennyinfo@gmail.com \
  --namespace=ntnx-system
```

##### k8s-agent

```
範例：
  helm install <release-name> <path-to-k8s-agent-folder>-<version> --set pc.endpoint=<pc-endpoint>,pc.username=<prism-central-user-name>,pc.password=<prism-central-user-password>,pc.insecure=true,agent.image.imageCredentials.dockerconfig=<docker-config>,k8sClusterUUID=<k8s-cluster-UUID>,k8sClusterName=<k8s-cluster-name>,k8sDistribution=<k8s-distribution>, categoryMappings=<key=value> -n <namespace-name> --create-namespace

取得 k8s cluster uuid: 

kubectl get namespace kube-system --output jsonpath={.metadata.uid}
f4f5efc7-a995-42e9-8492-65776de9a6ee
```

修改 value.yaml

要注意 categoryMappings 這邊KubernetesClusterUUID 前面會多 "k8s-" ，這樣才看得到 Storage 資訊

```
agent:
  namespaceOverride: ntnx-system
  name: nutanix-agent
  port: 8080
  image:
    repository: docker.io/kenkennyinfo
    name: k8s-agent
    pullPolicy: IfNotPresent
    tag: 688
    privateRegistry: false
    imageCredentials:
      dockerconfig: "ewogxxxxxxxxxxCn0KCg="
  updateConfigInMin: 10
  updateMetricsInMin: 360

pc:
  port: 9440
  insecure: true #set this to true if PC does not have https enabled
  endpoint: "172.16.90.75" # eg: ip or fqdn
  username: "admin" # eg: admin or any other user with Kubernetes Infrastructure provision role
  password: "Nutanix/Lab123"
k8sClusterName: "netfos-nkp"
k8sClusterUUID: "f4f5efc7-a995-42e9-8492-65776de9a6ee"
k8sDistribution: "NKP" # eg: CAPX or NKE or OCP or EKSA or NKP
categoryMappings: "KubernetesClusterName=netfos-nkp,KubernetesClusterUUID=k8s-f4f5efc7-a995-42e9-8492-65776de9a6ee" # "one or more comma separated key=value" eg: "key1=value1" or "key1=value1\,key2=value2"

---
  
helm install nutanix-k8s-agent-1.1.688 k8s-agent-1-1-688/chart  -n ntnx-system


# 移除 
helm uninstall nutanix-k8s-agent-1.1.688 -n ntnx-system
helm list -n ntnx-system
kubectl get secret -n ntnx-system | grep nutanix-k8s-agent-1.1.688
kubectl get configmap -n ntnx-system | grep nutanix-k8s-agent-1.1.688
kubectl get deployment -n ntnx-system | grep nutanix-k8s-agent-1.1.688
kubectl delete secret sh.helm.release.v1.nutanix-k8s-agent-1.1.688.v1 -n ntnx-system
```

```
kubectl get all -n ntnx-system
NAME                                          READY   STATUS    RESTARTS      AGE
pod/nutanix-agent-6d84987884-rdcw5            1/1     Running   0             81s
pod/nutanix-csi-controller-5fc7b6c7dc-6tqsh   7/7     Running   0             64s
pod/nutanix-csi-controller-5fc7b6c7dc-dc22h   7/7     Running   0             36s
pod/nutanix-csi-node-2gnz7                    3/3     Running   1 (10d ago)   10d
pod/nutanix-csi-node-55qbd                    3/3     Running   0             10d
pod/nutanix-csi-node-h49hh                    3/3     Running   1 (10d ago)   10d
pod/nutanix-csi-node-h96p7                    3/3     Running   0             10d
pod/nutanix-csi-node-nf4gg                    3/3     Running   0             10d
pod/nutanix-csi-node-p6zr4                    3/3     Running   1 (10d ago)   10d

NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
service/nutanix-csi-metrics   ClusterIP   10.102.111.224   <none>        9809/TCP,9810/TCP,9811/TCP,9812/TCP   10d

NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/nutanix-csi-node   6         6         6       6            6           kubernetes.io/os=linux   10d

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nutanix-agent            1/1     1            1           81s
deployment.apps/nutanix-csi-controller   2/2     2            2           10d

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nutanix-agent-6d84987884            1         1         1       81s
replicaset.apps/nutanix-csi-controller-596698b7c7   0         0         0       10d
replicaset.apps/nutanix-csi-controller-5fc7b6c7dc   2         2         2       64s
```

![image-20250421134131653](https://kenkenny.synology.me:5543/images/2025/04/image-20250421134131653.png)

![image-20250422142618610](https://kenkenny.synology.me:5543/images/2025/04/image-20250422142618610.png)

![image-20250421134147052](https://kenkenny.synology.me:5543/images/2025/04/image-20250421134147052.png)

![image-20250421134221398](https://kenkenny.synology.me:5543/images/2025/04/image-20250421134221398.png)

![image-20250421134232884](https://kenkenny.synology.me:5543/images/2025/04/image-20250421134232884.png)

DR Cluster

![image-20250422142658483](https://kenkenny.synology.me:5543/images/2025/04/image-20250422142658483.png)

![image-20250422142719162](https://kenkenny.synology.me:5543/images/2025/04/image-20250422142719162.png)

![image-20250422142733030](https://kenkenny.synology.me:5543/images/2025/04/image-20250422142733030.png)

![image-20250422142746244](https://kenkenny.synology.me:5543/images/2025/04/image-20250422142746244.png)



##### NDK

先建立 Prism Central 連線的 Secret

```
$ vim ntnx-pc-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: ntnx-pc-secret
  namespace: ntnx-system
stringData:
  # prism-pc-ip:prism-port:admin:password
  key: 172.16.90.75:9440:admin:password
  
$ kubectl apply -f ntnx-pc-secret.yaml 
```

原廠範例設定 image 跟 tag

```
helm install ndk -n ntnx-system chart/ \
--setmanager.repository=<repository-name> \
--set manager.tag=<ndk-manager-tag-name> \
--set infraManager.repository=<repository-name> \
--set infraManager.tag=<infra_manager-tag-name> \
--set kubeRbacProxy.repository=<repository-name> \ 
--set kubeRbacProxy.tag=<kube-rbac-proxy-tag-name> \
--set bitnamiKubectl.repository=<repository-name> \ 
--set bitnamiKubectl.tag=<bitnami-kubectl-tag-name> \
--set jobScheduler.repository=<repository-name> \
--set jobScheduler.tag=<job-scheduler-tag-name> \
--set tls.server.clusterName=<cluster-name> 
```

或是選擇修改 value.yaml 在 Chart 目錄底下，如果要改連線透過 Node port 的話要把 Loadbalancer 換掉

```

manager:
  # -- Image Repository
  repository: kenkennyinfo/ndk-manager
  # -- Image tag
  # @default -- .Chart.AppVersion
  tag: "1.2.0"
  # -- Image digest
  digest:
  # -- Image pull policy
  pullPolicy: Always

infraManager:
  # -- Image Repository
  repository: kenkennyinfo/ndk-infra-manager
  # -- Image tag
  # @default -- .Chart.AppVersion
  tag: "1.2.0"
  # -- Image digest
  digest:
  # -- Image pull policy
  pullPolicy: Always

kubeRbacProxy:
  # -- Image Repository
  repository: kenkennyinfo/ndk-kube-rbac-proxy
  # -- Image tag
  tag: "v0.17.0"
  # -- Image digest
  digest:

bitnamiKubectl:
  # -- Image Repository
  repository: ndk-bitnami-kubectl
  # -- Image tag
  tag: "1.30.3"
  # -- Image digest
  digest:
  # -- Image pull policy
  pullPolicy: Always
  
jobScheduler:
  # -- Image Repository
  repository: kenkennyinfo/ndk-job-scheduler
  # -- Image tag
  # @default -- .Chart.AppVersion
  tag: "1.2.0"
  # -- Image digest
  digest:
  # -- Image pull policy
  pullPolicy: Always
  
config:
  # To provide secret to be used by controllers for storage backend authentication
  # @Required-- secret should be in the helm release namespace
  secret:
    # pointer to the secret to be consumed by controllers
    name: ntnx-pc-secret
    
imageCredentials:
  # Name of the secret containing the credentials to pull image from the container registry.
  imagePullSecretName: nutanix-k8s-agent-pull-secret
  
```

安裝

```
helm install ndk -n ntnx-system ndk-1.2.0/chart/ --set tls.server.enable=false 

kubectl get pods -n ntnx-system 

kubectl get svc -n ntnx-system

ndk-intercom-service                     LoadBalancer   10.96.106.131    192.168.102.93   2021:32476/TCP
```

![image-20250422103729165](https://kenkenny.synology.me:5543/images/2025/04/image-20250422103729165.png)

![image-20250422103936296](https://kenkenny.synology.me:5543/images/2025/04/image-20250422103936296.png)

![image-20250422140539949](https://kenkenny.synology.me:5543/images/2025/04/image-20250422140539949.png)

DR Cluster

![image-20250422143411540](https://kenkenny.synology.me:5543/images/2025/04/image-20250422143411540.png)

![image-20250422144010079](https://kenkenny.synology.me:5543/images/2025/04/image-20250422144010079.png)



#### 設定 NDK

##### StorageCluster

要先確認 PE Cluster 的 UUID 以及 PC 的 UUID

```
nutanix@cvm$ ncli cluster info

    Cluster Uuid              : 00061972-aeba-fee2-0000-000000028e95
    Cluster Name              : NX1365G6PE
    Cluster Version           : 6.8.1

nutanix@pcvm$ ncli cluster info

    Cluster Uuid              : acb94940-7f03-416e-9dcd-6b442d3fe081
    Cluster Name              : NTNXSELABPC
    Cluster Version           : pc.2024.1.0.1
    
DR
    Cluster Uuid              : 00061f20-3fc1-adc4-7dda-002590c84b9a
    Cluster Name              : NX3500PE
    Cluster Version           : 6.8.1
    
    Cluster Uuid              : 12ecbf3b-352d-4369-a7bb-ef71b7262f47
    Cluster Name              : Unnamed
    Cluster Version           : pc.2024.2
```

設定 StorageCluster-NX1365G6PE.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: StorageCluster
metadata:
 name: nx1365g6pe
spec:
 storageServerUuid: 00061972-aeba-fee2-0000-000000028e95
 managementServerUuid: acb94940-7f03-416e-9dcd-6b442d3fe081
```

StorageCluster-NX3500PE.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: StorageCluster
metadata:
 name: nx3500pe
spec:
 storageServerUuid: 00061f20-3fc1-adc4-7dda-002590c84b9a
 managementServerUuid: 12ecbf3b-352d-4369-a7bb-ef71b7262f47
```

部署 Storage Cluster

```
kubectl apply -f StorageCluster-NX1365G6PE.yaml

kubectl get storagecluster
kubectl describe storagecluster nx1365g6pe
```

![image-20250422105053031](https://kenkenny.synology.me:5543/images/2025/04/image-20250422105053031.png)

DR Cluster

![image-20250422143943377](https://kenkenny.synology.me:5543/images/2025/04/image-20250422143943377.png)



##### Remote Cluster

先確認 Remote 的 Cluster 有安裝好 NDK

Remote CR without TLS

RemoteCR.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: Remote
metadata:
  name: netfos-nkp-dr
spec:
  clusterName: netfos-nkp-dr
  ndkServiceIp: 172.16.90.210
  ndkServicePort: 2021
  
kubectl apply -f RemoteCR.yaml
kubectl get remote
kubectl describe remote netfos-nkp-dr
```

![image-20250422144349788](https://kenkenny.synology.me:5543/images/2025/04/image-20250422144349788.png)

![image-20250422144854146](https://kenkenny.synology.me:5543/images/2025/04/image-20250422144854146.png)

![image-20250422145156679](https://kenkenny.synology.me:5543/images/2025/04/image-20250422145156679.png)

ReplicationTarget.yaml

這個 Target 如果有要抄寫的 namespace 都要設定

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ReplicationTarget
metadata:
  name: netfos-nkp-dr
  namespace: ntnx-system
spec:
  namespaceName: ntnx-system
  remoteName: netfos-nkp-dr
  serviceAccountName: default
  
  
kubectl apply -f ReplicationTarget.yaml
kubectl get ReplicationTarget -n ntnx-system
kubectl describe ReplicationTarget -n ntnx-system


wordpress

apiVersion: dataservices.nutanix.com/v1alpha1
kind: ReplicationTarget
metadata:
  name: netfos-nkp-dr
  namespace: wordpress
spec:
  namespaceName: wordpress
  remoteName: netfos-nkp-dr
  serviceAccountName: default
  
kubectl apply -f ReplicationTarget-wordpress.yaml
kubectl get ReplicationTarget -n wordpress
kubectl describe ReplicationTarget -n wordpress
```

![image-20250422151958348](https://kenkenny.synology.me:5543/images/2025/04/image-20250422151958348.png)

### 測試 NDK

#### 確認Remote端連線

1. 確認兩地 Prism Central

   primary

   ![image-20250422154552843](https://kenkenny.synology.me:5543/images/2025/04/image-20250422154552843.png)

   ![image-20250422154841681](https://kenkenny.synology.me:5543/images/2025/04/image-20250422154841681.png)

   DR

   ![image-20250422154901028](https://kenkenny.synology.me:5543/images/2025/04/image-20250422154901028.png)

   ![image-20250422154919861](https://kenkenny.synology.me:5543/images/2025/04/image-20250422154919861.png)

   

2. 確認 StorageCluster

   ```
   kubectl get storagecluster
   ```

3. 確認 Remote CR

   ```
   kubectl get remote
   ```

4. 確認 ReplicationTarget

   ```
   kubectl get ReplicationTarget -n ntnx-system
   ```



#### 建立App CR

#Application CR

此範例是 namespace wordpress 內全部的東西，也可以設定 includeResources: 來指定要快照的項目

wordpress-cr.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: Application
metadata:
  name: wordpress-cr
  namespace: wordpress
spec:
  applicationSelector:

kubectl apply -f wordpress-cr.yaml
kubectl get application -n wordpress
```

![image-20250422155101699](https://kenkenny.synology.me:5543/images/2025/04/image-20250422155101699.png)

![image-20250422155148905](https://kenkenny.synology.me:5543/images/2025/04/image-20250422155148905.png)

#### 本地快照還原

#ApplicationSnapshot CR

#ApplicationSnapshotRestore CR

wordpress-snap-cr.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ApplicationSnapshot
metadata:
  name: wordpress-snap-1
  namespace: wordpress
spec:
  source:
    applicationRef:
     name: wordpress-cr 
  expiresAfter: 7200m
  
kubectl apply -f wordpress-snap-cr.yaml
kubectl get applicationsnapshot -n wordpress
kubectl get applicationsnapshot -n wordpress wordpress-snap-1 -o yaml -w
kubectl get applicationsnapshotcontent asc-396f53f3-3a85-408e-a13d-2ea494002b18 -o yaml -w
```

![image-20250422165749416](https://kenkenny.synology.me:5543/images/2025/04/image-20250422165749416.png)

delete wordpress namespace 內的物件

```
kubectl get all -n wordpress

kubectl delete service/mysql -n wordpress
kubectl delete service/wordpress -n wordpress
kubectl delete deployment.apps/wordpress -n wordpress
kubectl delete statefulset.apps/mysql-set -n wordpress
kubectl delete pvc mysql-data-1-mysql-set-0 -n wordpress
kubectl delete pvc mysql-store-mysql-set-0 -n wordpress
kubectl delete pvc wordpress-pv-claim -n wordpress
```

wordpress-restore-cr.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ApplicationSnapshotRestore
metadata:
  name: restore-wordpress-snap-1
  namespace: wordpress
spec:
  applicationSnapshotName: wordpress-snap-1
  
kubectl apply -f wordpress-restore-cr.yaml
kubectl get  ApplicationSnapshotRestore -n wordpress
kubectl describe ApplicationSnapshotRestore -n wordpress

```

![image-20250422165717853](https://kenkenny.synology.me:5543/images/2025/04/image-20250422165717853.png)

![image-20250422165656196](https://kenkenny.synology.me:5543/images/2025/04/image-20250422165656196.png)

#### 異地快照還原

#ApplicationSnapshotReplication CR

#ApplicationSnapshotRestore CR



wordpress-snap-replica.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ApplicationSnapshotReplication
metadata:
  name: wordpress-replication-cr
  namespace: wordpress
spec:
  applicationSnapshotName: wordpress-snap-1
  replicationTargetName: netfos-nkp-dr
  
kubectl get applicationsnapshots -n wordpress
kubectl apply -f wordpress-snap-replica.yaml
kubectl get applicationsnapshotreplication -n wordpress

dr cluster
kubectl get applicationsnapshots -n wordpress
```

wordpress-restore-replica.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ApplicationSnapshotRestore
metadata:
  name: restore-wordpress-snap-replica-1
  namespace: wordpress
spec:
  applicationSnapshotName: wordpress-replication-cr
  
kubectl apply -f wordpress-restore-replica.yaml
kubectl get ApplicationSnapshotRestore -n wordpress
kubectl describe ApplicationSnapshotRestore -n wordpress
```



#### 排程快照

#JobScheduler CR YAML

#ProtectionPlan CR YAML

#AppProtectionPlan CR



JobSchedulerCR.yaml

```
apiVersion: scheduler.nutanix.com/v1alpha1
kind: JobScheduler
metadata:
  name: minute-interval
spec:
  interval:
    minutes: 720
  timeZoneName: "Asia/Taipei"
  
kubectl apply -f JobSchedulerCR.yaml
```



ProtectionPlanCR.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ProtectionPlan
metadata: 
 name: wordpress-protectn-plan-1
 namespace: wordpress
spec: 
 scheduleName: minute-interval
 retentionPolicy:
     retentionCount: 5
 replicationConfigs:
   - replicationTargetName: netfos-nkp-dr
   
kubectl apply -f ProtectionPlanCR.yaml
```

AppProtectionPlanCR.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: AppProtectionPlan
metadata:
  name: app-plan
  namespace: wordpress
spec:
  applicationName: wordpress-cr
  labels:
    - appName: wordpress-cr
  protectionPlanNames:
  - wordpress-protectn-plan-1
```



#### 還原不同 namespace

要另外安裝 ReferenceGrant CRD

https://github.com/kubernetes-sigs/gateway-api/blob/main/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml

```
kubectl apply -f gateway.networking.k8s.io_referencegrants.yaml

kubectl get crd | grep 'referencegrants.gateway.networking.k8s.io'

ReferenceGrant YAML:

apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
 name: <reference_grant_name>
 namespace: <source_namespace>
spec:
 from:
 - group: dataservices.nutanix.com
   kind: ApplicationSnapshotRestore
   namespace: <target_namespace>
 to:
 - group: dataservices.nutanix.com
   kind: ApplicationSnapshot
   name: <resource_name>
   
kubectl apply -f <reference-grant-file-name>.yaml

ApplicationSnapshotRestore YAML (with specified namespace):

apiVersion: dataservices.nutanix.com/v1alpha1
kind: ApplicationSnapshotRestore
metadata:
 name: <restore-snapshot-name>
 namespace: <target_namespace>
spec:
 applicationSnapshotName: <snapshot_name>
 applicationSnapshotNamespace: <snapshot_namespace>  #optional
 
 kubectl apply -f <application-snapshot-restore-file-name>.yaml
```

