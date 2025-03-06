---
title: OCP-4.16.3_on_Nutanix(IPI)
description: OCP-4.16.3_on_Nutanix(IPI)
slug: OCP-4.16.3_on_Nutanix(IPI)
date: 2025-03-06T03:05:48+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - Openshift
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# OCP 4.16.3_on_Nutanix (IPI)

整合測試 OCP 4.16.3 透過 IPI (Installer-provisioned Infrastructure) 安裝在 Nutanix

並且結合 Nutanix CSI、COSI、NDK (Nutanix Data service for kubernetes) 、 Autoscaler 自動擴展Worker



## 大致流程

1. 建立Bastion 
1. Prism Central 更換憑證
1. 建立 install-config.yaml , CCO
1. 建立 Openshift Cluster
1. 安裝Nutanix CSI 3.0 (beta)連線NUS
1. 安裝測試 Nutanix NDK (For VG Only)
1. 建立Machine Config ，worker node autoscaler



## Reference

Openshift:

https://docs.openshift.com/container-platform/4.16/installing/installing_nutanix/preparing-to-install-on-nutanix.html

Openshift Download:

https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.3/

Nutanix NVD for OCP:

https://portal.nutanix.com/page/documents/solutions/details?targetId=NVD-2177-Cloud-Native-6-5-OpenShift:NVD-2177-Cloud-Native-6-5-OpenShift

https://portal.nutanix.com/page/documents/solutions/details?targetId=TN-2030-Red-Hat-OpenShift-on-Nutanix:TN-2030-Red-Hat-OpenShift-on-Nutanix

Nutanix NDK:

https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Data-Services-for-Kubernetes-v1_0:Nutanix-Data-Services-for-Kubernetes-v1_0

Nutanix PC CA:

https://www.nutanix.dev/2023/09/26/generating-a-self-signed-san-certificate-to-use-red-hat-openshift-in-nutanix-marketplace/



## 架構與版本



| 名稱             | IP / DNS                                       | 版本                                |
| ---------------- | ---------------------------------------------- | ----------------------------------- |
| AOS / PE         | 172.16.90.74                                   | 6.8                                 |
| Prism Central    | 172.16.90.75                                   | pc2024.1.0.1                        |
| NDK              | none                                           | 1.0.0                               |
| Nutanix CSI      | none                                           | nutanix-csi-storage-3.0.0-beta.1912 |
| Nutanix Object   | none                                           | 5.0                                 |
| Nutanix IPAM     | 172.16.90.x                                    | none                                |
| bastion          | 172.16.90.206                                  | RHEL 9.3                            |
| OCP Cluster Name | ocplab.nutanixlab.local                        | 4.16.3                              |
| OCP API VIP      | 172.16.90.205 / api.ocplab.nutanixlab.local    | none                                |
| OCP Ingress VIP  | 172.16.90.207 / *.apps.ocplab.nutanixlab.local | none                                |

![image-20240716100928430](https://kenkenny.synology.me:5543/images/2024/07/image-20240716100928430.png)

![image-20240716094638388](https://kenkenny.synology.me:5543/images/2024/07/image-20240716094638388.png)





## OCP Installation

### Prepare Bastion

1. 準備一台 RHEL VM 

   ```
   [nutanix@ken-rhel9 ~]$ cat /etc/os-release 
   NAME="Red Hat Enterprise Linux"
   VERSION="9.3 (Plow)"
   ```

2. 下載安裝檔 ccoctl , oc , openshift-install , 

   ```
   # 建立安裝目錄
   $ mkdir ocp-install-4-16-3
   $ cd ocp-install-4-16-3/
   
   # 下載檔案
   $ wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.3/ccoctl-linux-rhel9-4.16.3.tar.gz
   
   $ wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.3/openshift-client-linux-amd64-rhel9-4.16.3.tar.gz
   
   $ wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.3/openshift-install-linux-4.16.3.tar.gz
   
   # 解壓縮檔案
   $ tar xvf ccoctl-linux-rhel9-4.16.3.tar.gz 
   $ tar xvf openshift-client-linux-amd64-rhel9-4.16.3.tar.gz 
   $ tar xvf openshift-install-linux-4.16.3.tar.gz 
   
   $ ll
   total 1689088
   -rwxr-xr-x. 1 nutanix nutanix  89185776 Jul  4 05:23 ccoctl
   -rw-r--r--. 1 nutanix nutanix  35912054 Jul 12 03:52 ccoctl-linux-rhel9-4.16.3.tar.gz
   -rwxr-xr-x. 2 nutanix nutanix 159916496 Jul  9 04:50 kubectl
   -rwxr-xr-x. 2 nutanix nutanix 159916496 Jul  9 04:50 oc
   -rw-r--r--. 1 nutanix nutanix  66698442 Jul 12 03:52 openshift-client-linux-amd64-rhel9-4.16.3.tar.gz
   -rwxr-xr-x. 1 nutanix nutanix 707719168 Jul  9 20:10 openshift-install
   -rw-r--r--. 1 nutanix nutanix 510267167 Jul 12 03:52 openshift-install-linux-4.16.3.tar.gz
   
   $ ./oc version
   Client Version: 4.16.3
   Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
   
   $ ./openshift-install version
   ./openshift-install 4.16.3
   built from commit e1f9f057ce87c1a4a5f3c268812fa4c9dc003cb7
   release image quay.io/openshift-release-dev/ocp-release@sha256:3ec3a43ded1decc18134e5677f56037d8929f4442930f5d1156e7a77cdf1b9b3
   release architecture amd64
   
   ```




### Prism Central CA



1. 產生 CA 憑證

   ```
   $ export PC_IP=172.16.90.75
   $ export PC_FQDN=ntnxselabpc.nutanixlab.local
   
   $ openssl req -x509 -nodes -days 3650 \
   -newkey rsa:2048 -keyout ${PC_IP}.key -out ${PC_IP}.crt \
   -subj "/C=US/ST=CA/L=San Jose/O=Nutanix Inc./OU=Manageability/CN=*.nutanix.local" \
   -addext "subjectAltName=IP:${PC_IP},DNS:${PC_FQDN}"
   
   $ ll
   total 1689096
   -rw-r--r--. 1 nutanix nutanix      1444 Jul 16 14:48 172.16.90.75.crt
   -rw-------. 1 nutanix nutanix      1704 Jul 16 14:48 172.16.90.75.key
   ```

   

2. 置換 Prism Central CA

   ![image-20240716144747534](https://kenkenny.synology.me:5543/images/2024/07/image-20240716144747534.png)

   ![image-20240716144805568](https://kenkenny.synology.me:5543/images/2024/07/image-20240716144805568.png)

   ![image-20240716144834705](https://kenkenny.synology.me:5543/images/2024/07/image-20240716144834705.png)

3. 檢視憑證

   ![image-20240716144925561](https://kenkenny.synology.me:5543/images/2024/07/image-20240716144925561.png)

4. 將憑證匯入至Bastion

   ```
   $ sudo cp 172.16.90.75.* /etc/pki/ca-trust/source/anchors
   $ sudo update-ca-trust extract
   ```
   



### Install-config.yaml

1. 建立install-config.yaml

   ```
   # 建立安裝目錄
   $ mkdir install
   
   # 產生install-config.yaml
   
   $ ./openshift-install create install-config --dir /home/nutanix/ocp-install-4-16-3/install
   ? SSH Public Key /home/nutanix/.ssh/id_rsa.pub
   ? Platform nutanix
   ? Prism Central 172.16.90.75
   ? Port 9440
   ? Username ocpadmin
   ? Password [? for help] ***********
   INFO Connecting to Prism Central 172.16.90.75     
   ? Prism Element NX1365G6PE
   ? Subnet Netfos_90_Access_IPAM
   ? Virtual IP Address for API 172.16.90.205
   ? Virtual IP Address for Ingress 172.16.90.207
   ? Base Domain nutanixlab.local
   ? Cluster Name ocplab
   ? Pull Secret [? for help] *****************************************************************************************************************INFO Install-Config created in: /home/nutanix/ocp-install-4-16-3/install
   
   $ ll install/
   total 8
   -rw-r-----. 1 nutanix nutanix 4374 Jul 16 14:59 install-config.yaml
   ```

   

2. 修改檔案

   additionalTrustBundle: 加入 Prism Central 的 root CA

   additionalTrustBundlePolicy: 改為 Always

   machineNetwork: 改為與VM相同網段 xxx.xxx.xxx.xxx/24

   platform 自定義cpu、memory、disk

   完成後如下，建議備份此檔案以利部署到其他地方或是重新部署

   ```
   $ cat install-config.yaml 
   additionalTrustBundlePolicy: Always
   additionalTrustBundle: |
   -----BEGIN CERTIFICATE-----
   MIID/jCCAuagAwIBAgIUKDpX++EriXrP4L1R+e15qqR3+4owDQYJKoZIhvcNAQEL
   BQAwdjELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAkNBMREwDwYDVQQHDAhTYW4gSm9z
   ZTEVMBMGxxx....
   -----END CERTIFICATE-----
   apiVersion: v1
   baseDomain: nutanixlab.local
   compute:
   - architecture: amd64
     hyperthreading: Enabled
     name: worker
     platform:
       nutanix:
         cpus: 2
         coresPerSocket: 2
         memoryMiB: 8196
         osDisk:
           diskSizeGiB: 120
         categories:
         - key: ocplab
           value: worker
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
         categories:
         - key: ocpalb
           value: master
     replicas: 3
   credentialsMode: Manual
   metadata:
     creationTimestamp: null
     name: ocplab
   networking:
     clusterNetwork:
     - cidr: 10.128.0.0/14
       hostPrefix: 23
     machineNetwork:
     - cidr: 172.16.90.0/24
     networkType: OVNKubernetes
     serviceNetwork:
     - 172.30.0.0/16
   platform:
     nutanix:
       apiVIPs:
       - 172.16.90.205
       ingressVIPs:
       - 172.16.90.207
       prismCentral:
         endpoint:
           address: 172.16.90.75
           port: 9440
         password: password
         username: ocpadmin
       prismElements:
       - endpoint:
           address: 172.16.90.74
           port: 9440
         uuid: 00061972-aeba-fee2-0000-000000028e95
       subnetUUIDs:
       - 060837bb-aefc-41f2-aaae-25c95267d87a
   publish: External
   pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnN....'
   sshKey: |
     ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCy4sUIXB5QMIETJe6+niy4SrO0ly/ymlwBtnCbgAPNtIKfQfkjFrbcUgsdUsjEQ+M0giXfjlgDcAMiasxaFvfhKCNJ1dR5L6hyi/JrWAdG/Ue5ydbc84vFgQRQ1AolnovlxWcM4xdsYYQvpZwo9v1dOIAQ9rasubEiDxHaw7HK+mGChu+6uRli4EdJ7HvRg9Ha5dKAQFawsZ/cBcMZZnW84bfK72naHI7XyVOEPnbQ8NG+Pk55S2za6J7VIIHDr3rSbNYBZUJnTroYMLxXnC0I0xqcXZMDiUm6ZQzE6kJnrFHIdqVEXkgqtbdXQDK+zdeA/vOttq8ojYSFpa9qCHMCHK66OC2P0vE54hI439uEHPJI7aIYNGPxPJtL6k8rDklWyMnkh8jJMtxM+sZkZxI2/+L4szC/S4qJ/OmRJI3TeTob+0kptqwJrAuWK3T/YiCWbyqytr4RyGJTFA5pB0SucJHaqJ4PbiJ633hJ8AJ7KepYHB65KyvHgDga2v3QDf0= nutanix@ken-rhel9.nutanixlab.local
   ```

   

### Cloud Credential Operator 

安裝Cluster需要Cloud Credential Operator (CCO)，讓OCP能與Prism溝通

大致作法為

1. 建立帳號密碼檔案
2. 建立要求帳密的目錄及檔案
3. 利用上述檔案建立OCP Shared Secret




1. 建立cred.yaml

   ```
   $ vim credentials.yaml
   
   credentials:
   - type: basic_auth
     data:
       prismCentral:
         username: ocpadmin
         password: password
       prismElements:
       - name: 
         username: ocpadmin
         password: password
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
   $ ./oc adm release extract --credentials-requests --cloud=nutanix --to=./credrequests  $RELEASE_IMAGE
   
   warning: if you intend to pass CredentialsRequests to ccoctl, you should use --included to filter out requests that your cluster is not expected to need.
   Extracted release payload created at 2024-07-11T11:19:04Z
   
   $ ll credrequests/
   total 8
   -rw-r--r--. 1 nutanix nutanix 599 Jul 16 15:43 0000_26_cloud-controller-manager-operator_18_credentialsrequest-nutanix.yaml
   -rw-r--r--. 1 nutanix nutanix 543 Jul 16 15:43 0000_30_machine-api-operator_00_credentials-request.yaml
   ```

   

5. 建立 CCO Secret

   ```
   $ ./ccoctl nutanix create-shared-secrets --credentials-requests-dir=/home/nutanix/ocp-install-4-16-3/credrequests --output-dir=/home/nutanix/ocp-install-4-16-3/install --credentials-source-filepath=/home/nutanix/ocp-install-4-16-3/install/credentials.yaml
   
   2024/07/16 15:45:45 Saved credentials configuration to: /home/nutanix/ocp-install-4-16-3/install/manifests/openshift-cloud-controller-manager-nutanix-credentials-credentials.yaml
   2024/07/16 15:45:45 Saved credentials configuration to: /home/nutanix/ocp-install-4-16-3/install/manifests/openshift-machine-api-nutanix-credentials-credentials.yaml
   
   $ tree install
   install
   ├── credentials.yaml
   ├── install-config.yaml
   └── manifests
       ├── openshift-cloud-controller-manager-nutanix-credentials-credentials.yaml
       └── openshift-machine-api-nutanix-credentials-credentials.yaml
   
   1 directory, 4 files
   ```



### Create Cluster

1. Create manifests

   ```
   $ ./openshift-install create manifests --dir /home/nutanix/ocp-install-4-16-3/install
   
   INFO Consuming Install Config from target directory 
   INFO Manifests created in: /home/nutanix/ocp-install-4-16-3/install/cluster-api, /home/nutanix/ocp-install-4-16-3/install/manifests and /home/nutanix/ocp-install-4-16-3/install/openshift 
   
   $ tree install
   install
   ├── cluster-api
   │   ├── 000_capi-namespace.yaml
   │   ├── 01_capi-cluster.yaml
   │   ├── 01_nutanix-cluster.yaml
   │   ├── 01_nutanix-creds.yaml
   │   └── machines
   │       ├── 10_inframachine_ocplab-k7zds-bootstrap.yaml
   │       ├── 10_inframachine_ocplab-k7zds-master-0.yaml
   │       ├── 10_inframachine_ocplab-k7zds-master-1.yaml
   │       ├── 10_inframachine_ocplab-k7zds-master-2.yaml
   │       ├── 10_machine_ocplab-k7zds-bootstrap.yaml
   │       ├── 10_machine_ocplab-k7zds-master-0.yaml
   │       ├── 10_machine_ocplab-k7zds-master-1.yaml
   │       └── 10_machine_ocplab-k7zds-master-2.yaml
   ├── credentials.yaml
   ├── manifests
   │   ├── cloud-provider-config.yaml
   │   ├── cluster-config.yaml
   │   ├── cluster-dns-02-config.yml
   │   ├── cluster-infrastructure-02-config.yml
   │   ├── cluster-ingress-02-config.yml
   │   ├── cluster-network-02-config.yml
   │   ├── cluster-proxy-01-config.yaml
   │   ├── cluster-scheduler-02-config.yml
   │   ├── cvo-overrides.yaml
   │   ├── kube-cloud-config.yaml
   │   ├── kube-system-configmap-root-ca.yaml
   │   ├── machine-config-server-tls-secret.yaml
   │   ├── openshift-cloud-controller-manager-nutanix-credentials-credentials.yaml
   │   ├── openshift-config-secret-pull-secret.yaml
   │   ├── openshift-machine-api-nutanix-credentials-credentials.yaml
   │   └── user-ca-bundle-config.yaml
   └── openshift
       ├── 99_feature-gate.yaml
       ├── 99_kubeadmin-password-secret.yaml
       ├── 99_openshift-cluster-api_master-machines-0.yaml
       ├── 99_openshift-cluster-api_master-machines-1.yaml
       ├── 99_openshift-cluster-api_master-machines-2.yaml
       ├── 99_openshift-cluster-api_master-user-data-secret.yaml
       ├── 99_openshift-cluster-api_worker-machineset-0.yaml
       ├── 99_openshift-cluster-api_worker-user-data-secret.yaml
       ├── 99_openshift-machine-api_master-control-plane-machine-set.yaml
       ├── 99_openshift-machineconfig_99-master-ssh.yaml
       ├── 99_openshift-machineconfig_99-worker-ssh.yaml
       └── openshift-install-manifests.yaml
   
   4 directories, 41 files
   
   ```

   

2. Create Cluster

   ```
   $ ./openshift-install create cluster --dir /home/nutanix/ocp-install-4-16-3/install --log-level=debug
   
   DEBUG Generating Cluster...                        
   INFO Creating infrastructure resources...         
   INFO created the category key "kubernetes-io-cluster-ocplab-k7zds" 
   INFO created the category value "owned" with name "kubernetes-io-cluster-ocplab-k7zds" 
   INFO created the category value "shared" with name "kubernetes-io-cluster-ocplab-k7zds" 
   INFO creating the rhcos image ocplab-k7zds-rhcos (uuid: 1c8db94e-17aa-4eea-bc75-2e05876e8f31). 
   INFO waiting the image data uploading from https://rhcos.mirror.openshift.com/art/storage/prod/streams/4.16-9.4/builds/416.94.202406251923-0/x86_64/rhcos-416.94.202406251923-0-nutanix.x86_64.qcow2?sha256=, taskUUID: cf6ef687-c95d-4648-b97e-c0bccfcb3052. 
   ...
   
   INFO Network infrastructure is ready              
   DEBUG No infrastructure ready requirements for the nutanix provider 
   INFO creating the image ocplab-k7zds-bootstrap-ign.iso (uuid: 466b5db5-69e4-4f39-aaac-9ae65594a09f), taskUUID: 04588942-57c9-4b0b-8bb2-321e6eaf60ad 
   INFO created the image ocplab-k7zds-bootstrap-ign.iso (uuid: 466b5db5-69e4-4f39-aaac-9ae65594a09f). used_time 10.829777095s 
   INFO preparing to upload the image ocplab-k7zds-bootstrap-ign.iso (uuid: 466b5db5-69e4-4f39-aaac-9ae65594a09f) data from file /home/nutanix/.cache/openshift-installer/image_cache/ocplab-k7zds-bootstrap-ign.iso 
   ...
   
   INFO All cluster operators have completed progressing 
   INFO Checking to see if there is a route at openshift-console/console... 
   DEBUG Route found in openshift-console namespace: console 
   DEBUG OpenShift console route is admitted          
   INFO Install complete!                            
   INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/nutanix/ocp-install-4-16-3/install/auth/kubeconfig' 
   INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocplab.nutanixlab.local 
   INFO Login to the console with user: "kubeadmin", and password: "JrHfc-s958g-V3oQ7-eXHxY" 
   DEBUG Time elapsed per stage:                      
   DEBUG     Infrastructure Pre-provisioning: 2m6s    
   DEBUG Network-infrastructure Provisioning: 54s     
   DEBUG     Bootstrap Ignition Provisioning: 18s     
   DEBUG                Machine Provisioning: 30s     
   DEBUG                  Bootstrap Complete: 22m27s  
   DEBUG                                 API: 4m0s    
   DEBUG                   Bootstrap Destroy: 11s     
   DEBUG         Cluster Operators Available: 12m37s  
   DEBUG            Cluster Operators Stable: 20s     
   INFO Time elapsed: 39m33s     
   ```

   ![image-20240716155629530](https://kenkenny.synology.me:5543/images/2024/07/image-20240716155629530.png)

   ![image-20240716163717437](https://kenkenny.synology.me:5543/images/2024/07/image-20240716163717437.png)

3. 安裝完畢確認連線

   ![image-20240716163827676](https://kenkenny.synology.me:5543/images/2024/07/image-20240716163827676.png)

   ![image-20240716163936258](https://kenkenny.synology.me:5543/images/2024/07/image-20240716163936258.png)

   ![image-20240716163947487](https://kenkenny.synology.me:5543/images/2024/07/image-20240716163947487.png)

4. 指令確認

   ```
   $ cp /home/nutanix/ocp-install-4-16-3/install/auth/kubeconfig ~/.kube/config
   $ sudo cp oc /usr/local/bin/
   [sudo] password for nutanix: 
   $ oc get nodes
   NAME                        STATUS   ROLES                  AGE   VERSION
   ocplab-k7zds-master-0       Ready    control-plane,master   34m   v1.29.6+aba1e8d
   ocplab-k7zds-master-1       Ready    control-plane,master   34m   v1.29.6+aba1e8d
   ocplab-k7zds-master-2       Ready    control-plane,master   34m   v1.29.6+aba1e8d
   ocplab-k7zds-worker-27snp   Ready    worker                 19m   v1.29.6+aba1e8d
   ocplab-k7zds-worker-29wr6   Ready    worker                 19m   v1.29.6+aba1e8d
   ocplab-k7zds-worker-bdgmd   Ready    worker                 19m   v1.29.6+aba1e8d
   
   $ oc get nodes -o wide
   NAME                        STATUS   ROLES                  AGE   VERSION           INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                                                KERNEL-VERSION                 CONTAINER-RUNTIME
   ocplab-k7zds-master-0       Ready    control-plane,master   34m   v1.29.6+aba1e8d   172.16.90.158   <none>        Red Hat Enterprise Linux CoreOS 416.94.202407081958-0   5.14.0-427.26.1.el9_4.x86_64   cri-o://1.29.6-3.rhaos4.16.gitfd433b7.el9
   ocplab-k7zds-master-1       Ready    control-plane,master   34m   v1.29.6+aba1e8d   172.16.90.175   <none>        Red Hat Enterprise Linux CoreOS 416.94.202407081958-0   5.14.0-427.26.1.el9_4.x86_64   cri-o://1.29.6-3.rhaos4.16.gitfd433b7.el9
   ocplab-k7zds-master-2       Ready    control-plane,master   34m   v1.29.6+aba1e8d   172.16.90.151   <none>        Red Hat Enterprise Linux CoreOS 416.94.202407081958-0   5.14.0-427.26.1.el9_4.x86_64   cri-o://1.29.6-3.rhaos4.16.gitfd433b7.el9
   ocplab-k7zds-worker-27snp   Ready    worker                 19m   v1.29.6+aba1e8d   172.16.90.154   <none>        Red Hat Enterprise Linux CoreOS 416.94.202407081958-0   5.14.0-427.26.1.el9_4.x86_64   cri-o://1.29.6-3.rhaos4.16.gitfd433b7.el9
   ocplab-k7zds-worker-29wr6   Ready    worker                 20m   v1.29.6+aba1e8d   172.16.90.167   <none>        Red Hat Enterprise Linux CoreOS 416.94.202407081958-0   5.14.0-427.26.1.el9_4.x86_64   cri-o://1.29.6-3.rhaos4.16.gitfd433b7.el9
   ocplab-k7zds-worker-bdgmd   Ready    worker                 20m   v1.29.6+aba1e8d   172.16.90.160   <none>        Red Hat Enterprise Linux CoreOS 416.94.202407081958-0   5.14.0-427.26.1.el9_4.x86_64   cri-o://1.29.6-3.rhaos4.16.gitfd433b7.el9
   ```



## 安裝 Nutanix CSI 、COSI

### Nutanix CSI 3.0

1. 先安裝Helm Chart 在Bastion上面

   ```
   $ sudo curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64 -o /usr/local/bin/helm
   
   $ sudo chmod +x /usr/local/bin/helm
   $ helm version
   WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
   version.BuildInfo{Version:"v3.13.2+35.el9", GitCommit:"fa6e939d7984e1be0d6fbc2dc920b6bbcf395932", GitTreeState:"clean", GoVersion:"go1.20.12"}
   
   $ helm repo add nutanix https://nutanix.github.io/helm/
   ```

   

2. 建立 Nutanix Secret 在 ntnx-system 內

   ```
   # Create Namespace ntnx-system
   $ oc create ns ntnx-system
   namespace/ntnx-system created
   
   # Edit Secret for PE,PC
   $ vi ntnx-secret.yaml
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: ntnx-secret
     namespace: ntnx-system
   stringData:
     # prism-element-ip:prism-port:admin:password
     key: pe-ip:9440:admin:password
     
   $ vi ntnx-pc-secret.yaml
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: ntnx-pc-secret
     namespace: ntnx-system
   stringData:
     # prism-pc-ip:prism-port:admin:password
     key: pc-ip:9440:admin:password
    
   ---
   # Create secret
   
   $ oc apply -f ntnx-pc-secret.yaml
   $ oc apply -f ntnx-secret.yaml
   
   $ oc get secret -n ntnx-system
   NAME             TYPE     DATA   AGE
   ntnx-pc-secret   Opaque   1      13s
   ntnx-secret      Opaque   1      9s
   ```

   

3. 下載 Nutanix CSI 3.0 beta

   ```
   $ wget https://github.com/nutanix/helm-releases/releases/download/nutanix-csi-storage/nutanix-csi-storage-3.0.0-beta.1912.tgz
   
   $ tar -xvf nutanix-csi-storage-3.0.0-beta.1912.tgz
   nutanix-csi-storage/Chart.yaml
   nutanix-csi-storage/values.yaml
   nutanix-csi-storage/templates/NOTES.txt
   nutanix-csi-storage/templates/_helpers.tpl
   nutanix-csi-storage/templates/csi-driver.yaml
   nutanix-csi-storage/templates/machine-config.yaml
   nutanix-csi-storage/templates/ntnx-csi-controller-deployment.yaml
   nutanix-csi-storage/templates/ntnx-csi-init-configmap.yaml
   nutanix-csi-storage/templates/ntnx-csi-node-ds.yaml
   nutanix-csi-storage/templates/ntnx-csi-rbac.yaml
   nutanix-csi-storage/templates/ntnx-csi-scc.yaml
   nutanix-csi-storage/templates/ntnx-sc.yaml
   nutanix-csi-storage/templates/ntnx-secret.yaml
   nutanix-csi-storage/templates/service-prometheus-csi.yaml
   nutanix-csi-storage/.helmignore
   nutanix-csi-storage/Nutanix core_k8s-csi_beta1 Notice.txt
   nutanix-csi-storage/README.md
   nutanix-csi-storage/questions.yml
   ```

   

4. Edit value.yaml

   ```
   $ cd nutanix-csi-storage
   $ vim values.yaml
   
   ---
   # all namespace
   namespace: ntnx-system
   
   createPrismCentralSecret: false
   # prismCentralEndPoint: 00.00.00.00
   # pcUsername: username
   # pcPassword: password
   pcSecretName: ntnx-pc-secret
   
   createSecret: false
   # prismEndPoint: 00.00.000.000
   # username: username
   # password: password
   peSecretName: ntnx-secret
   
   #supportedPCVersions: "fraser-2023.4-stable-pc-0"
   supportedPCVersions: "fraser-2024.1-stable-pc-0.1"
   --
   ```

   

5. Install CSI Driver using Helm

   ```
   $ helm install -n ntnx-system -f nutanix-csi-storage/values.yaml nutanix-csi ./nutanix-csi-storage
   
   $ oc get all -n ntnx-system
   Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
   NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                               AGE
   service/nutanix-csi-metrics   ClusterIP   172.30.52.86   <none>        9809/TCP,9810/TCP,9811/TCP,9812/TCP   2m43s
   
   NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
   daemonset.apps/nutanix-csi-node   3         0         0       0            0           kubernetes.io/os=linux   2m43s
   
   NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/nutanix-csi-controller   0/2     0            0           2m43s
   
   NAME                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/nutanix-csi-controller-8669b97dc8   2         0         0       2m43s
   
   
   $ oc describe daemonset.apps/nutanix-csi-node
   ----                  -------
     Warning  FailedCreate  12m (x19 over 23m)  daemonset-controller  Error creating: pods "nutanix-csi-node-" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount
   ```

   

6. Apply privilege to ntnx-system namespace

   ```
   $ oc adm policy add-scc-to-user privileged -z nutanix-csi-controller -n ntnx-system
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "nutanix-csi-controller"
   
   $ oc adm policy add-scc-to-user privileged -z nutanix-csi-node -n ntnx-system
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "nutanix-csi-node"
   
   $ oc adm policy add-scc-to-user privileged -z node-exporter -n ntnx-system
   
   $ oc adm policy add-scc-to-user anyuid -z nutanix-csi-controller -n ntnx-system
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "nutanix-csi-controller"
   
   $ oc adm policy add-scc-to-user anyuid -z nutanix-csi-node -n ntnx-system
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "nutanix-csi-node"
   
   $ oc adm policy add-scc-to-user anyuid -z node-exporter -n ntnx-system
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "node-exporter"
   
   # reinstall 
   $ helm uninstall -n ntnx-system nutanix-csi
   $ helm install -n ntnx-system -f nutanix-csi-storage/values.yaml nutanix-csi ./nutanix-csi-storage
   
   ```

   

7. 確認安裝完成

   ```
   $ oc get all -n ntnx-system
   Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
   NAME                                          READY   STATUS    RESTARTS   AGE
   pod/nutanix-csi-controller-8669b97dc8-pwcq4   7/7     Running   0          9s
   pod/nutanix-csi-controller-8669b97dc8-srkbr   7/7     Running   0          9s
   pod/nutanix-csi-node-gz278                    3/3     Running   0          9s
   pod/nutanix-csi-node-p6lrb                    3/3     Running   0          9s
   pod/nutanix-csi-node-qdrdp                    3/3     Running   0          9s
   
   NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
   service/nutanix-csi-metrics   ClusterIP   172.30.34.177   <none>        9809/TCP,9810/TCP,9811/TCP,9812/TCP   9s
   
   NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
   daemonset.apps/nutanix-csi-node   3         3         3       3            3           kubernetes.io/os=linux   9s
   
   NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/nutanix-csi-controller   2/2     2            2           9s
   
   NAME                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/nutanix-csi-controller-8669b97dc8   2         2         2       9s
   ```

   ![image-20240716172840878](https://kenkenny.synology.me:5543/images/2024/07/image-20240716172840878.png)

#### 連線 Nutanix Volume

1. Create nutanix-volume.yaml

   ```
   $ vim nutanix-volume.yaml
   
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     annotations:
       storageclass.kubernetes.io/is-default-class: "true"
     name: nutanixvolume
   provisioner: csi.nutanix.com
   parameters:
     csi.storage.k8s.io/provisioner-secret-name: ntnx-secret
     csi.storage.k8s.io/provisioner-secret-namespace: ntnx-system
     csi.storage.k8s.io/node-publish-secret-name: ntnx-secret
     csi.storage.k8s.io/node-publish-secret-namespace: ntnx-system
     csi.storage.k8s.io/controller-expand-secret-name: ntnx-secret
     csi.storage.k8s.io/controller-expand-secret-namespace: ntnx-systems
     csi.storage.k8s.io/fstype: ext4
     dataServiceEndPoint: 172.16.90.76:3260 # Prism data service ip
     storageContainer: NX_1365_AHV_cr # Prism 上建立的Storage Container
     storageType: NutanixVolumes
     #whitelistIPMode: ENABLED
     #chapAuth: ENABLED
   allowVolumeExpansion: true
   reclaimPolicy: Delete
   ```

2. apply nutanix-volume.yaml

   ```
   $ oc apply -f nutanix-volume.yaml -n ntnx-system
   storageclass.storage.k8s.io/nutanix-volume created
   
   $ oc get storageclass
   NAME                       PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   nutanix-volume (default)   csi.nutanix.com   Delete          Immediate           true                   45s
   ```

   

3. 測試VG

   ```
   $ oc get pv,pvc -n ntnx-system
   NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS     VOLUMEATTRIBUTESCLASS   REASON   AGE
   persistentvolume/pvc-49b7f489-05a1-479a-ada3-a1cd2ae76bfa   1Gi        RWO            Delete           Bound    ntnx-system/test-vg   nutanix-volume   <unset>                          77s
   
   NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     VOLUMEATTRIBUTESCLASS   AGE
   persistentvolumeclaim/test-vg   Bound    pvc-49b7f489-05a1-479a-ada3-a1cd2ae76bfa   1Gi        RWO            nutanix-volume   <unset>                 110s
   
   ```

   ![image-20240716174552557](https://kenkenny.synology.me:5543/images/2024/07/image-20240716174552557.png)

   ![image-20240716174543937](https://kenkenny.synology.me:5543/images/2024/07/image-20240716174543937.png)
   ![image-20240716174532000](https://kenkenny.synology.me:5543/images/2024/07/image-20240716174532000.png)

   



