---
title: OCP-4.12.45_on_Nutanix(IPI)
description: OCP-4.12.45_on_Nutanix(IPI)
slug: OCP-4.12.45_on_Nutanix(IPI)
date: 2024-03-04T08:15:21+08:00
categories:
    - Lab Category
tags:
    - Kubernetes
    - Openshift
    - Nutanix
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# OCP4.12.45 on Nutanix - IPI 

Using IPI method install ocp 4.12.45 ( Stable )

大致步驟

1. 建立Bastion
2. 建立Nutanix Prism Central CA
3. 建立Install-config.yaml
4. 建立OCP Cluster
5. Nutanix CSI Operator
6. Worker node autoscale



## Reference

Disk Resize:

https://www.geekersdigest.com/how-to-extend-grow-linux-file-systems-without-downtime/#using-disk-partitions-vm

CA:

https://www.nutanix.dev/2023/09/26/generating-a-self-signed-san-certificate-to-use-red-hat-openshift-in-nutanix-marketplace/

Installation:

https://www.youtube.com/watch?v=G8fFB6EUiOA

https://docs.openshift.com/container-platform/4.12/installing/installing_nutanix/installing-restricted-networks-nutanix-installer-provisioned.html

https://www.nutanix.dev/2022/08/16/red-hat-openshift-ipi-on-nutanix-cloud-platform/

https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html

https://blog.csdn.net/weixin_43902588/article/details/107727804

https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.45/

![image-20231214105046158](https://kenkenny.synology.me:5543/images/2023/12/image-20231214105046158.png)



## Cluster Information

Prism Central: IP + DNS FQDN

Prism Element: IP + DNS FQDN


iSCSI Data Service: IP

DNS Server: IP

NTP: IP

Network Subnet:  Name
VM Network IPAM:  IP Range

---

bastion: IP

OCP Cluster Name: poc

OCP Cluster Base Domain: Domain Name

OCP API VIP: IP + DNS FQDN

OCP Ingress VIP: IP + DNS FQDN



## Create Bastion VM rhel9.3

1. 從Prism建立rhel 9.3 VM，配置vCPU, Memory, Disk, Network，掛rhel 9.3 的 iso檔案並開機

   ![image-20231228092100302](https://kenkenny.synology.me:5543/images/2023/12/image-20231228092100302.png)

   ![image-20231228092139601](https://kenkenny.synology.me:5543/images/2023/12/image-20231228092139601.png)

2. 開機後點選Console, 設定基本的帳號密碼、網路、硬碟等

   ![image-20231214143007168](https://kenkenny.synology.me:5543/images/2023/12/image-20231214143007168.png)

3. 安裝完成ssh 進入bastion

   

4. 下載ccoctl、openshift-install、oc client

   ```
   $ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.45/ccoctl-linux-4.12.45.tar.gz
   
   $ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.45/openshift-client-linux-4.12.45.tar.gz
   
   $ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.45/openshift-install-linux-4.12.45.tar.gz
   ```

   ![image-20231221102507731](https://kenkenny.synology.me:5543/images/2023/12/image-20231221102507731.png)

5. 解壓縮下載的bin檔，並放到/usr/local/bin、建立ocp-install資料夾

   ```
   $ tar xvf ccoctl-linux-4.12.45.tar.gz 
   
   $ tar xvf openshift-client-linux-4.12.45.tar.gz 
   
   $ tar xvf openshift-install-linux-4.12.45.tar.gz 
   
   $ chmod 775 ccoctl
   
   $ sudo cp ccoctl oc kubectl openshift-install /usr/local/bin
   
   $ mkdir ocp-install
   
   $ cp openshift-install ocp-install/
   
   $ oc version
   Client Version: 4.12.45
   Kustomize Version: v4.5.7
   ```

   ![image-20231221102521345](https://kenkenny.synology.me:5543/images/2023/12/image-20231221102521345.png)

   ![image-20231221102532790](https://kenkenny.synology.me:5543/images/2023/12/image-20231221102532790.png)

6. 下載pull-secret from registry.redhat.io

   ```
   {"auths":{"cloud.openshift.com":{"auth":.....}}}
   ```

7. Create SSH Key

   ```
   ssh-keygen
   ```

   ![image-20231221102325288](https://kenkenny.synology.me:5543/images/2023/12/image-20231221102325288.png)



## Prism Central CA

1. 產生CA

```
$ export PC_IP= PC_IP
$ export PC_FQDN= PC_FQDN

$ openssl req -x509 -nodes -days 3650 \
-newkey rsa:2048 -keyout ${PC_IP}.key -out ${PC_IP}.crt \
-subj "/C=US/ST=CA/L=San Jose/O=Nutanix Inc./OU=Manageability/CN=*.nutanix.local" \
-addext "subjectAltName=IP:${PC_IP},DNS:${PC_FQDN}"
```

![image-20231221102352813](https://kenkenny.synology.me:5543/images/2024/03/image-20231221102352813.png)

2. 置換Prism Central CA

   ![image-20231221102428912](https://kenkenny.synology.me:5543/images/2024/03/image-20231221102428912.png)

   ![image-20231221102436959](https://kenkenny.synology.me:5543/images/2024/03/image-20231221102436959.png)

   ![image-20231221102446241](https://kenkenny.synology.me:5543/images/2024/03/image-20231221102446241_.png)

   ![image-20231221102454156](https://kenkenny.synology.me:5543/images/2024/03/image-20231221102454156.png)

3. 加入CA至Bastion

   ```
   $ sudo cp xxx.crt /etc/pki/ca-trust/source/anchors
   $ sudo cp xxx.key /etc/pki/ca-trust/source/anchors
   $ sudo update-ca-trust extract
   ```
   

## Install OCP

### Requirements

1. Storage Space: 800GB

2. 1 Cluster VIP for load balancer on control plane

3. 1 Ingress VIP for load balancer on worker node

4. DNS record : 
   vip.(cluster name).(base domain)
   *.apps.(cluster name).(base domain)

5. IPAM with 7 ip addresses.

6. Virtual machines:

- 1 temporary bootstrap node
- 3 control plane nodes
- 3 compute machines
- 1 installation bastion machines 

7. SSH Key
8. download oc , ccoctl, openshift-install, 
9. CoreOS image on web server or Nutanix Object
10. Prism Central CA 



### Create Install-config.yaml

利用openshift-install建立install-config.yaml檔案，依序輸入ssh key、Nutanix資訊、OCP Cluster資訊

```
$ ./openshift-install create install-config --dir <installation_directory> 
```

![image-20231221102751103](https://kenkenny.synology.me:5543/images/2024/03/image-20231221102751103.png)

#### 修改 install-config.yaml

additionalTrustBundle: 加入 Prism Central 的 root CA

additionalTrustBundlePolicy: 改為 Always

machineNetwork: 改為與VM相同網段 xxx.xxx.xxx.xxx/24

platform 自定義cpu、memory、disk

完成後如下，建議備份此檔案以利部署到其他地方或是重新部署

```
apiVersion: v1
baseDomain: XXX.XXX
additionalTrustBundle: |
    -----BEGIN CERTIFICATE-----
    MIID8zCCAtugAwIBAgIUPE0LteDWc8JLjEm5LlqJF0s17n0wDQYJKoZIhvcNAQEL
    XXX
    -----END CERTIFICATE-----
additionalTrustBundlePolicy: Always
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    nutanix:
      cpus: 4
      coresPerSocket: 2
      memoryMiB: 16384
      osDisk:
        diskSizeGiB: 120
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: 
    nutanix:
      cpus: 4
      coresPerSocket: 2
      memoryMiB: 16384
      osDisk:
        diskSizeGiB: 120
  replicas: 3
credentialsMode: Manual
metadata:
  creationTimestamp: null
  name: poc
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: xxx.xxx.xxx.xxx/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  nutanix:
    apiVIPs:
    - xxx.xxx.xxx.xxx
    ingressVIPs:
    - xxx.xxx.xxx.xxx
    prismCentral:
      endpoint:
        address: xxx.xxx.xxx.xxx
        port: 9440
      password: password
      username: admin
    prismElements:
    - endpoint:
        address: xxx.xxx.xxx.xxx
        port: 9440
      uuid: xxxxxxx-cd36-0cc8-0f4a-xxxxxxxxxx
    subnetUUIDs:
    - xxxxxxxx-27ab-46ab-a0b1-xxxxxxxxxx3
publish: External
pullSecret: '{"auths":.....}}}'
sshKey: |
  ssh-rsa AAAAxxxxxxxxxxxxxxxxxxxxxxx nutanix@ocp-bastion

```



###  設定CCO 

安裝Cluster需要Cloud Credential Operator (CCO)，讓OCP能與Prism溝通

大致作法為

1. 建立帳號密碼檔案
2. 建立要求帳密的目錄及檔案
3. 利用上述檔案建立OCP Shared Secret



1. 在 ocp-install 資料夾內建立 credentials.yaml

   ```
   $ vim credentials.yaml
   
   credentials:
   - type: basic_auth
     data:
       prismCentral:
         username: admin
         password: P@ssw0rd123
       prismElements:
       - name: AHV-Cluster
         username: admin
         password: P@ssw0rd123
   ```

   

2. 設定Image環境變數

   ```
   $ RELEASE_IMAGE=$(./openshift-install version | awk '/release image/ {print $3}')
   ```

   

3. 建立credrequests資料夾

   ```
   $ mkdir credrequests
   ```

   

4. 建立0000_30_machine-api-operator_00_credentials-request.yaml

   ```
   $ oc adm release extract --credentials-requests --cloud=nutanix --to=./credrequests  $RELEASE_IMAGE
   ```

   

5. 建立 CCO Secret

   ```
   $ ccoctl nutanix create-shared-secrets --credentials-requests-dir=/home/nutanix/ocp_install/credrequests --output-dir=/home/nutanix/ocp-install --credentials-source-filepath=/home/nutanix/ocp_install/credentials.yaml
   ```

   ![image-20231221103631965](https://kenkenny.synology.me:5543/images/2024/03/image-20231221103631965.png)

   ![image-20231221103640795](https://kenkenny.synology.me:5543/images/2023/12/image-20231221103640795.png)

   ![image-20231221103812824](https://kenkenny.synology.me:5543/images/2023/12/image-20231221103812824.png)

   ![image-20231221103830445](https://kenkenny.synology.me:5543/images/2023/12/image-20231221103830445.png)

### Create Cluster

#### Create Manifests

建立 manifests資料夾，確認openshift-machine-api-nutanix-credentials-credentials.yaml有在裡面

```
$ ./openshift-install create manifests --dir /home/nutanix/ocp-install/
```

![image-20231221103956961](https://kenkenny.synology.me:5543/images/2023/12/image-20231221103956961.png)



#### Create Cluster

1. ocp-install

   ```
   $ ./openshift-install create cluster --dir /home/nutanix/ocp-install/ --log-level=debug
   ```

   ![image-20231221104222075](https://kenkenny.synology.me:5543/images/2023/12/image-20231221104222075.png)

2. 安裝過程，會先建立OS image

   ![image-20231221104256996](https://kenkenny.synology.me:5543/images/2023/12/image-20231221104256996.png)

   ![image-20231221104311871](https://kenkenny.synology.me:5543/images/2024/03/image-20231221104311871.png)

3. install completed

   ![image-20231228101028495](https://kenkenny.synology.me:5543/images/2024/03/image-20231228101028495.png)

4. 安裝完後在安裝目錄底下會有auth的資料夾，裡面含有kubeadmin的帳密、kubeconfig檔案

   建議將kubeconfig檔案複製到~/.kube/config ，這樣就可以直接使用 oc 指令連線到該叢集

   ```
   in /home/nutanix/ocp-install
   
   $ mkdir ~/.kube
   $ cp auth/kubeconfig ~/.kube/config
   ```

   

5. oc get nodes

   ![image-20231228101001702](https://kenkenny.synology.me:5543/images/2024/03/image-20231228101001702.png)

6. get console url

   ![image-20231228101053013](https://kenkenny.synology.me:5543/images/2024/03/image-20231228101053013.png)

7. login page，密碼在/home/nutanix/ocp-install/auth/

   ![image-20231228101115835](https://kenkenny.synology.me:5543/images/2024/03/image-20231228101115835.png)

8. login

   ![image-20231228101128534](https://kenkenny.synology.me:5543/images/2024/03/image-20231228101128534.png)

   ![image-20231228101140167](https://kenkenny.synology.me:5543/images/2024/03/image-20231228101140167.png)

9. 安裝完有遇到預設ntp無法連通，需改為內部的ntp server
   修改MachineConfig方法為撰寫bu設定檔，再透過bu去轉成yaml檔案，最後部署上去OCP

   ![image-20231229120824457](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120824457.png)



## Nutanix CSI

1. 在OCP Console上點選OperatorHub，搜尋Nutanix

   ![image-20231229112011014](https://kenkenny.synology.me:5543/images/2024/03/image-20231229112011014.png)

2. 點進去點選Install，描述內容比較重要的像支援Nutanix Volumes (rwo)、Nutanix FIles (rwx)

   以及下方有部署yaml範例，可以先複製下來

   ![image-20231229112814864](https://kenkenny.synology.me:5543/images/2024/03/image-20231229112814864.png)

3. 選安裝版本、Namespace、更新方式

   ![image-20231229113026621](https://kenkenny.synology.me:5543/images/2024/03/image-20231229113026621.png)

4. 安裝完成

   ![image-20231229113056589](https://kenkenny.synology.me:5543/images/2024/03/image-20231229113056589.png)

5. 點選Nutanix CSI Storage，點Create NutanixCsiStorage

   ![image-20231229113122565](https://kenkenny.synology.me:5543/images/2024/03/image-20231229113122565.png)

6. 建立NutanixCsiStorage，點選Create即可

   ![image-20231229113155776](https://kenkenny.synology.me:5543/images/2024/03/image-20231229113155776.png)

   ![image-20231229113229946](https://kenkenny.synology.me:5543/images/2024/03/image-20231229113229946.png)

7. 建立一個 Secret 讓OCP可以透過該Secret與Prism做溝通

   ```
   $ vi ntnx-secret
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: ntnx-secret
     namespace: openshift-cluster-csi-drivers
   stringData:
     # prism-element-ip:prism-port:admin:password
     key: PE-IP:9440:admin:P@ssw0rd123
   ```

   ![image-20231229113317508](https://kenkenny.synology.me:5543/images/2024/03/image-20231229113317508.png)

8. 建立Volume的Storage Class ，需修改dataServiceEndPoint、storageContainer

   ```
   $ vi nutanix-volume.yaml
   
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: nutanix-volume
   provisioner: csi.nutanix.com
   parameters:
     csi.storage.k8s.io/provisioner-secret-name: ntnx-secret
     csi.storage.k8s.io/provisioner-secret-namespace: openshift-cluster-csi-drivers
     csi.storage.k8s.io/node-publish-secret-name: ntnx-secret
     csi.storage.k8s.io/node-publish-secret-namespace: openshift-cluster-csi-drivers
     csi.storage.k8s.io/controller-expand-secret-name: ntnx-secret
     csi.storage.k8s.io/controller-expand-secret-namespace: openshift-cluster-csi-drivers
     csi.storage.k8s.io/fstype: ext4
     dataServiceEndPoint: IP:3260 # Prism data service ip
     storageContainer: default-container # Prism 上建立的Storage Container
     storageType: NutanixVolumes
     #whitelistIPMode: ENABLED
     #chapAuth: ENABLED
   allowVolumeExpansion: true
   reclaimPolicy: Delete
   ```

   ![image-20231229113507440](https://kenkenny.synology.me:5543/images/2024/03/image-20231229113507440.png)

9. 建立Nutanix Files的Storage Class

   有分為兩種：NFS Servers、Dynamic NFS Shares (Valid Design)

   NFS Share Server：

   ```
   $ vi ntnx-files.yaml
   
   
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
       name: nutanix-files
       annotations:
           storageclass.kubernetes.io/is-default-class: "false"
   provisioner: csi.nutanix.com
   parameters:
       nfsServer: files.poc.xxx.xxx.xxx
       nfsPath: /ocp
       storageType: NutanixFiles
       squashType: "root-squash" # 若沒有資安疑慮可以用none，比較簡單
   reclaimPolicy: Delete or Retain
   ```

   ![image-20231229114237696](https://kenkenny.synology.me:5543/images/2024/03/image-20231229114237696.png)

   Dynamic NFS Shares

   ```
   $ vi ntnx-dynamic-files.yaml
   
   
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
       name: nutanix-dynfiles
   provisioner: csi.nutanix.com
   parameters:
       dynamicProv: ENABLED
       csi.storage.k8s.io/provisioner-secret-name: ntnx-secret
       csi.storage.k8s.io/provisioner-secret-namespace: openshift-cluster-csi-drivers
       csi.storage.k8s.io/node-publish-secret-name: ntnx-secret
       csi.storage.k8s.io/node-publish-secret-namespace: openshift-cluster-csi-drivers
       csi.storage.k8s.io/controller-expand-secret-name: ntnx-secret
       csi.storage.k8s.io/controller-expand-secret-namespace: openshift-cluster-csi-drivers
       storageType: NutanixFiles
       squashType: "root-squash" # 若沒有資安疑慮可以用none，比較簡單
       nfsServerName: files # 對應Nutanix Files Server的名稱
   allowVolumeExpansion: true
   ```

   ![image-20231229114932023](https://kenkenny.synology.me:5543/images/2024/03/image-20231229114932023.png)

   

   兩者建PVC在Nutanix Files Console上的差異如下：

   選擇第一種會建立在ocp這個share內，選擇第二種會建立各個PVC對應的share

   ![image-20231229120003104](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120003104.png)

   

10. 建立PVC去確認Status為Bound即可，Volume測試

    ![image-20231229120126695](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120126695.png)

    在Nutanix Prism上的Volume Group

    ![image-20231229120205001](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120205001.png)

11. 擴充PVC

    ![image-20231229120324056](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120324056.png)

    ![image-20231229120337732](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120337732.png)

    ![image-20231229120352157](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120352157.png)

    ![image-20231229120408383](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120408383.png)

12. 建立Files PVC測試

    ![image-20231229120613807](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120613807.png)

    ![image-20231229120634249](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120634249.png)

13. 檢視PV確認都為Bound

    ![image-20231229120448778](https://kenkenny.synology.me:5543/images/2024/03/image-20231229120448778.png)

## Autoscaler

Nutanix 結合Openshift的Autoscaler做到Worker Node的自動擴展

名詞解釋：

Machine：為透過Openshift部署的機器

Nodes：為部署完加入OCP Cluster內的節點

scale up：對於Openshift 來說是整體的Resource增加，以實際看到的意思就是worker node新增，因為master node不能被使用

scale down：相對scale up 就是整體的Resource減少，意思就是減少worker node的數量

Clusterautoscaler：針對整個Cluster做Resource管控的設定，例如整體CPU、Memory、整體最大節點等

Machinesets：machine的設定值，包含有多種不同的環境例如 AWS、GCP、Nutanix，就會有多個Machinesets

Machineautoscaler：針對Machinesets去做scale的設定，例如最大12個節點、最小2個節點等



### Config

1. 確認現有的Machinesets、Machine數量

   ```
   在Bastion上
   $ oc get machinesets -n openshift-machine-api
   $ oc get machine -n openshift-machine-api
   ```

   ![image-20231229132903201](https://kenkenny.synology.me:5543/images/2023/12/image-20231229132903201.png)

   

2. 新增ClusterAutoscaler設定檔，這邊還可自定義Cluster的Resource資源總共多少、Woker Node使用量到多少時需要擴展
   預設是單一Work Node 使用量達50%時會自動擴展，詳情見Redhat 官方連結
   https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html

   ```
   $ vi clusterautoscaler.yaml
   
   apiVersion: "autoscaling.openshift.io/v1"
   kind: "ClusterAutoscaler"
   metadata:
     name: "default"
   spec:
     resourceLimits:
       maxNodesTotal: 10 # 最大的worker node數量
     scaleDown: # scaleDown 參數
       enabled: true
       delayAfterAdd: 10s
       delayAfterDelete: 10s
       delayAfterFailure: 10s
   ```

   ![image-20231229133058122](https://kenkenny.synology.me:5543/images/2024/03/image-20231229133058122.png)

3. 新增MachineAutoscaler設定檔

   ```
   $ vi machineautoscaler.yaml
   
   apiVersion: "autoscaling.openshift.io/v1beta1"
   kind: "MachineAutoscaler"
   metadata:
     name: "poc-gqwvb-worker" # 這裡請對應machineset的名稱
     namespace: "openshift-machine-api"
   spec:
     minReplicas: 1 
     maxReplicas: 12 
     scaleTargetRef: 
       apiVersion: machine.openshift.io/v1beta1
       kind: MachineSet 
       name: poc-gqwvb-worker # 這裡請對應machineset的名稱
   ```

   ![image-20231229133331596](https://kenkenny.synology.me:5543/images/2024/03/image-20231229133331596.png)

4. 部署兩個設定檔案到ocp cluster內，即可看到MachineAutoscalers內有該設定檔

   ```
   $ oc create -f clusterautoscaler.yaml 
   clusterautoscaler.autoscaling.openshift.io/default created
   
   $ oc create -f machineautoscaler.yaml 
   machineautoscaler.autoscaling.openshift.io/ocp-demo-t82dn-worker created
   ```

   ![image-20231229133835454](https://kenkenny.synology.me:5543/images/2024/03/image-20231229133835454.png)



### Test

現有Machine及Nodes

Machine

![image-20231229134502956](https://kenkenny.synology.me:5543/images/2024/03/image-20231229134502956.png)

Nodes

![image-20231229134439968](https://kenkenny.synology.me:5543/images/2024/03/image-20231229134439968.png)



1. 建立測試用的Project (namespace)

   ```
   $ oc adm new-project autoscale-example && oc project autoscale-example
   ```

   

2. 建立測試的Pod，50個pod、480秒後自動關閉

   ```
   $ vi demo-job.yaml
   
   apiVersion: batch/v1
   kind: Job
   metadata:
     generateName: demo-job-
   spec:
     template:
       spec:
         containers:
         - name: work
           image: busybox
           command: ["sleep",  "480"]
           resources:
             requests:
               memory: 1000Mi
               cpu: 1000m
         restartPolicy: Never
     backoffLimit: 4
     completions: 50
     parallelism: 50
   ```

   

3. 部署到OCP Cluster內

   ```
   $ oc apply -f demo-job.yaml
   ```

   ![](https://kenkenny.synology.me:5543/images/2024/03/image-20231229134951317.png)

   ![image-20231229135114899](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135114899.png)

4. 當OCP的autoscaler偵測到時，即會自動開始新增worker node (scale up 約10分鐘)
   可在Machines的頁面查看 

   ![image-20231229135203313](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135203313.png)

5. Prism Central上也可以看到VM在建立中

   ![image-20231229135249675](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135249675.png)

6. Machinesets 會看到應該有的machines要有4個現在只有3個

   ![image-20231229135323816](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135323816.png)

7. Machine Provision完成後，會自動加入OCP Cluster，自動安裝叢集所需的Pod、CSI Driver等

   ![image-20231229135432143](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135432143.png)

8. Node加入完成

   ![image-20231229135533396](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135533396.png)

   ![image-20231229135557129](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135557129.png)

   ![image-20231229135618865](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135618865.png)

   ![image-20231229135634076](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135634076.png)

9. 當Pod的時間到了自動關閉之後，即會開始做縮減的動作 (scale down 約3分鐘完成 )

   OCP會自動選擇其中一個Worker Node做刪除，會排除該節點上有local PV、使用量低於50%的Worker Node

   ![image-20231229140147396](https://kenkenny.synology.me:5543/images/2024/03/image-20231229140147396.png)

   ![image-20231229135806745](https://kenkenny.synology.me:5543/images/2024/03/image-20231229135806745.png)

10. 最後回歸為原本的狀態

    ![image-20231229140933239](https://kenkenny.synology.me:5543/images/2024/03/image-20231229140933239.png)

    ![image-20231229140947045](https://kenkenny.synology.me:5543/images/2024/03/image-20231229140947045.png)

以上為使用Nutanix 透過 IPI Online的方式安裝Red Hat Openshift 叢集的測試過程。
