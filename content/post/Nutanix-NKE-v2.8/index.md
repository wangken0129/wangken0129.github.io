---
title: Nutanix-NKE-v2.8
description: Nutanix-NKE-v2.8
slug: Nutanix-NKE-v2.8
date: 2023-10-11T05:34:51+08:00
categories:
    - Lab Category
tags:
    - Kubernetes
    - Nutanix
    - NKE
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Nutanix NKE (Karbon) 實作



## 前言

此測試主要目的為安裝NKE、測試CSI Driver、驗證服務及升級，

並且能夠執行擴充、刪減、升級等操作，皆使用非離線的方式完成。



## 版本、資訊

PC2023.3 預設NKE版本為v2.2.3，已先升級到v2.8.0，以測試更高的Kubernetes版本。

參考連結：https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Engine-v2_8:Nutanix-Kubernetes-Engine-v2_8

Port需求：

https://portal.nutanix.com/page/documents/list?type=software&filterKey=software&filterVal=Ports%20and%20Protocols&productType=Nutanix%20Kubernetes%20Engine

Calico vs Flannel (Calico不支援Node在不同Vlan)：

https://www.modb.pro/db/152417

Shell：https://blog.csdn.net/low5252/article/details/103646475



版本：

AOS: 6.5.3.6

Prism Central: 2023.3

NKE:  2.8.0

Kubernetes: 1.24.10



架構：

共會有八個VM：Control Plane *2 、etcd *3 、Worker Node *3

NKE 運作架構

![The Nutanix Bible](https://kenkenny.synology.me:5543/images/2023/10/nke_multi-cluster_onprem_diagram_v2.svg)



## 安裝步驟

### Check NKE Enabled

![image-20231002180337566](https://kenkenny.synology.me:5543/images/2023/10/image-20231002180337566.png)

### Check NKE Version

NKE: v2.7.0

![image-20231002180455854](https://kenkenny.synology.me:5543/images/2023/10/image-20231002180455854.png)

v2.7.0 UI出不來，故升級到v2.8.0

![image-20231003113901890](https://kenkenny.synology.me:5543/images/2023/10/image-20231003113901890.png)

![image-20231003113957749](https://kenkenny.synology.me:5543/images/2023/10/image-20231003113957749.png)

### Download OS Image

下載的是ntnx-1.5

![image-20231003114547878](https://kenkenny.synology.me:5543/images/2023/10/image-20231003114547878.png)

### Create Cluster

1. 配置IPAM

   ![image-20231003095321526](https://kenkenny.synology.me:5543/images/2023/10/image-20231003095321526.png)

2. 到Kubernetes Management --> Create Cluster --> Production Cluster --> Next

   ![image-20231003130720675](https://kenkenny.synology.me:5543/images/2023/10/image-20231003130720675.png)

   ![image-20231003114440806](https://kenkenny.synology.me:5543/images/2023/10/image-20231003114440806.png)

3. 輸入Cluster Name、版本等資訊，先安裝1.24.10

   ![image-20231003134121155](https://kenkenny.synology.me:5543/images/2023/10/image-20231003134121155.png)

4. 選擇Node的子網，Additional可不選，Woker Node數量

   Control Plane可選擇External Loadbalance or Active-Passive，

   前者需建立好Loadbalance，可擴充性較高，

   後者只能使用兩個Control Plane，作為主要跟備援的Node，且要提供一個VIP在IPAM的管理範圍外面。

   ![image-20231003135441332](https://kenkenny.synology.me:5543/images/2023/10/image-20231003135441332.png)

5. etcd的數量

   ![image-20231003135458249](https://kenkenny.synology.me:5543/images/2023/10/image-20231003135458249.png)

6. 選擇Kubenetes網路 Calico or Flannel ，兩者比較在上面的連結，這裡選擇Flannel 其他預設

   ![image-20231003135128065](https://kenkenny.synology.me:5543/images/2023/10/image-20231003135128065.png)

7. 選擇預設的Storage Class ，預設的會使用iSCSI Data Service的IP去連結Nutanix Cluster的Volume空間

   後續有需要NFS可以再加入，其他預設即可點選Create

   ![image-20231003135239648](https://kenkenny.synology.me:5543/images/2023/10/image-20231003135239648.png)

8. 之後會開始自動安裝NKE Cluster

   ![image-20231003135634766](https://kenkenny.synology.me:5543/images/2023/10/image-20231003135634766.png)

9. 順序大概是 Create etcd VM --> Create etcd Cluster --> Create Master VM --> Create Worker VM --> Create NKE Cluster

   會自動建立一些PVC給Cluster內部使用

   ![image-20231003140541856](https://kenkenny.synology.me:5543/images/2023/10/image-20231003140541856.png)

   <img src="https://kenkenny.synology.me:5543/images/2023/10/image-20231003141114844.png" alt="image-20231003141114844" style="zoom:50%;" />

10. 建立的VM 以及VIP已掛在Master Node上面

    ![image-20231003141434688](https://kenkenny.synology.me:5543/images/2023/10/image-20231003141434688.png)

11. 約15分鐘Cluster就建立完成了

    ![image-20231003141957766](https://kenkenny.synology.me:5543/images/2023/10/image-20231003141957766.png)

12. 回到Kubenetes Management 可看到NKE Cluster建立完成，Status: Healthy

    ![image-20231003142114057](https://kenkenny.synology.me:5543/images/2023/10/image-20231003142114057.png)

    ![image-20231003142215697](https://kenkenny.synology.me:5543/images/2023/10/image-20231003142215697.png)



## 驗證 NKE 叢集

### 介面

Node Pools：Worker可以再加，etcd跟 Control Plane是不能再新增

Woker Node

![image-20231003144146664](https://kenkenny.synology.me:5543/images/2023/10/image-20231003144146664.png)

Control Plane Node

![image-20231003142534886](https://kenkenny.synology.me:5543/images/2023/10/image-20231003142534886.png)

etcd Node

![image-20231003142549532](https://kenkenny.synology.me:5543/images/2023/10/image-20231003142549532.png)

Storage Class為預設的 Nutanix Volume

![image-20231003142618575](https://kenkenny.synology.me:5543/images/2023/10/image-20231003142618575.png)

### Enter Cluster

#### SSH Access：

此方法可直接進入被鎖定的節點上面，下載.sh檔案並執行，裡面會內含Private Key

![image-20231003142742864](https://kenkenny.synology.me:5543/images/2023/10/image-20231003142742864.png)

#### Download Kubeconfig：

下載Kubeconfig並設定環境變數，需要安裝kubectl

此檔案可能會自動rotate certicifate，記得再重新下載即可

![image-20231003142830309](https://kenkenny.synology.me:5543/images/2023/10/image-20231003142830309.png)

1. 安裝Kubectl ( Mac )

   ```shell
   wangken@wangken-MAC ken-nke1 % curl -LO "https://dl.k8s.io/release/v1.24.10/bin/darwin/amd64/kubectl"
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
   100   138  100   138    0     0    518      0 --:--:-- --:--:-- --:--:--   532
   100 50.2M  100 50.2M    0     0   717k      0  0:01:11  0:01:11 --:--:--  984k
   
   wangken@wangken-MAC ken-nke1 % ll
   total 104600
   drwxr-xr-x   5 wangken  staff       160 10  3 14:52 .
   drwxr-xr--+ 73 wangken  staff      2336 10  3 14:30 ..
   -rw-r--r--@  1 wangken  staff      3497 10  3 14:28 ken-nke1-kubectl.cfg
   -rwxr-xr-x@  1 wangken  staff      4538 10  3 14:26 ken-nke1-ssh-access.sh
   -rw-r--r--   1 wangken  staff  52739872 10  3 14:45 kubectl
   
   wangken@wangken-MAC ken-nke1 % chmod +x kubectl 
   
   wangken@wangken-MAC ken-nke1 % ./kubectl version
   WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
   Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.10", GitCommit:"5c1d2d4295f9b4eb12bfbf6429fdf989f2ca8a02", GitTreeState:"clean", BuildDate:"2023-01-18T19:15:31Z", GoVersion:"go1.19.5", Compiler:"gc", Platform:"darwin/amd64"}
   Kustomize Version: v4.5.4
   The connection to the server localhost:8080 was refused - did you specify the right host or port?
   ```

2. 可直接執行或是把kubectl 丟到 /usr/local/bin

3. 直接執行需要帶入 --kubeconfig + 下載的config檔案

   ```
   wangken@wangken-MAC ken-nke1 % ./kubectl get nodes --kubeconfig ken-nke1-kubectl.cfg 
   NAME                       STATUS   ROLES                  AGE   VERSION
   ken-nke1-906a07-master-0   Ready    control-plane,master   55m   v1.24.10
   ken-nke1-906a07-master-1   Ready    control-plane,master   54m   v1.24.10
   ken-nke1-906a07-worker-0   Ready    node                   51m   v1.24.10
   ken-nke1-906a07-worker-1   Ready    node                   51m   v1.24.10
   ken-nke1-906a07-worker-2   Ready    node                   51m   v1.24.10
   ```

4. 另一種就是把kubectl 放到預設指令執行的地方，可用echo $PATH去找

   再輸入export KUBECONFIG=下載的kubeconfig 存放的路徑即可直接執行kubectl get nodes等指令

   ```
   wangken@wangken-MAC ken-nke1 % echo $PATH
   /Library/Frameworks/Python.framework/Versions/3.11/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/X11/bin:/Library/Apple/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin
   ```

5. 因為我不想把kubectl放到系統裡面(日後可能還要更新)，所以我直接做一個執行檔案來連線此NKE叢集

   日後就透過nke-ctl這個執行檔來連線

   ```
   wangken@wangken-MAC ken-nke1 % cat nke-ctl 
   #!/bin/bash
   
   ./kubectl --kubeconfig=./ken-nke1-kubectl.cfg $*
   ```

   ```
   範例
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get nodes
   NAME                       STATUS   ROLES                  AGE   VERSION
   ken-nke1-906a07-master-0   Ready    control-plane,master   79m   v1.24.10
   ken-nke1-906a07-master-1   Ready    control-plane,master   78m   v1.24.10
   ken-nke1-906a07-worker-0   Ready    node                   75m   v1.24.10
   ken-nke1-906a07-worker-1   Ready    node                   75m   v1.24.10
   ken-nke1-906a07-worker-2   Ready    node                   75m   v1.24.10
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get storageclass   
   NAME                             PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   default-storageclass (default)   csi.nutanix.com   Delete          Immediate           true                   74m
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get ns          
   NAME              STATUS   AGE
   default           Active   84m
   kube-node-lease   Active   84m
   kube-public       Active   84m
   kube-system       Active   84m
   ntnx-system       Active   79m
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get pod -n ntnx-system
   NAME                                        READY   STATUS    RESTARTS   AGE
   alertmanager-main-0                         2/2     Running   0          76m
   alertmanager-main-1                         2/2     Running   0          76m
   alertmanager-main-2                         2/2     Running   0          76m
   csi-snapshot-controller-5fc6f8f5c4-7bq96    1/1     Running   0          79m
   csi-snapshot-webhook-78957b8ddd-2c9n8       1/1     Running   0          79m
   fluent-bit-67vdl                            1/1     Running   0          79m
   fluent-bit-6kxjz                            1/1     Running   0          79m
   fluent-bit-fgnrx                            1/1     Running   0          79m
   fluent-bit-gffxw                            1/1     Running   0          79m
   fluent-bit-rs5n8                            1/1     Running   0          79m
   kube-state-metrics-7bcb7f8cb6-tkjwf         3/3     Running   0          76m
   kubernetes-events-printer-8d4c49454-m9ss6   1/1     Running   0          79m
   node-exporter-9gtjl                         2/2     Running   0          76m
   node-exporter-dl7k8                         2/2     Running   0          76m
   node-exporter-g7jw7                         2/2     Running   0          76m
   node-exporter-mv6xt                         2/2     Running   0          76m
   node-exporter-nvk6w                         2/2     Running   0          76m
   nutanix-csi-controller-864cc4767b-gl2sx     5/5     Running   0          79m
   nutanix-csi-node-8c2cc                      3/3     Running   0          79m
   nutanix-csi-node-sf6wd                      3/3     Running   0          79m
   nutanix-csi-node-shkkl                      3/3     Running   0          79m
   prometheus-k8s-0                            2/2     Running   0          76m
   prometheus-k8s-1                            2/2     Running   0          76m
   prometheus-operator-59d64b7bc7-t8slv        2/2     Running   0          76m
   
   kube-system
   kube-system   coredns-7b95cd4468-nv7jk                       1/1     Running   0          2d1h
   kube-system   coredns-7b95cd4468-xxwgf                       1/1     Running   0          2d1h
   kube-system   kube-apiserver-ken-nke1-906a07-master-0        3/3     Running   0          2d1h
   kube-system   kube-apiserver-ken-nke1-906a07-master-1        3/3     Running   0          2d1h
   kube-system   kube-flannel-ds-4r6j2                          1/1     Running   0          2d1h
   kube-system   kube-flannel-ds-dp5jv                          1/1     Running   0          2d1h
   kube-system   kube-flannel-ds-gqz5j                          1/1     Running   0          2d1h
   kube-system   kube-flannel-ds-slmqt                          1/1     Running   0          2d1h
   kube-system   kube-flannel-ds-z7hnh                          1/1     Running   0          2d1h
   kube-system   kube-proxy-ds-6x8c2                            1/1     Running   0          2d1h
   kube-system   kube-proxy-ds-892ch                            1/1     Running   0          2d1h
   kube-system   kube-proxy-ds-frrjt                            1/1     Running   0          2d1h
   kube-system   kube-proxy-ds-nkfrm                            1/1     Running   0          2d1h
   kube-system   kube-proxy-ds-pf9xm                            1/1     Running   0          2d1h
   ```

## Nutanix Files 串接

### Files

1. 建立一個File Server，DNS設定 192.168.101.198 > ken-files.nutanixlab.local

   ![image-20231003165718907](https://kenkenny.synology.me:5543/images/2023/10/image-20231003165718907.png)

2. 選擇一個Files VM就好

   ![image-20231003165749736](https://kenkenny.synology.me:5543/images/2023/10/image-20231003165749736.png)

3. 設定Client Network

   <img src="https://kenkenny.synology.me:5543/images/2023/10/image-20231003165917669.png" alt="image-20231003165917669" style="zoom:50%;" />

4. 設定gateway、IP等

   <img src="https://kenkenny.synology.me:5543/images/2023/10/image-20231003170011745.png" alt="image-20231003170011745" style="zoom:50%;" />

5. 設定Storage Network

   <img src="https://kenkenny.synology.me:5543/images/2023/10/image-20231003170149444.png" alt="image-20231003170149444" style="zoom:50%;" />

6. 選擇SMB、NFS Protocol，然後Create

   <img src="https://kenkenny.synology.me:5543/images/2023/10/image-20231003170257630.png" alt="image-20231003170257630" style="zoom:50%;" />

7. 建立完成

   ![image-20231005100336390](https://kenkenny.synology.me:5543/images/2023/10/image-20231005100336390.png)

10. 建立一個NFS Share : ken-nke

    ![image-20231003163145985](https://kenkenny.synology.me:5543/images/2023/10/image-20231003163145985.png)

11. 選擇Authentication: System、Root Squash (後續因redmine需要修改資料夾權限為777，故改為none)

    ![image-20231003163212281](https://kenkenny.synology.me:5543/images/2023/10/image-20231003163212281.png)

12. 建立完後複製Mount Path、Share Path

    ![image-20231003180229922](https://kenkenny.synology.me:5543/images/2023/10/image-20231003180229922.png)

### Create Storage Class

1. 點回Kubenetes Management --> Storage --> Storage Class

   ![image-20231003163529624](https://kenkenny.synology.me:5543/images/2023/10/image-20231003163529624.png)

2. 輸入 Endpoint、Export Path，

   <img src="https://kenkenny.synology.me:5543/images/2023/10/image-20231003180314869.png" alt="image-20231003180314869" style="zoom:50%;" />

3. 建立完成

   ![image-20231003164104978](https://kenkenny.synology.me:5543/images/2023/10/image-20231003164104978.png)

4. 確認Storageclass

   ```
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get storageclass       
   NAME                             PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   default-storageclass (default)   csi.nutanix.com   Delete          Immediate           true                   154m
   ntnx-files                       csi.nutanix.com   Delete          Immediate           false                  3m35s
   ```

5. 建立測試pvc

   ```
   wangken@wangken-MAC ken-nke1 % cat test.pvc.yaml 
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
     name: test-pv-claim
     labels:
       app: test
   spec:
     storageClassName: ntnx-files
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 20Gi
         
   wangken@wangken-MAC ken-nke1 % ./nke-ctl create -f test.pvc.yaml 
   persistentvolumeclaim/test-pv-claim created
   ```

6. 確認為bond狀態代表OK

   ```
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get pvc
   NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   test-pv-claim   Bound    pvc-3f91422b-03d6-469e-940a-da87691be83a   20Gi       RWO            ntnx-files     7s
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl delete -f test.pvc.yaml 
   persistentvolumeclaim "test-pv-claim" deleted
   ```

## 部署服務

之後有其他服務部署上去會再更新。

### 部署redmine

redmine是開源的專案管理軟體，部署時會需要使用PostgreSQL作為Database

部署在redmine的namespace，共會有以下幾個檔案

```shell
wangken@wangken-MAC ken-nke1 % ll
-rw-r--r--   1 wangken  staff       244 10  3 17:30 pg-config.yaml
-rw-r--r--   1 wangken  staff      1106 10  3 17:30 pg-deploy.yaml
-rw-r--r--   1 wangken  staff       255 10  3 17:30 pg-pvc.yaml
-rw-r--r--   1 wangken  staff       222 10  3 15:47 pg-svc.yaml
-rw-r--r--   1 wangken  staff      1070 10  3 16:24 redmine-config.yaml
-rw-r--r--   1 wangken  staff      2461 10  3 17:29 redmine-deploy.yaml
-rw-r--r--   1 wangken  staff       264 10  5 10:05 redmine-svc.yaml
-rw-r--r--   1 wangken  staff       218 10  3 16:40 test.pvc.yaml
```

1. 建立Namespace

   ```
   wangken@wangken-MAC ken-nke1 % ./nke-ctl create ns redmine
   namespace/redmine created
   ```

2. 建立PostgreSQL

   ```
   wangken@wangken-MAC ken-nke1 % ./nke-ctl create -f pg-pvc.yaml -n redmine
   persistentvolumeclaim/redmine-postgres-pv-claim created
   wangken@wangken-MAC ken-nke1 % ./nke-ctl create -f pg-config.yaml -n redmine
   configmap/redmine-postgres-config created
   wangken@wangken-MAC ken-nke1 % ./nke-ctl create -f pg-deploy.yaml -n redmine 
   deployment.apps/redmine-postgres-deployment created
   wangken@wangken-MAC ken-nke1 % ./nke-ctl create -f pg-svc.yaml -n redmine 
   service/redmine-postgres-service created
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get all -n redmine
   NAME                                               READY   STATUS    RESTARTS   AGE
   pod/redmine-postgres-deployment-7bd9859c6d-8c4b5   1/1     Running   0          67s
   
   NAME                               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
   service/redmine-postgres-service   NodePort   172.19.78.32   <none>        5432:31432/TCP   60s
   
   NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/redmine-postgres-deployment   1/1     1            1           67s
   
   NAME                                                     DESIRED   CURRENT   READY   AGE
   replicaset.apps/redmine-postgres-deployment-7bd9859c6d   1         1         1       67s
   ```

3. 驗證PostgreSQL，可以下載 pgAdmin連線確認

   https://www.pgadmin.org/download/pgadmin-4-macos/

   ![image-20231005102945359](https://kenkenny.synology.me:5543/images/2023/10/image-20231005102945359.png)

4. 確認有redmine database存在

   ![image-20231005103031353](https://kenkenny.synology.me:5543/images/2023/10/image-20231005103031353.png)

5. 建立redmine服務，service選用cluster vip，Port 8000

   ```shell
   wangken@wangken-MAC ken-nke1 % ./nke-ctl apply -f redmine-pvc.yaml -n redmine
   persistentvolumeclaim/redmine-files created
   wangken@wangken-MAC ken-nke1 % ./nke-ctl apply -f redmine-config.yaml -n redmine
   configmap/redmine-config created
   wangken@wangken-MAC ken-nke1 % ./nke-ctl apply -f redmine-deploy.yaml -n redmine 
   deployment.apps/redmine created
   wangken@wangken-MAC ken-nke1 % ./nke-ctl apply -f redmine-svc.yaml -n redmine
   service/redmine-service created
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get pvc -n redmine
   NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
   redmine-files               Bound    pvc-fa7253bf-da76-4048-a622-e6b159a94124   1Gi        RWO            ntnx-files             12s
   redmine-postgres-pv-claim   Bound    pvc-532dc4e2-876e-4ebf-b3c5-481226c0eb4f   20Gi       RWO            default-storageclass   24m
   
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get all -n redmine                  
   NAME                                               READY   STATUS    RESTARTS   AGE
   pod/redmine-7fd78679f9-cwxg4                       1/1     Running   0          6m31s
   pod/redmine-postgres-deployment-7bd9859c6d-fv5pb   1/1     Running   0          53m
   
   NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP       PORT(S)          AGE
   service/redmine-postgres-service   NodePort    172.19.193.158   <none>            5432:31432/TCP   53m
   service/redmine-service            ClusterIP   172.19.139.23    192.168.101.199   8000/TCP         2s
   
   NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/redmine                       1/1     1            1           6m31s
   deployment.apps/redmine-postgres-deployment   1/1     1            1           53m
   
   NAME                                                     DESIRED   CURRENT   READY   AGE
   replicaset.apps/redmine-7fd78679f9                       1         1         1       6m31s
   replicaset.apps/redmine-postgres-deployment-7bd9859c6d   1         1         1       53m
   ```

6. 確認config file可以存在ntnx-files這個storageclass

   ![image-20231005153105877](https://kenkenny.synology.me:5543/images/2023/10/image-20231005153105877.png)

7. 網頁連線redmine : http://192.168.101.199:8000  ，預設 admin/admin 再進去修改

   ![image-20231005153419432](https://kenkenny.synology.me:5543/images/2023/10/image-20231005153419432.png)

   ![image-20231005153452116](https://kenkenny.synology.me:5543/images/2023/10/image-20231005153452116.png)

8. 確認OK 

   ![image-20231005155001640](https://kenkenny.synology.me:5543/images/2023/10/image-20231005155001640.png)

9. 後續就可以透由redmine去做專案管理，甚至是結合CI/CD 

## Options & Add-Ons



### Infra Logging

說明：如果沒有Logging stack的話，可以啟用Infra logging (Elasticsearch and Kibana)來使用。

```
Note:

    You can deploy the ELK-based NKE logging stack to provide a simplified experience for users with environments that do not have the logging stack implementation. In environments where the logging implementation exists or where there are no explicit requirements for logging, disable the logging stack to reduce resource consumption.
    The minimum uplink speed recommended is 30 Mb/s.
```

1. 從Prism Central VM 登入Karbon

   ```
   nutanix@NTNX-172-16-90-75-A-PCVM:~$ karbon/karbonctl login --pc-username ken.wang@nutanixlab.local
   Please enter the password for the PC user: ken.wang@nutanixlab.local
   Login successful
   
   nutanix@NTNX-172-16-90-75-A-PCVM:~$ karbon/karbonctl cluster list
   Name        UUID                                    Control Plane IPs (VIP)                               Version      OS Version    Status      Worker IPs
   ken-nke1    906a075d-e27b-47bd-59c3-b71701147650    192.168.101.213, 192.168.101.215 (192.168.101.199)    1.24.10-0    ntnx-1.5      kSuccess    192.168.101.208, 192.168.101.201, 192.168.101.207
   ```

   

2. enable infra logging

   ```
   nutanix@NTNX-172-16-90-75-A-PCVM:~$ karbon/karbonctl cluster infra-logging enable --cluster-name=ken-nke1
   Successfully enabled infra logging: [POST /karbon/v1-alpha.1/k8s/clusters/{name}/enable-infra-logging][200] postEnableInfraLoggingO
   ```

3. 回到NKE Cluster確認 pod status

   ```
   wangken@wangken-MAC ken-nke1 % ./nke-ctl get pod -A
   NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
   kube-system   coredns-7b95cd4468-nv7jk                       1/1     Running   0          2d2h
   kube-system   coredns-7b95cd4468-xxwgf                       1/1     Running   0          2d2h
   kube-system   kube-apiserver-ken-nke1-906a07-master-0        3/3     Running   0          2d2h
   kube-system   kube-apiserver-ken-nke1-906a07-master-1        3/3     Running   0          2d2h
   kube-system   kube-flannel-ds-4r6j2                          1/1     Running   0          2d2h
   kube-system   kube-flannel-ds-dp5jv                          1/1     Running   0          2d2h
   kube-system   kube-flannel-ds-gqz5j                          1/1     Running   0          2d2h
   kube-system   kube-flannel-ds-slmqt                          1/1     Running   0          2d2h
   kube-system   kube-flannel-ds-z7hnh                          1/1     Running   0          2d2h
   kube-system   kube-proxy-ds-6x8c2                            1/1     Running   0          2d2h
   kube-system   kube-proxy-ds-892ch                            1/1     Running   0          2d2h
   kube-system   kube-proxy-ds-frrjt                            1/1     Running   0          2d2h
   kube-system   kube-proxy-ds-nkfrm                            1/1     Running   0          2d2h
   kube-system   kube-proxy-ds-pf9xm                            1/1     Running   0          2d2h
   ntnx-system   alertmanager-main-0                            2/2     Running   0          2d2h
   ntnx-system   alertmanager-main-1                            2/2     Running   0          2d2h
   ntnx-system   alertmanager-main-2                            2/2     Running   0          2d2h
   ntnx-system   csi-snapshot-controller-5fc6f8f5c4-7bq96       1/1     Running   0          2d2h
   ntnx-system   csi-snapshot-webhook-78957b8ddd-2c9n8          1/1     Running   0          2d2h
   ntnx-system   elasticsearch-logging-0                        1/1     Running   0          2m45s
   ntnx-system   fluent-bit-6cjst                               1/1     Running   0          2m43s
   ntnx-system   fluent-bit-74fw9                               1/1     Running   0          2m42s
   ntnx-system   fluent-bit-dn9qc                               1/1     Running   0          2m43s
   ntnx-system   fluent-bit-gwrsq                               1/1     Running   0          2m43s
   ntnx-system   fluent-bit-xdq2n                               1/1     Running   0          2m43s
   ntnx-system   kibana-logging-6ccd475856-cb4dd                1/2     Running   0          2m45s
   ntnx-system   kube-state-metrics-7bcb7f8cb6-tkjwf            3/3     Running   0          2d2h
   ntnx-system   kubernetes-events-printer-8d4c49454-m9ss6      1/1     Running   0          2d2h
   ntnx-system   node-exporter-9gtjl                            2/2     Running   0          2d2h
   ntnx-system   node-exporter-dl7k8                            2/2     Running   0          2d2h
   ntnx-system   node-exporter-g7jw7                            2/2     Running   0          2d2h
   ntnx-system   node-exporter-mv6xt                            2/2     Running   0          2d2h
   ntnx-system   node-exporter-nvk6w                            2/2     Running   0          2d2h
   ntnx-system   nutanix-csi-controller-864cc4767b-gl2sx        5/5     Running   0          2d2h
   ntnx-system   nutanix-csi-node-8c2cc                         3/3     Running   0          2d2h
   ntnx-system   nutanix-csi-node-sf6wd                         3/3     Running   0          2d2h
   ntnx-system   nutanix-csi-node-shkkl                         3/3     Running   0          2d2h
   ntnx-system   prometheus-k8s-0                               2/2     Running   0          2d2h
   ntnx-system   prometheus-k8s-1                               2/2     Running   0          2d2h
   ntnx-system   prometheus-operator-59d64b7bc7-t8slv           2/2     Running   0          2d2h
   redmine       redmine-7fd78679f9-cwxg4                       1/1     Running   0          75m
   redmine       redmine-postgres-deployment-7bd9859c6d-fv5pb   1/1     Running   0          123m
   ```

   

4. 多了以下幾個pod出現

   ```
   ntnx-system   elasticsearch-logging-0                        1/1     Running   0          2m45s
   ntnx-system   fluent-bit-6cjst                               1/1     Running   0          2m43s
   ntnx-system   fluent-bit-74fw9                               1/1     Running   0          2m42s
   ntnx-system   fluent-bit-dn9qc                               1/1     Running   0          2m43s
   ntnx-system   fluent-bit-gwrsq                               1/1     Running   0          2m43s
   ntnx-system   fluent-bit-xdq2n                               1/1     Running   0          2m43s
   ntnx-system   kibana-logging-6ccd475856-cb4dd                1/2     Running   0          2m45s
   ```

   

5. 從Prism 介面上去看多了Logging的選項

   ![image-20231005164515179](https://kenkenny.synology.me:5543/images/2023/10/image-20231005164515179.png)

6. 點進去Logging介面，預設使用worker node的30950 port，點選Explore on my own

   ![image-20231005164742058](https://kenkenny.synology.me:5543/images/2023/10/image-20231005164742058.png)

   ![image-20231005170115884](https://kenkenny.synology.me:5543/images/2023/10/image-20231005170115884.png)

7. 點選左邊的選單--> Analytics --> Discover

   ![image-20231005170207799](https://kenkenny.synology.me:5543/images/2023/10/image-20231005170207799.png)

8. 建立新的index pattern，名稱輸入*

   ![image-20231005170304609](https://kenkenny.synology.me:5543/images/2023/10/image-20231005170304609.png)

   ![image-20231005170428292](https://kenkenny.synology.me:5543/images/2023/10/image-20231005170428292.png)

9. 點回Discover確認有抓到資訊

   ![image-20231005170840491](https://kenkenny.synology.me:5543/images/2023/10/image-20231005170840491.png)

   

## 擴充節點

兩個方式，第一個是新增Node Pool，第二種是在現有的Node Pool 調整 Node的數量

### Add Node Pool

![image-20231005172622386](https://kenkenny.synology.me:5543/images/2023/10/image-20231005172622386.png)

原本的Node數量

```
wangken@wangken-MAC ken-nke1 % ./nke-ctl get nodes                             
NAME                       STATUS   ROLES                  AGE    VERSION
ken-nke1-906a07-master-0   Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-master-1   Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-worker-0   Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-1   Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-2   Ready    node                   2d3h   v1.24.10
```

新增後約3~5分鐘就完成建立，並帶有dev的前綴詞

```
wangken@wangken-MAC ken-nke1 % ./nke-ctl get nodes
NAME                           STATUS   ROLES                  AGE    VERSION
ken-nke1-906a07-dev-worker-0   Ready    node                   54s    v1.24.10
ken-nke1-906a07-master-0       Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-master-1       Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-worker-0       Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-1       Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-2       Ready    node                   2d3h   v1.24.10


```

![image-20231005173405029](https://kenkenny.synology.me:5543/images/2023/10/image-20231005173405029.png)

要刪除Node Pool 要先刪除那個Node

![image-20231005173856946](https://kenkenny.synology.me:5543/images/2023/10/image-20231005173856946.png)

約3~5分鐘後就刪除完成，再刪除Node Pool

![image-20231005174024769](https://kenkenny.synology.me:5543/images/2023/10/image-20231005174024769.png)

```
wangken@wangken-MAC ken-nke1 % ./nke-ctl get nodes                                                     
NAME                       STATUS   ROLES                  AGE    VERSION
ken-nke1-906a07-master-0   Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-master-1   Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-worker-0   Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-1   Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-2   Ready    node                   2d3h   v1.24.10
```

### Resize Node Pool

點選Node Pool --> Actions --> Resize ，從 3 調整到4

![image-20231005174203052](https://kenkenny.synology.me:5543/images/2023/10/image-20231005174203052.png)

約3~5分鐘就出現新的worker node 3

```
wangken@wangken-MAC ken-nke1 % ./nke-ctl get nodes
NAME                       STATUS   ROLES                  AGE    VERSION
ken-nke1-906a07-master-0   Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-master-1   Ready    control-plane,master   2d3h   v1.24.10
ken-nke1-906a07-worker-0   Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-1   Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-2   Ready    node                   2d3h   v1.24.10
ken-nke1-906a07-worker-3   Ready    node                   54s    v1.24.10
```

![image-20231005174557923](https://kenkenny.synology.me:5543/images/2023/10/image-20231005174557923.png)

除了用Delete的方式移除worker-3之外，也可以用resize調整worker node數量

![image-20231005174749203](https://kenkenny.synology.me:5543/images/2023/10/image-20231005174749203.png)

![image-20231005174815545](https://kenkenny.synology.me:5543/images/2023/10/image-20231005174815545.png)

![image-20231005175011853](https://kenkenny.synology.me:5543/images/2023/10/image-20231005175011853.png)

驗證OK

## 升級Kubernetes

理論上順序應為 LCM 升級NKE版本 --> OS image 升級 --> Kubernetes version 升級

這裡NKE、OS image已經為最新版本，故只做Kubernetes version升級



### 步驟

點進NKE Cluster --> More --> Upgrade Kubernetes

![image-20231011093836949](https://kenkenny.synology.me:5543/images/2023/10/image-20231011093836949.png)

點選要升級的版本 1.25.6-0 --> Upgrade

![image-20231011093920386](https://kenkenny.synology.me:5543/images/2023/10/image-20231011093920386.png)

開始跑 Health ckeck

![image-20231011094057124](https://kenkenny.synology.me:5543/images/2023/10/image-20231011094057124.png)

現行版本

![image-20231011094304560](https://kenkenny.synology.me:5543/images/2023/10/image-20231011094304560.png)

會先進行master的升級，再來執行worker node滾動升級

![image-20231011094357148](https://kenkenny.synology.me:5543/images/2023/10/image-20231011094357148.png)

我的服務replica 只有設定1，所以服務會暫時中斷

![image-20231011094815399](https://kenkenny.synology.me:5543/images/2023/10/image-20231011094815399.png)

Cluster內也會有服務重啟，Kubenetes 已升級完成，因為只做Kubernetes升級所以Kernel 跟Container-runtime都不會升級

![image-20231011095351235](https://kenkenny.synology.me:5543/images/2023/10/image-20231011095351235.png)

約30分鐘內即升級完畢

![image-20231011100949610](https://kenkenny.synology.me:5543/images/2023/10/image-20231011100949610.png)

![image-20231011101014871](https://kenkenny.synology.me:5543/images/2023/10/image-20231011101014871.png)

因redmine服務需要先啟動DB再啟動redmine，所以將redmine的pod刪除，服務才會正常

這個可以去設定啟動順序，但因測試使用，故沒特別設定

確認服務正常

![image-20231011101550166](https://kenkenny.synology.me:5543/images/2023/10/image-20231011101550166.png)



## Advanced Kubernetes (TP)

進階的Kubernetes是Nutanix目前正在開發中的功能，啟用方式是會有一個管理用的NKE Cluster，然後去部署Agent到被管理的叢集

運作模式有點像是Rancher的管理方式，都需要一個Management的Cluster，這樣好處是Management掛了不會去影響現有服務。

![image-20231011103506802](https://kenkenny.synology.me:5543/images/2023/10/image-20231011103506802.png)

### 新增Management Cluster

1. 官方建議是要至少兩個worker node，先用一個來測試功能

   ![image-20231011110314489](https://kenkenny.synology.me:5543/images/2023/10/image-20231011110314489.png)

2. 給予名稱、版本

   ![image-20231011110355671](https://kenkenny.synology.me:5543/images/2023/10/image-20231011110355671.png)

3. 選擇網段、worker node數量

   ![image-20231011110436588](https://kenkenny.synology.me:5543/images/2023/10/image-20231011110436588.png)

4. 選擇 Network Provider

   ![image-20231011110520814](https://kenkenny.synology.me:5543/images/2023/10/image-20231011110520814.png)

5. 選擇Storage class，之後點選Create

   ![image-20231011110617541](https://kenkenny.synology.me:5543/images/2023/10/image-20231011110617541.png)

6. 建立中

   ![image-20231011111332153](https://kenkenny.synology.me:5543/images/2023/10/image-20231011111332153.png)

7. NKE Cluster 建立完成

   ![image-20231011113040549](https://kenkenny.synology.me:5543/images/2023/10/image-20231011113040549.png)

8. ssh 進入PCVM ， 針對ken-mgmt cluster啟用Advance Management

   ```
   nutanix@NTNX-172-16-90-75-A-PCVM:~$ /home/nutanix/karbon/karbonctl login --pc-username ken.wang@nutanixlab.local
   Please enter the password for the PC user: ken.wang@nutanixlab.local
   Login successful
   
   nutanix@NTNX-172-16-90-75-A-PCVM:~$ /home/nutanix/karbon/karbonctl karbon-management enable --cluster-name ken-mgmt
   2023-10-11T03:32:34.711Z client_wrapper.go:300: [INFO] Successfully created new resource: &{karbon-mgmt false 0xc0005340e0 0xc0008925e0 /v1, Kind=Namespace karbon-mgmt}
   2023-10-11T03:32:34.784Z client_wrapper.go:317: [INFO] Successfully patched existing resource. NS: karbon-mgmt, Name: servicemonitors.monitoring.coreos.com
   2023-10-11T03:32:34.805Z client_wrapper.go:300: [INFO] Successfully created new resource: &{karbon-mgmt true 0xc000e66ee0 0xc00103e1f0 networking.k8s.io/v1, Kind=NetworkPolicy traefik}
   2023-10-11T03:32:34.832Z client_wrapper.go:300: [INFO] Successfully created new resource: &{karbon-mgmt true 0xc0001f7490 0xc000c5a300 /v1, Kind=ConfigMap traefik-dynamic-configmap}
   2023-10-11T03:32:34.881Z client_wrapper.go:300: [INFO] Successfully created new resource: &{karbon-mgmt false 0xc00062b3b0 0xc000c5a428 apiextensions.k8s.io/v1, Kind=CustomResourceDefinition ingressroutes.traefik.containo.us}
   2023-10-11T03:32:34.92Z client_wrapper.go:300: [INFO] Successfully created new resource: &{karbon-mgmt false 0xc0001dc7e0 0xc000c5a4b8 apiextensions.k8s.io/v1, Kind=CustomResourceDefinition ingressroutetcps.traefik.containo.us}
   2023-10-11T03:32:34.999Z client_wrapper.go:300: [INFO] Successfully created
   ...
   Waiting on 172.16.90.75 (Up, ZeusLeader) to start:  KarbonUI KarbonCore
   
   Waiting on 172.16.90.75 (Up, ZeusLeader) to start:  KarbonUI KarbonCore
   
   Waiting on 172.16.90.75 (Up, ZeusLeader) to start: 
   ...
   2b7839f469 karbon-ui:v2.8.0 Up 38 seconds (healthy)
   2ef4e4d10f karbon-core:v2.8.0 Up 35 seconds (healthy)
   2b7839f469 karbon-ui:v2.8.0 Up 43 seconds (healthy)
   Successfully enabled karbon management!
   ```

   

9. Deploy Agent 在ken-nke1 Cluster

   ```
   nutanix@NTNX-172-16-90-75-A-PCVM:~$ /home/nutanix/karbon/karbonctl karbon-agent enable --cluster-name ken-nke1 --mgmt-name ken-mgmt
   2023-10-11T03:38:30.244Z k8s_versioning.go:181: [ERROR] Failed to get Namespace for verification : namespaces "karbon-agent" not found
   2023-10-11T03:38:30.28Z client_wrapper.go:300: [INFO] Successfully created new resource: &{ false 0xc001142850 0xc001136038 /v1, Kind=Namespace karbon-agent}
   2023-10-11T03:38:30.302Z karbon_mgmt_sentry.go:169: [INFO] The Karbon-Mgmt pod is ready
   2023-10-11T03:38:30.303Z karbon_mgmt_sentry.go:173: [ERROR] Empty payload return
   2023-10-11T03:38:30.325Z karbon_mgmt_sentry.go:169: [INFO] The Karbon-Mgmt pod is ready
   2023-10-11T03:38:31Z karbon_mgmt_sentry.go:169: [INFO] The Karbon-Mgmt pod is ready
   2023-10-11T03:38:33.758Z agent_installer.go:187: [INFO] Edgemgmt image passed is: sherlock-edgemgmt:3274
   2023-10-11T03:38:33.758Z agent_installer.go:194: [INFO] Controller image passed is: datastream-controller:2490
   2023-10-10 20:38:33.947060 I | creating 3 resource(s)
   2023-10-10 20:38:33.971481 I | Clearing discovery cache
   2023-10-10 20:38:33.971530 I | beginning wait for 3 resources with timeout of 1m0s
   2023-10-10 20:38:37.401949 I | creating 29 resource(s)
   2023-10-10 20:38:37.635612 I | beginning wait for 29 resources with timeout of 15m0s
   2023-10-10 20:38:37.803750 I | Deployment is not ready: karbon-agent/controller-deployment. 0 out of 1 expected pods are ready
   2023-10-10 20:38:39.845704 I | Deployment is not ready: karbon-agent/controller-deployment. 0 out of 1 expected pods are ready
   ...
   2023-10-10 20:38:53.847939 I | Deployment is not ready: karbon-agent/controller-deployment. 0 out of 1 expected pods are ready
   2023-10-10 20:38:55.865197 I | Deployment is not ready: karbon-agent/edgemgmt. 0 out of 1 expected pods are ready
   2023-10-10 20:38:57.851395 I | Deployment is not ready: karbon-agent/edgemgmt. 0 out of 1 expected pods are ready
   ...
   2023-10-10 20:39:25.875878 I | release installed successfully: karbon-agent/servicedomain-2.5.0
   2023-10-11T03:39:25.875Z helm_agent_installer.go:66: [INFO] Deployed Agent /home/nutanix/karbon/sherlock_edge_deployer_pc.tgz using helm chart
   2023-10-11T03:39:25.875Z karbonagent_enable.go:182: [INFO] Successfully deployed karbon agent.
   Successfully Enabled the Karbon Agent
   
   ```

   

10. 原本的 pod

    ```
    wangken@wangken-MAC ken-nke1 % ./nke-ctl get po -A        
    NAMESPACE     NAME                                           READY   STATUS    RESTARTS       AGE
    kube-system   coredns-7b95cd4468-5zqhg                       1/1     Running   0              42m
    kube-system   coredns-7b95cd4468-qhszt                       1/1     Running   0              109m
    kube-system   kube-apiserver-ken-nke1-906a07-master-0        3/3     Running   2 (113m ago)   113m
    kube-system   kube-apiserver-ken-nke1-906a07-master-1        3/3     Running   1 (113m ago)   113m
    kube-system   kube-flannel-ds-dp5jv                          1/1     Running   0              7d21h
    kube-system   kube-flannel-ds-gqz5j                          1/1     Running   0              7d21h
    kube-system   kube-flannel-ds-slmqt                          1/1     Running   0              7d21h
    kube-system   kube-flannel-ds-z7hnh                          1/1     Running   0              7d21h
    kube-system   kube-proxy-ds-75tnd                            1/1     Running   0              104m
    kube-system   kube-proxy-ds-lqmzt                            1/1     Running   0              104m
    kube-system   kube-proxy-ds-qwzcd                            1/1     Running   0              104m
    kube-system   kube-proxy-ds-rcqfv                            1/1     Running   0              105m
    ntnx-system   alertmanager-main-0                            2/2     Running   0              42m
    ntnx-system   alertmanager-main-1                            2/2     Running   1 (103m ago)   103m
    ntnx-system   alertmanager-main-2                            2/2     Running   1 (103m ago)   103m
    ntnx-system   blackbox-exporter-7d494dffb6-5w2zm             3/3     Running   0              42m
    ntnx-system   csi-snapshot-controller-5fc6f8f5c4-hvcsj       1/1     Running   0              109m
    ntnx-system   csi-snapshot-webhook-78957b8ddd-pwggj          1/1     Running   0              109m
    ntnx-system   elasticsearch-logging-0                        1/1     Running   0              104m
    ntnx-system   fluent-bit-6cjst                               1/1     Running   0              5d18h
    ntnx-system   fluent-bit-74fw9                               1/1     Running   0              5d18h
    ntnx-system   fluent-bit-dn9qc                               1/1     Running   0              5d18h
    ntnx-system   fluent-bit-gwrsq                               1/1     Running   0              5d18h
    ntnx-system   kibana-logging-6ccd475856-xgmjz                2/2     Running   1 (104m ago)   109m
    ntnx-system   kube-state-metrics-758b544d66-ns667            3/3     Running   0              42m
    ntnx-system   kubernetes-events-printer-8d4c49454-kjtkm      1/1     Running   0              42m
    ntnx-system   node-exporter-9698d                            2/2     Running   0              102m
    ntnx-system   node-exporter-hhqzn                            2/2     Running   0              103m
    ntnx-system   node-exporter-wkg4z                            2/2     Running   0              104m
    ntnx-system   node-exporter-z5d8f                            2/2     Running   0              102m
    ntnx-system   nutanix-csi-controller-864cc4767b-mhftm        5/5     Running   0              109m
    ntnx-system   nutanix-csi-node-8c2cc                         3/3     Running   0              7d21h
    ntnx-system   nutanix-csi-node-sf6wd                         3/3     Running   0              7d21h
    ntnx-system   prometheus-adapter-56755b575c-clhp2            1/1     Running   0              104m
    ntnx-system   prometheus-adapter-56755b575c-fb979            1/1     Running   0              42m
    ntnx-system   prometheus-k8s-0                               2/2     Running   0              42m
    ntnx-system   prometheus-k8s-1                               2/2     Running   0              103m
    ntnx-system   prometheus-operator-655cf4b548-tr7kv           2/2     Running   0              42m
    redmine       redmine-7fd78679f9-7682g                       1/1     Running   0              42m
    redmine       redmine-postgres-deployment-7bd9859c6d-thrcv   1/1     Running   0              42m
    ```

    

11. 部署Agent完成後的pod，新增了兩個Namespace: karbon-agent、project-ingress

    ```
    wangken@wangken-MAC ken-nke1 % ./nke-ctl get po -A
    NAMESPACE         NAME                                           READY   STATUS    RESTARTS       AGE
    karbon-agent      controller-deployment-788c9df5b-rg47k          1/1     Running   0              119s
    karbon-agent      edgemgmt-c97765dc7-fhs6d                       2/2     Running   0              119s
    karbon-agent      nats-c9bd445bb-lzvdg                           1/1     Running   0              72s
    kube-system       coredns-7b95cd4468-5zqhg                       1/1     Running   0              46m
    kube-system       coredns-7b95cd4468-qhszt                       1/1     Running   0              113m
    kube-system       kube-apiserver-ken-nke1-906a07-master-0        3/3     Running   2 (117m ago)   117m
    kube-system       kube-apiserver-ken-nke1-906a07-master-1        3/3     Running   1 (117m ago)   117m
    kube-system       kube-flannel-ds-dp5jv                          1/1     Running   0              7d21h
    kube-system       kube-flannel-ds-gqz5j                          1/1     Running   0              7d21h
    kube-system       kube-flannel-ds-slmqt                          1/1     Running   0              7d21h
    kube-system       kube-flannel-ds-z7hnh                          1/1     Running   0              7d21h
    kube-system       kube-proxy-ds-75tnd                            1/1     Running   0              108m
    kube-system       kube-proxy-ds-lqmzt                            1/1     Running   0              108m
    kube-system       kube-proxy-ds-qwzcd                            1/1     Running   0              108m
    kube-system       kube-proxy-ds-rcqfv                            1/1     Running   0              109m
    ntnx-system       alertmanager-main-0                            2/2     Running   0              46m
    ntnx-system       alertmanager-main-1                            2/2     Running   1 (107m ago)   108m
    ntnx-system       alertmanager-main-2                            2/2     Running   1 (107m ago)   108m
    ntnx-system       blackbox-exporter-7d494dffb6-5w2zm             3/3     Running   0              46m
    ntnx-system       csi-snapshot-controller-5fc6f8f5c4-hvcsj       1/1     Running   0              113m
    ntnx-system       csi-snapshot-webhook-78957b8ddd-pwggj          1/1     Running   0              113m
    ntnx-system       elasticsearch-logging-0                        1/1     Running   0              109m
    ntnx-system       fluent-bit-6cjst                               1/1     Running   0              5d19h
    ntnx-system       fluent-bit-74fw9                               1/1     Running   0              5d19h
    ntnx-system       fluent-bit-dn9qc                               1/1     Running   0              5d19h
    ntnx-system       fluent-bit-gwrsq                               1/1     Running   0              5d19h
    ntnx-system       kibana-logging-6ccd475856-xgmjz                2/2     Running   1 (108m ago)   113m
    ntnx-system       kube-state-metrics-758b544d66-ns667            3/3     Running   0              46m
    ntnx-system       kubernetes-events-printer-8d4c49454-kjtkm      1/1     Running   0              46m
    ntnx-system       node-exporter-9698d                            2/2     Running   0              106m
    ntnx-system       node-exporter-hhqzn                            2/2     Running   0              107m
    ntnx-system       node-exporter-wkg4z                            2/2     Running   0              108m
    ntnx-system       node-exporter-z5d8f                            2/2     Running   0              106m
    ntnx-system       nutanix-csi-controller-864cc4767b-mhftm        5/5     Running   0              113m
    ntnx-system       nutanix-csi-node-8c2cc                         3/3     Running   0              7d21h
    ntnx-system       nutanix-csi-node-sf6wd                         3/3     Running   0              7d21h
    ntnx-system       prometheus-adapter-56755b575c-clhp2            1/1     Running   0              108m
    ntnx-system       prometheus-adapter-56755b575c-fb979            1/1     Running   0              46m
    ntnx-system       prometheus-k8s-0                               2/2     Running   0              46m
    ntnx-system       prometheus-k8s-1                               2/2     Running   0              108m
    ntnx-system       prometheus-operator-655cf4b548-tr7kv           2/2     Running   0              46m
    project-ingress   datastream-mqtt-ingress-54b8c67bdd-d7kbl       1/1     Running   0              56s
    project-ingress   datastream-rtsp-ingress-59c47c8f7c-pgwpc       1/1     Running   0              56s
    project-ingress   nats-8bc8494cd-vxs4b                           1/1     Running   0              72s
    redmine           redmine-7fd78679f9-7682g                       1/1     Running   0              46m
    redmine           redmine-postgres-deployment-7bd9859c6d-thrcv   1/1     Running   0              46m
    ```

    

### 完成前後對照

原本的畫面

![image-20231011092930911](https://kenkenny.synology.me:5543/images/2023/10/image-20231011092930911.png)

因爲測試使用，故把Worker Node數量降到2

![image-20231011105655407](https://kenkenny.synology.me:5543/images/2023/10/image-20231011105655407.png)

完成後多了一個Namespace的選項

![image-20231011114956233](https://kenkenny.synology.me:5543/images/2023/10/image-20231011114956233.png)

權限部分應該是還沒做好，登出後用local admin帳號登入就看得到namespace內容

然後ken-mgmt的worker node數量調整為2後，就可以看到namespace裡面的Deployment跟其他資訊

![image-20231011120129274](https://kenkenny.synology.me:5543/images/2023/10/image-20231011120129274.png)

Workloads

![image-20231011132124834](https://kenkenny.synology.me:5543/images/2023/10/image-20231011132124834.png)

Workloads : Deployments

![image-20231011125442631](https://kenkenny.synology.me:5543/images/2023/10/image-20231011125442631.png)

Deployment詳細資訊

![image-20231011125601847](https://kenkenny.synology.me:5543/images/2023/10/image-20231011125601847.png)

Deployment詳細資訊，Pod名稱跟Yaml格式

![image-20231011125620243](https://kenkenny.synology.me:5543/images/2023/10/image-20231011125620243.png)

Config有 ConfigMaps跟Secrets

![image-20231011125748609](https://kenkenny.synology.me:5543/images/2023/10/image-20231011125748609.png)

Network裡面的Service，看是走Nodeport or ClusterIP or Ingress

![image-20231011125845677](https://kenkenny.synology.me:5543/images/2023/10/image-20231011125845677.png)

Endpoints

![image-20231011125950105](https://kenkenny.synology.me:5543/images/2023/10/image-20231011125950105.png)

PVC

![image-20231011130008347](https://kenkenny.synology.me:5543/images/2023/10/image-20231011130008347.png)

點進去都可以看到Yaml格式，就跟使用kubectl edit 會出現的yaml是一樣的，但這邊無法編輯

![image-20231011130058520](https://kenkenny.synology.me:5543/images/2023/10/image-20231011130058520.png)

Charts還不確定會有什麼可以呈現

![image-20231011130156905](https://kenkenny.synology.me:5543/images/2023/10/image-20231011130156905.png)

## 總結

NKE 的部署很簡單、快速，透過CSI Driver 可以結合Nutanix本身的檔案系統 ( Volume / Files ) 來做儲存

其實是很適合現有使用Nutanix ，但又同時想要擁有Kuberntes來做開發測試，或是高階玩家想省授權費用的人，

因為NKE 本身的監控、Log沒有像Redhat Openshift 有很多預設做好的介面跟Operator，

所以若是正式環境會需要自己部署ELK、Prometheus、Service Mesh等軟體。