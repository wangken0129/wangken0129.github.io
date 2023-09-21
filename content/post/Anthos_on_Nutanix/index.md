---
title: Anthos_on_Nutanix
description: Anthos_on_Nutanix
slug: Anthos_on_Nutanix
date: 2023-09-21T10:03:33+08:00
categories:
    - Lab Category
tags:
    - Nutanix
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


# Nutanix上快速部署 Anthos 



## 目標 :

透過 Nutanix 的 Calm 快速部署 Google Anthos Cluster並註冊到雲端，過程約一個小時，
並在Google Cloud Console 上來管理地端Anthos Cluster，達到混合雲的架構。

除了Nutanix Calm外，同樣也可以使用Terraform來達到相同目的。

## 環境準備 :

1. 一座 Nutanix Cluster 含至少192GB 可用記憶體、768GB 硬碟可用空間、
   6個 IPAM 給Anthos Cluster VM、3個Kubernetes用的IP

2. Prism Central 啟用Calm服務

3. 一台 Calm DSL VM (啟動Calm DSL Container )

4. 一組 ssh key 給Anthos 部署的VM 使用

其他準備：

```
Prerequisites

Before using any of the automation methods, make sure to meet the following requirements:
Automation	

    Calm:
        3.0.0.2 or later
        A project with AHV account

    ~ or ~
    Terraform:
        0.13.x or later
        Nutanix provider 1.2.x or later

Credentials	

    (Calm only) SSH key. It must start with —BEGIN RSA PRIVATE KEY—
    Prism Element account with User Admin role
    Prism Central account with CRUD VM permissions

Networking	

    Internet connectivity
    AHV IPAM pool with minimum 6 IP addresses
    Kubernetes:
        Control plane VIP
        Ingress VIP
        Load balancing pool

Nutanix	

    Prism Element cluster:
        AHV: 20201105.1045 or later
        AOS: 5.19.1 or later
        iSCSI data service IP configured
        VLAN network with AHV IPAM configured
    Prism Central: 2020.11.0.1 or later

Google Cloud	

    A project with Owner role
    Project must have monitoring enabled (console)
    A service account (how-to)
        Role: Project Owner
        A private key: JSON format
```




## 版本資訊 :

- AOS Version: 6.5.2
- Prism Central Version: 2023.1.01
- Calm Version: 3.6.2 
- Calm DSL Version: 3.6.1.2023.03.15.commit.e753678
- Anthos Version: 1.15.0  (v1.26.2-gke.1001)
- CentOS Version: 8.2.2004-20200611.2

## 參考資料

Anthos on AHV
https://www.nutanix.dev/2021/04/26/anthos-clusters-on-ahv-getting-started/#calm-anthos-ahv

Calm Blueprint
https://github.com/nutanixdev/anthos-on-ahv/tree/main/calm

Service Account
https://cloud.google.com/anthos/run/docs/securing/service-accounts

## 架構圖 :

Calm

![Going Hybrid with Kubernetes on Google Cloud Platform and Nutanix | Google  Cloud Blog](https://kenkenny.synology.me:5543/images/2023/09/nutanix-kubernetes-7zv4o.max-700x700.png)



Anthos Cluster

<img src="assets/Anthos-clusters-on-AHV-Architecture-1024x733.png" alt="img"  />

Calm 與 Jenkins整合的流程，日後可做為參考

![image-20230711125654636](https://kenkenny.synology.me:5543/images/2023/09/image-20230711125654636.png)

Lab01 Info

| 名稱                  | IP                |
| --------------------- | ----------------- |
| Prism Central         | 172.16.90.72      |
| Prism Element         | 172.16.90.71      |
| Prism iSCSI Data IP   | 172.16.90.73      |
| Calm DSL VM           | 172.16.90.210     |
| anthos-lab01-admin-vm | 172.16.90.211-216 |
| anthos-lab01-master1  | 172.16.90.211-216 |
| anthos-lab01-master2  | 172.16.90.211-216 |
| anthos-lab01-master3  | 172.16.90.211-216 |
| anthos-lab01-worker1  | 172.16.90.211-216 |
| anthos-lab01-worker2  | 172.16.90.211-216 |
| Anthos cluster01 VIP  | 172.16.90.207     |
| Ingress VIP           | 172.16.90.205     |
| Loadbalncer IP        | 172.16.90.205-206 |

Lab02

| 名稱                  | IP                |
| --------------------- | ----------------- |
| anthos-lab02-admin-vm | 172.16.90.211-216 |
| anthos-lab02-master0  | 172.16.90.217-223 |
| anthos-lab02-master1  | 172.16.90.217-223 |
| anthos-lab02-master2  | 172.16.90.217-223 |
| anthos-lab02-worker0  | 172.16.90.217-223 |
| anthos-lab02-worker1  | 172.16.90.217-223 |
| anthos-lab02-worker2  | 172.16.90.217-223 |
| Anthos cluster02 VIP  | 172.16.90.224     |
| Ingress VIP           | 172.16.90.208     |
| Loadbalncer IP        | 172.16.90.208-209 |



## Calm DSL執行步驟 :

### Google Cloud Console :

1. 登入 Google console

https://console.cloud.google.com

2. 建立 Project & Enable Anthos
3. 建立 Service Account &私密金鑰並上傳

### Nutanix Cluster :

1. 登入 Prism Central 

2. 建立 Nutanix Project 並設定使用者、基礎環境

3. 取得blueprint、設定環境變數 (Calm DSL VM)

   ```shell
   $ cd ~/nutanix
   $ git clone https://github.com/mat0606/anthos-on-ahv 
   $ cd anthos-on-ahv/calm/
   $ vim blueprint.py
   
   Create Variables
   $ mkdir -p .local/secrets
   $ ls -l .local/secrets
   -rw-rw-r--. 1 nutanix nutanix    1  6月 14 07:03 gcloud_account
   -rw-r--r--. 1 nutanix nutanix    1  6月 14 07:04 gcloud_key
   -rw-rw-r--. 1 nutanix nutanix   11  6月  6 07:39 os2_key
   -rw-rw-r--. 1 nutanix nutanix    8  6月  6 07:39 os2_username
   -rw-------. 1 nutanix nutanix 2455  6月  9 04:19 os_key
   -rw-rw-r--. 1 nutanix nutanix    8  6月  6 06:57 os_username
   -rw-rw-r--. 1 nutanix nutanix   15  6月  6 06:57 pc_password
   -rw-rw-r--. 1 nutanix nutanix    6  6月  9 04:03 pc_username
   -rw-rw-r--. 1 nutanix nutanix   15  6月  6 06:57 pe_password
   -rw-rw-r--. 1 nutanix nutanix    6  6月  6 06:58 pe_username
   ```

4. 啟動Calm DSL Container 

   ```shell
   $ sudo docker run -v /home/nutanix/anthos-on-ahv:/home/centos/anthos-on-ahv:z -it ntnx/calm-dsl /bin/bash
   ```

5. Calm DSL連線到Prism Central及該Project (Calm DSL Container)

   ```shell
   $ ls -l /home/centos/anthos-on-ahv
   $ calm init dsl
   $ calm show config
   ```

6. Calm DSL 建立blueprint到Prism Central

   ```shell
   $ calm compile bp --name esun-bp --file /home/centos/anthos-on-ahv/calm/blueprint.py
   $ calm create bp --name esun-bp --file /home/centos/anthos-on-ahv/calm/blueprint.py
   ```

7. 在Prism Central 的blueprint貼上ssh key

8. Calm DSL 利用blueprint啟動Application

   ```shell
   $ calm launch bp "Anthos-AHV-DSL" --app_name anthos
   ```

9. 等待約30~45分鐘即可看到Cluster建立完成

   

### Anthos-Cluster：

1. 登入admin-VM取得kubeconfig確認Cluster

   ```shell
   $ SECRET_NAME=$(kubectl get serviceaccount google-cloud-console -o jsonpath='{$.secrets[0].name}')
   $ kubectl get secret ${SECRET_NAME} -o jsonpath='{$.data.token}' | base64 --decode
   ```

2. 驗證workload

   ```shell
   $ sudo yum -y install git
   $ git clone https://github.com/mat0606/K8S.git
   $ kubectl create ns wordpress
   $ kubectl -n wordpress create secret generic mysql-pass --from-literal=password='Nutanix/4u'
   $ kubectl -n wordpress apply -f mysql-deployment.yaml
   $ kubectl -n wordpress get pvc
   ```

3. Google Console 確認Cluster已有註冊



## Terraform 執行步驟：

### Google Cloud Console：

與Calm 執行步驟相同

1. 登入 Google console

https://console.cloud.google.com

2. 建立 Project & Enable Anthos
3. 建立 Service Account &私密金鑰並上傳

### Nutanix Cluster：

只需準備IPAM的可用IP即可

### Terraform：

1. 建立terraform.tfvars

   ```shell
   echo 'subnet_name = "<subnet_name>"
   anthos_version = "<anthos_version>"
   anthos_controlplane_vip = "<anthos_controlplane_vip>"
   anthos_ingress_vip = "<anthos_ingress_vip>"
   anthos_lb_addresspool = "<anthos_lb_addresspool>"
   anthos_cluster_name = "<anthos_cluster_name>"
   google_application_credentials_path = "<google_application_credentials_path>"
   amount_of_anthos_worker_vms = "<amount_of_anthos_worker_vms>"
   ntnx_pc_username = "<ntnx_pc_username>"
   ntnx_pc_password = "<ntnx_pc_password>"
   ntnx_pc_ip = "<ntnx_pc_ip>"
   ntnx_pe_storage_container = "<ntnx_pe_storage_container>"
   ntnx_pe_username = "<ntnx_pe_username>"
   ntnx_pe_password = "<ntnx_pe_password>"
   ntnx_pe_ip = "<ntnx_pe_ip>"
   ntnx_pe_dataservice_ip = "<ntnx_pe_dataservice_ip>"' > terraform.tfvars
   
   $ chmod 0600 terraform.tfvars
   ```

2. 執行terraform初始化並產生plan

   ```shell
   $ terraform init
   $ terraform plan
   ```

3. 部署anthos cluster

   ```shell
   $ terraform apply
   ```

### Anthos-Cluster：

1. 部署完成後登入admin-vm確認Cluster

2. 取得kubeconfig以登入Google Cloud Console

   ```shell
   $ SECRET_NAME=$(kubectl get serviceaccount google-cloud-console -o jsonpath='{$.secrets[0].name}')
   $ kubectl get secret ${SECRET_NAME} -o jsonpath='{$.data.token}' | base64 --decode
   ```

3. Google Console 確認Cluster已有註冊

4. Scale out 測試，修改terraform.tfvars的worker數量

   ```shell
   $ terraform apply
   ```

5. 登入admin-vm 確認Scale out 完成



## Google與Calm bp完整步驟



### Anthos

#### Create Service Accounts

1. 進入IAM與管理,點選project (Anthos-Test-202303)
   ![image-20230314115438141](https://kenkenny.synology.me:5543/images/2023/09/anthos-sa01.png)

2. 點選建立服務帳戶
   ![image-20230314115626110](https://kenkenny.synology.me:5543/images/2023/09/anthos-sa02.png)

3. 給擁有者的權限

   ![image-20230314115729988](https://kenkenny.synology.me:5543/images/2023/09/anthos-sa03.png)

4. 郵件地址複製
   netfos@anthos-test-202303.iam.gserviceaccount.com

5. 點選該服務帳戶,新增金鑰
   ![image-20230314115902286](https://kenkenny.synology.me:5543/images/2023/09/anthos-sa04.png)

6. 儲存金鑰
   ![image-20230314120003591](https://kenkenny.synology.me:5543/images/2023/09/anthos-sa05.png)
   ![image-20230314120124523](https://kenkenny.synology.me:5543/images/2023/09/anthos-sa06.png)

#### Enable Anthos

![image-20230314110332738](https://kenkenny.synology.me:5543/images/2023/09/anthos-active01.png)



### Prism Central & Calm

1. IPAM設定IP Pool

   ![image-20230314111246357](https://kenkenny.synology.me:5543/images/2023/09/anthos-IPAM.png)

2. Calm建立Project
   ![image-20230314113139889](https://kenkenny.synology.me:5543/images/2023/09/anthos-project01.png)

   ![image-20230314113258956](https://kenkenny.synology.me:5543/images/2023/09/anthos-project02.png)

   ![image-20230314155154308](https://kenkenny.synology.me:5543/images/2023/09/anthos-project03.png)

3. 上傳github 上的 blueprint
   ![image-20230314113458998](https://kenkenny.synology.me:5543/images/2023/09/anthos-blueprint.png)

   點開Blueprint

   ![image-20230314113547773](https://kenkenny.synology.me:5543/images/2023/09/anthos-blueprint02.png)

4. 調整參數可以安裝時再輸入即可 (有藍色的人代表安裝時可以輸入)

5. VM網卡要先設定好
   ![image-20230314160312365](https://kenkenny.synology.me:5543/images/2023/09/anthos-bp-network.png)

6. Credential 要先輸入
   ![image-20230314152343047](https://kenkenny.synology.me:5543/images/2023/09/anthos-cred.png)

7. CRED_OS ssh-keygen

   ```shell
   ssh-keygen -t rsa -f ~/.ssh/KEY_FILENAME -C USERNAME -b 2048
   
   
   wangken@wangken-MAC ~ % ssh-keygen -t rsa -f ~/.ssh/anthos_key -C nutanix -b 2048
   Generating public/private rsa key pair.
   Enter passphrase (empty for no passphrase): 
   Enter same passphrase again: 
   Your identification has been saved in /Users/wangken/.ssh/anthos_key
   Your public key has been saved in /Users/wangken/.ssh/anthos_key.pub
   The key fingerprint is:
   SHA256:vBzHezjT7zfmG5ZWQX+eS78TLDHmfNj7g1Rki4PxV5Q nutanix
   The key's randomart image is:
   +---[RSA 2048]----+
   |               oo|
   |           .  .Eo|
   |            + +.=|
   |       . . . B =+|
   |        S o + Xoo|
   |       . + + *.==|
   |        o = + +*+|
   |           + oo*+|
   |             .==B|
   +----[SHA256]-----+
   
   wangken@wangken-MAC .ssh % ls |grep anthos 
   anthos_key.pub 
   anthos_key
   ```

8. anthos sa private key

   ```shell
   wangken@wangken-MAC notes % cat anthos-test-202303-privatekey.json 
   {
     "type": "service_account",
     "project_id": "anthos-test-202303",
     "private_key_id": "b76e04e16af19f535e185f2f36bae2be8038c363",
     "private_key": "-----BEGIN PRIVATE KEY-----
     XXXX\n-----END PRIVATE KEY-----\n",
     "client_email": "netfos@anthos-test-202303.iam.gserviceaccount.com",
     "client_id": "110466507071833816563",
     "auth_uri": "https://accounts.google.com/o/oauth2/auth",
     "token_uri": "https://oauth2.googleapis.com/token",
     "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
     "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/netfos%40anthos-test-202303.iam.gserviceaccount.com"
   }
   ```

9. 輸入完後按save , 執行
   ![image-20230314153857881](https://kenkenny.synology.me:5543/images/2023/09/anthos-launch.png)

10. 填入資訊
    ![image-20230314154058030](https://kenkenny.synology.me:5543/images/2023/09/anthos-lauch01.png)

11. IP 資訊
    ![image-20230314154540212](https://kenkenny.synology.me:5543/images/2023/09/anthos-lauch02.png)

12. IP 資訊 Ingress VIP 要在load balancing pool 內
    ![image-20230314154644473](https://kenkenny.synology.me:5543/images/2023/09/anthos-lauch03.png)

13. Deploy
    ![image-20230314160527582](https://kenkenny.synology.me:5543/images/2023/09/anthos-deploy.png)

14. 自動化部屬
    ![image-20230314160559318](https://kenkenny.synology.me:5543/images/2023/09/anthos-deploy05.png)

15. Manage可以查看目前的部署狀態
    ![image-20230314160801127](https://kenkenny.synology.me:5543/images/2023/09/anthos-deploy06.png)

16. 部署完成
    ![image-20230314173530825](https://kenkenny.synology.me:5543/images/2023/09/anthos-vm.png)

17. 登入Admin VM 取得Secret並在Anthos Console登入

    ```shell
    wangken@wangken-MAC ~ % ssh -i .ssh/anthos_key nutanix@172.16.90.206
    Activate the web console with: systemctl enable --now cockpit.socket
    
    Last login: Tue Mar 14 09:21:38 2023 from 172.16.90.72
    
    [nutanix@anthos-netfos1-anthos-adminVm-0 ~]$ kubectl get nodes
    NAME                                STATUS   ROLES                  AGE   VERSION
    anthos-netfos1-anthos-controlvm-0   Ready    control-plane,master   57m   v1.21.5-gke.1200
    anthos-netfos1-anthos-controlvm-1   Ready    control-plane,master   53m   v1.21.5-gke.1200
    anthos-netfos1-anthos-controlvm-2   Ready    control-plane,master   53m   v1.21.5-gke.1200
    anthos-netfos1-anthos-workervm-0    Ready    <none>                 50m   v1.21.5-gke.1200
    anthos-netfos1-anthos-workervm-1    Ready    <none>                 50m   v1.21.5-gke.1200
    ```

    ```shell
    [nutanix@anthos-netfos1-anthos-adminVm-0 anthos-netfos1]$ pwd
    /home/nutanix/baremetal/bmctl-workspace/anthos-netfos1
    
    [nutanix@anthos-netfos1-anthos-adminVm-0 anthos-netfos1]$ cat anthos-netfos1-kubeconfig 
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: 
    XXXX
        server: https://172.16.90.212:443
      name: anthos-netfos1
    contexts:
    - context:
        cluster: anthos-netfos1
        user: anthos-netfos1-admin
      name: anthos-netfos1-admin@anthos-netfos1
    current-context: anthos-netfos1-admin@anthos-netfos1
    kind: Config
    preferences: {}
    users:
    - name: anthos-netfos1-admin
      user:
        client-certificate-data: 
    XXX
        client-key-data: 
    XXXX
    
    ```

    

18. 登入Anthos Console用憑證下面兩行指令得出的base64 碼

    ```shell
    $SECRET_NAME=$(kubectl get serviceaccount google-cloud-console -o jsonpath='{$.secrets[0].name}')
    $kubectl get secret ${SECRET_NAME} -o jsonpath='{$.data.token}' | base64 --decode
    ```

    登入完成
    ![image-20230314174734950](https://kenkenny.synology.me:5543/images/2023/09/anthos-login.png)

    ![image-20230314175007215](https://kenkenny.synology.me:5543/images/2023/09/anthos-GKE.png)

19. Workload
    ![image-20230314175203478](https://kenkenny.synology.me:5543/images/2023/09/anthos-workload.png)

20. Marketplace部署測試程式
    ![image-20230314175624682](https://kenkenny.synology.me:5543/images/2023/09/anthos-marketplace01.png)

21. 設定並部署
    ![image-20230314175712418](https://kenkenny.synology.me:5543/images/2023/09/anthos-marketplace02.png)

    ![image-20230314180133597](https://kenkenny.synology.me:5543/images/2023/09/anthos-marketplace03.png)

22. 在地端查看namespace狀態

    ```shell
    [nutanix@anthos-netfos1-anthos-adminVm-0 ~]$ kubectl get all -n harbor
    NAME                                          READY   STATUS             RESTARTS   AGE
    pod/harbor-1-chartmuseum-967bb456b-2vmjk      1/1     Running            0          3m18s
    pod/harbor-1-core-7956bd9f54-t4jcw            1/1     Running            1          3m18s
    pod/harbor-1-database-0                       1/1     Running            0          3m18s
    pod/harbor-1-deployer-rq8m5                   0/1     Completed          0          3m59s
    pod/harbor-1-exporter-5c86cc7b6d-s78kb        1/2     CrashLoopBackOff   5          3m18s
    pod/harbor-1-jobservice-9898c468b-5bhzp       1/1     Running            3          3m18s
    pod/harbor-1-nginx-66d4ccd87-7p7q2            1/1     Running            0          3m18s
    pod/harbor-1-notary-server-7d7b4b598d-85nm5   1/1     Running            2          3m18s
    pod/harbor-1-notary-signer-5d59d79f4-4tkn4    1/1     Running            2          3m18s
    pod/harbor-1-portal-66f4b7478b-5xv5b          1/1     Running            0          3m18s
    pod/harbor-1-redis-0                          1/1     Running            0          3m18s
    pod/harbor-1-registry-67bc689779-bvw8m        2/2     Running            0          3m18s
    pod/harbor-1-trivy-0                          1/1     Running            0          3m18s
    
    NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    service/harbor                   ClusterIP   172.31.202.245   <none>        80/TCP,443/TCP,4443/TCP      3m19s
    service/harbor-1-chartmuseum     ClusterIP   172.31.6.127     <none>        80/TCP                       3m19s
    service/harbor-1-core            ClusterIP   172.31.17.10     <none>        80/TCP,8001/TCP              3m19s
    service/harbor-1-database        ClusterIP   172.31.204.175   <none>        5432/TCP                     3m19s
    service/harbor-1-exporter        ClusterIP   172.31.107.19    <none>        8001/TCP                     3m19s
    service/harbor-1-jobservice      ClusterIP   172.31.128.4     <none>        80/TCP,8001/TCP              3m19s
    service/harbor-1-notary-server   ClusterIP   172.31.240.223   <none>        4443/TCP                     3m19s
    service/harbor-1-notary-signer   ClusterIP   172.31.221.6     <none>        7899/TCP                     3m19s
    service/harbor-1-portal          ClusterIP   172.31.38.67     <none>        80/TCP                       3m19s
    service/harbor-1-redis           ClusterIP   172.31.38.202    <none>        6379/TCP                     3m19s
    service/harbor-1-registry        ClusterIP   172.31.16.8      <none>        5000/TCP,8080/TCP,8001/TCP   3m18s
    service/harbor-1-trivy           ClusterIP   172.31.70.71     <none>        8080/TCP                     3m18s
    
    NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/harbor-1-chartmuseum     1/1     1            1           3m18s
    deployment.apps/harbor-1-core            1/1     1            1           3m18s
    deployment.apps/harbor-1-exporter        0/1     1            0           3m18s
    deployment.apps/harbor-1-jobservice      1/1     1            1           3m18s
    deployment.apps/harbor-1-nginx           1/1     1            1           3m18s
    deployment.apps/harbor-1-notary-server   1/1     1            1           3m18s
    deployment.apps/harbor-1-notary-signer   1/1     1            1           3m18s
    deployment.apps/harbor-1-portal          1/1     1            1           3m18s
    deployment.apps/harbor-1-registry        1/1     1            1           3m18s
    
    NAME                                                DESIRED   CURRENT   READY   AGE
    replicaset.apps/harbor-1-chartmuseum-967bb456b      1         1         1       3m18s
    replicaset.apps/harbor-1-core-7956bd9f54            1         1         1       3m18s
    replicaset.apps/harbor-1-exporter-5c86cc7b6d        1         1         0       3m18s
    replicaset.apps/harbor-1-jobservice-9898c468b       1         1         1       3m18s
    replicaset.apps/harbor-1-nginx-66d4ccd87            1         1         1       3m18s
    replicaset.apps/harbor-1-notary-server-7d7b4b598d   1         1         1       3m18s
    replicaset.apps/harbor-1-notary-signer-5d59d79f4    1         1         1       3m18s
    replicaset.apps/harbor-1-portal-66f4b7478b          1         1         1       3m18s
    replicaset.apps/harbor-1-registry-67bc689779        1         1         1       3m18s
    
    NAME                                 READY   AGE
    statefulset.apps/harbor-1-database   1/1     3m18s
    statefulset.apps/harbor-1-redis      1/1     3m18s
    statefulset.apps/harbor-1-trivy      1/1     3m18s
    
    NAME                          COMPLETIONS   DURATION   AGE
    job.batch/harbor-1-deployer   1/1           44s        3m59s
    ```

    

23. 測試redmine

    ```shell
    [nutanix@anthos-netfos1-anthos-adminVm-0 redmine]$ kubectl get all -n redmine
    NAME                                               READY   STATUS    RESTARTS   AGE
    pod/redmine-58cfd55655-qthfz                       1/1     Running   0          38m
    pod/redmine-postgres-deployment-6b8f85c97b-72vkl   1/1     Running   0          106m
    
    NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
    service/redmine-postgres-service   NodePort    172.31.163.134   <none>          5432:31432/TCP   106m
    service/redmine-service            ClusterIP   172.31.151.36    172.16.90.213   80/TCP           38m
    
    NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/redmine                       1/1     1            1           38m
    deployment.apps/redmine-postgres-deployment   1/1     1            1           106m
    
    NAME                                                     DESIRED   CURRENT   READY   AGE
    replicaset.apps/redmine-58cfd55655                       1         1         1       38m
    replicaset.apps/redmine-postgres-deployment-6b8f85c97b   1         1         1       106m
    ```

24. IngressVIP : 172.16.90.213
    ![image-20230320170954778](https://kenkenny.synology.me:5543/images/2023/09/anthos-redmine.png)

25. anthos畫面
    ![image-20230320171120742](https://kenkenny.synology.me:5543/images/2023/09/anthos-redmine2.png)

    ![image-20230320171223253](https://kenkenny.synology.me:5543/images/2023/09/anthos-redmine3.png)

    StorageClass --> default block storage

    預設會是用Nutanix的iSCSI block storage，如果要使用NFS則要另外設定

    ![image-20230320171344271](https://kenkenny.synology.me:5543/images/2023/09/anthos-redmine4.png)

    

