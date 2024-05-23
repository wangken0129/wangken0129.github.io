---
title: Nutanix_NDK-v1.0.0_On_OCP-4.13
description: Nutanix_NDK-v1.0.0_On_OCP-4.13
slug: Nutanix_NDK-v1.0.0_On_OCP-4.13
date: 2024-05-23T08:00:46+08:00
categories:
    - Lab Category
tags:
    - Kubernetes
    - Nutanix
    - Openshift
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# NDK v1.0.0 on OCP v4.13

NDK ( Nutanix Data Service for Kubernetes ) ：針對Kubernetes 的容器備份還原

目前備援功能只支援在同一個供應商提供的 Kubernetes 叢集，且目前只有Volume 的PVC才可快照

排程備份只支援Async ，此測試僅有針對 Namespace 內全部的項目做手動快照還原 

OCP IPI 安裝此篇不贅述，請參考其他篇



2024.05.23

## Reference

https://portal.nutanix.com/page/documents/solutions/details?targetId=TN-2030-Red-Hat-OpenShift-on-Nutanix:TN-2030-Red-Hat-OpenShift-on-Nutanix

https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Data-Services-for-Kubernetes-v1_0:Nutanix-Data-Services-for-Kubernetes-v1_0

https://portal.nutanix.com/page/documents/details?targetId=Release-Notes-Nutanix-Data-Services-for-Kubernetes-v1_0:Release-Notes-Nutanix-Data-Services-for-Kubernetes-v1_0

https://github.com/nutanix/helm/tree/nutanix-csi-snapshot-6.3.2/charts/nutanix-csi-snapshot#install

https://github.com/nutanix/helm/tree/csi-v3.0.0-beta-beta1/charts/nutanix-csi-storage

https://access.redhat.com/documentation/zh-tw/openshift_container_platform/4.13/html/building_applications/working-with-helm-charts

https://blog.csdn.net/u010918487/article/details/115509935

https://portal.nutanix.com/page/documents/kbs/details?targetId=kA0VO00000008Qb0AI

https://quay.io/repository/karbon/k8s-agent?tab=tags

https://github.com/nutanix/helm-releases/releases

https://github.com/orgs/nutanix-cloud-native/repositories?type=all

## Requirements

NKE v2.9 以上

Prism Central 2023.4 以上

Kubernetes version v1.26 以上

Nutanix CSI Driver v3.0 (beta) 以上 

![image-20240523120449110](https://kenkenny.synology.me:5543/images/2024/05/image-20240523120449110.png)

![image-20240515115102568](https://kenkenny.synology.me:5543/images/2024/05/image-20240515115102568.png)

## Lab Enviroments

Prism Central : 2024.1

AOS: 6.7.1

NKE: 2.9

OpenShift : 4.13

Openshift IPI on Nutanix AHV            



## NDK install Lab

### install helm chart on bastion

https://access.redhat.com/documentation/zh-tw/openshift_container_platform/4.13/html/building_applications/working-with-helm-charts

```
$ sudo curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64 -o /usr/local/bin/helm

$ sudo chmod +x /usr/local/bin/helm
$ helm version
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
version.BuildInfo{Version:"v3.13.2+35.el9", GitCommit:"fa6e939d7984e1be0d6fbc2dc920b6bbcf395932", GitTreeState:"clean", GoVersion:"go1.20.12"}

$ helm repo add nutanix https://nutanix.github.io/helm/
```

### Nutanix  CSI 3.0 (beta)

#### Create OCP Secret

```
Create Secret pc and pe
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

$ oc apply -f ntnx-pc-secret.yaml
$ oc apply -f ntnx-secret.yaml
```

#### Install CSI Driver

1. Download CSI Driver and Edit value.yaml

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

2. Edit values.yaml

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
   ```

   

3. Install CSI Driver using helm

   ```
   $ helm install -n ntnx-system -f nutanix-csi-storage/values.yaml nutanix-csi ./nutanix-csi-storage
   WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
   NAME: nutanix-csi
   LAST DEPLOYED: Mon May 20 16:38:01 2024
   NAMESPACE: ntnx-system
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   Driver name: csi.nutanix.com
   
   Nutanix CSI provider was deployed in namespace ntnx-system. Check it's status by running:
   kubectl -n ntnx-system get pods | grep 'nutanix-csi'
   
   
   
   
   $ oc describe daemonset.apps/nutanix-csi-node
   
   ...
   ----                  -------
     Warning  FailedCreate  12m (x19 over 23m)  daemonset-controller  Error creating: pods "nutanix-csi-node-" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount
   ```

   

4. Apply privilege to openshift-cluster-csi-drivers namespace

   ```
   $ oc project ntnx-system
   Now using project "ntnx-system" on server "https://api.ocplab.nutanixlab.local:6443".
   
   $ oc adm policy add-scc-to-user privileged -z nutanix-csi-controller
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "nutanix-csi-controller"
   
   $ oc adm policy add-scc-to-user privileged -z nutanix-csi-node
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "nutanix-csi-node"
   
   $ oc adm policy add-scc-to-user privileged -z node-exporter
   
   $ oc adm policy add-scc-to-user anyuid -z nutanix-csi-controller
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "nutanix-csi-controller"
   
   $ oc adm policy add-scc-to-user anyuid -z nutanix-csi-node
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "nutanix-csi-node"
   
   ```

   

5. Check CSI installed

   ```
   $ oc get all
   NAME                                          READY   STATUS    RESTARTS      AGE
   pod/nutanix-csi-controller-844589fc9f-4jr5m   7/7     Running   3 (16h ago)   16h
   pod/nutanix-csi-controller-844589fc9f-wwq7m   7/7     Running   4 (16h ago)   16h
   pod/nutanix-csi-node-fkmdd                    3/3     Running   0             16h
   pod/nutanix-csi-node-hjnwg                    3/3     Running   0             16h
   pod/nutanix-csi-node-pwqwl                    3/3     Running   0             16h
   
   NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
   service/nutanix-csi-metrics   ClusterIP   172.30.168.77   <none>        9809/TCP,9810/TCP,9811/TCP,9812/TCP   17h
   
   NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
   daemonset.apps/nutanix-csi-node   3         3         3       3            3           kubernetes.io/os=linux   17h
   
   NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/nutanix-csi-controller   2/2     2            2           17h
   
   NAME                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/nutanix-csi-controller-844589fc9f   2         2         2       17h
   
   $ oc describe daemonset.apps/nutanix-csi-node
   
     Normal   SuccessfulCreate  40s                    daemonset-controller  Created pod: nutanix-csi-node-cgdvs
     Normal   SuccessfulCreate  40s                    daemonset-controller  Created pod: nutanix-csi-node-c49f5
     Normal   SuccessfulCreate  40s                    daemonset-controller  Created pod: nutanix-csi-node-4j69r
   ```

   

6. OCP Console

   ![image-20240509134700168](https://kenkenny.synology.me:5543/images/2024/05/image-20240509134700168.png)

#### Create StorageClass

1. create nutanix-volume.yaml 

2. oc apply -f nutanix-volume.yaml

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
     csi.storage.k8s.io/controller-expand-secret-namespace: ntnx-system
     csi.storage.k8s.io/fstype: ext4
     dataServiceEndPoint: IP:3260 # Prism data service ip
     storageContainer: NX_1365_AHV_cr # Prism 上建立的Storage Container
     storageType: NutanixVolumes
     #whitelistIPMode: ENABLED
     #chapAuth: ENABLED
   allowVolumeExpansion: true
   reclaimPolicy: Delete
   
   $ oc apply -f nutanix-volume.yaml
   ```

   

3. Check StorageClass

   ```
   $ oc get storageclass
   NAME            PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   nutanixvolume   csi.nutanix.com   Delete          Immediate           true                   121m
   ```

   ![image-20240509140930382](https://kenkenny.synology.me:5543/images/2024/05/image-20240509140930382.png)

4. Create test-pvc

   ![image-20240509140956018](https://kenkenny.synology.me:5543/images/2024/05/image-20240509140956018.png)

   ![image-20240509141008518](https://kenkenny.synology.me:5543/images/2024/05/image-20240509141008518.png)



### Cert-manager

1. Install cert-manager tools

   ```
   路徑 
   https://github.com/cert-manager/cert-manager/releases/tag/v1.14.5
   
   https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
   
   $ oc new-project cert-manager
   
   $ oc apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
   
   $ oc get pods -n cert-manager
   NAME                                      READY   STATUS    RESTARTS   AGE
   cert-manager-5658d944df-9tqqp             1/1     Running   0          34s
   cert-manager-cainjector-cb99ff845-24r6d   1/1     Running   0          34s
   cert-manager-webhook-7fd74b8dc7-ts6pk     1/1     Running   0          34s
   
   ```

### Onboard OCP to PC

建立Project 把 ocp相關的VM放進去 (ocp-lab)

用Support Portal 上下載的k8s-agent.tar 需要自行上傳到Dokcer Hub或是其他私倉，並且修改value.yaml 內的image路徑

1. get cluster uid and name 

   ```
   $ kubectl get ns kube-system -o json |grep uid
               "openshift.io/sa.scc.uid-range": "1000020000/10000"
           "uid": "25c8543a-3df0-442a-83eb-5af1ab1b7be3"
           
   $ kubectl get infrastructure cluster -o yaml | grep infrastructureName
     infrastructureName: ocplab-qx6t5
   $ oc get -o jsonpath='{.status.infrastructureName}{"\n"}' infrastructure cluster
   ocplab-qx6t5
     
   $ export CLUSTER_UUID=25c8543a-3df0-442a-83eb-5af1ab1b7be3
   $ export CLUSTER_INFRA_NAME=ocplab-qx6t5
   
   PC上的VG會多k8s-的prefix
   k8s-25c8543a-3df0-442a-83eb-5af1ab1b7be3
   ```

   

2. Download k8s-agent.tar and ndk-1.0.0.tar

   ```
   
   # 下載連結要重新產生
   $ curl -o k8s-agent-1.0.0.tar -O "https://download.nutanix.com/downloads/ndk/1.0.0/k8s-agent-1.0.0.tar?Expires=1715275315306&Key-Pair-Id=APKAJTTNCWPEI42QKMSA&Signature=exuDWlLvpFIhLfN6YpDd9SyAz1a8o0CudJabSf8j~FEAVpwd6CjFe2WgCae2Jp5gZjNwwIEvHq4lXbvRMqDWm~LEVxO9E84ribrxu2uDNbkbAuG7I0OeWEDROsdpM1-qN~g9fp16-X~Y8lG-sh1cOLz-N8hjgq83vDMeusfTUrRMQOjlrGV7h44QL0-LlIs8GkkoIJbh3rxhZO~vavCvs4EJ9uec9qH6BrQLMfstVZr7j6uLu4juf6hnEsWJzalJXZ04YaA8onpgm-jhNBj2m5fxjXlJ-iCsu6u6BSLmm2ffpzSkUUlFpoSJZX~QODiHHiQi9V9xe9R8wdkQPfo-1g__"
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
   100 59.2M  100 59.2M    0     0  10.4M      0  0:00:05  0:00:05 --:--:-- 13.2M
   
   $ curl -o ndk-1.0.0.tar -O "https://download.nutanix.com/downloads/ndk/1.0.0/ndk-1.0.0.tar?Expires=1715276310943&Key-Pair-Id=APKAJTTNCWPEI42QKMSA&Signature=A8y8JoJupVvWedB~jSHTysEZK18eyITvdT6DFw9C9JpVOhKc9jnZraaDir2DZDeRVpnPaOraMF2lR5lFqlSaikGj274ETgetuz33R0jcJ-fB44~xfXL~eK5cpqNhuuG3Yufs0biq1b8CC-frrVeupzKzOdmJyBEWXPe4GSCLWwqPubPjhUAHMKyoOgcxSd-A6kS9f~c0eEikartUrgtCjyZP-iZwPxn6gg3jGoZuHK3f2Nf~HjmyrJqwBmLQA0ibVdRTN0YkYVfplpkr5ucp8MLMPFbhdmAJZIAot8vc2-qMPoidczgU1Msd1w8NsXWMUFQBnO-Y73VZ31d4yxI31A__"
   
   $ mkdir ndk-k8s-agent
   $ cd ndk-k8s-agent
   
   $ ls k8s-agent-1.0.0.tar 
   k8s-agent-1.0.0.tar
   
   $ mv k8s-agent-1.0.0.tar ndk-k8s-agent
   
   $ tar xvf k8s-agent-1.0.0.tar 
   ./k8s-agent-1.0.0/
   ./k8s-agent-1.0.0/Chart.yaml
   ./k8s-agent-1.0.0/.helmignore
   ./k8s-agent-1.0.0/k8s-agent-1.0.0.tar
   ./k8s-agent-1.0.0/templates/
   ./k8s-agent-1.0.0/templates/hook-predelete.yaml
   ./k8s-agent-1.0.0/templates/hook-preinstall.yaml
   ./k8s-agent-1.0.0/templates/hook-postdelete.yaml
   ./k8s-agent-1.0.0/templates/hook-postupgrade.yaml
   ./k8s-agent-1.0.0/templates/hook-postinstall.yaml
   ./k8s-agent-1.0.0/templates/nutanix_agent_deployment.yaml
   ./k8s-agent-1.0.0/values.schema.json
   ./k8s-agent-1.0.0/README.md
   ./k8s-agent-1.0.0/values.yaml
   ./k8s-agent-1.0.0/Nutanix-core_k8s-agent-Notice.txt
   ```

   

3. push image to docker hub

   ```
   $ echo -n 'kenxxxxxxxx:xuxxxxxxxx' |base64
   dockerconfigVubnxxxxxxxxxxxxxxxx2
   
   
   {
       "auths": {
           "https://index.docker.io/v1/": {
               "auth": "a2Vua2Vubnxxxxxxxxxxxxxxxx2"
           }
       }
   }
   
   $ vim dockerconfig.json
   $ cat dockerconfig.json |base64
   dockerconfigVubnxxxxxxxxxxxxxxxx2
   
   $ podman load -i k8s-agent-1.0.0.tar 
   $ podman tag localhost/k8s-agent:1.0.0 kenkennyinfo/nke-k8s-agent:1.0.0
   $ podman push kenkennyinfo/nke-k8s-agent:1.0.0
   $ podman tag localhost/k8s-agent:1.0.0 kenkennyinfo/nke-k8s-agent:668
   $ podman push kenkennyinfo/nke-k8s-agent:668
   ```
   
   ![image-20240510154840948](https://kenkenny.synology.me:5543/images/2024/05/image-20240510154840948.png)
   
   
   
4. Install k8s-agent

   ```
   
   安裝失敗時可用刪除指令
   $ oc delete clusterroles.rbac.authorization.k8s.io "nke-k8s-agent-preinstall"
   $ oc delete ClusterRoleBinding nke-k8s-agent-preinstall
   $ oc delete project ntnx-system
   $ oc new-project ntnx-system
   
   !用最新版本的需要自行上傳!
   
   ## Docker Hub (版本較新 v668)
   ---
   $ vim values.yaml 
   
   agent:
     namespaceOverride: ntnx-system
     name: nutanix-agent
     port: 8080
     image:
       repository: docker.io/kenkennyinfo
       name: nke-k8s-agent
       pullPolicy: IfNotPresent
       tag: 668
       privateRegistry: true
       imageCredentials:
         dockerconfig: "dockerconfigVubnxxxxxxxxxxxxxxxx2"
     updateConfigInMin: 10
     updateMetricsInMin: 360
   
   pc:
     port: 9440
     insecure: false #set this to true if PC does not have https enabled
     endpoint: "" # eg: ip or fqdn
     username: "" # eg: admin or any other user with Kubernetes Infrastructure provision role
     password: ""
   k8sClusterName: ""
   k8sClusterUUID: ""
   k8sDistribution: "OCP" # eg: CAPX or NKE or OCP
   categoryMappings: "" # "one or more comma separated key=value" eg: "key1=value1" or "key1=value1,key2=value2"
   
   ---
   ## 官方quay.io (版本較舊 v594)
   
   $ vim values.yaml 
   
   agent:
     namespaceOverride: ntnx-system
     name: nutanix-agent
     port: 8080
     image:
       repository: quay.io/karbon
       name: k8s-agent
       pullPolicy: IfNotPresent
       tag: 594
       privateRegistry: false
     updateConfigInMin: 10
     updateMetricsInMin: 360
   
   pc:
     port: 9440
     insecure: false #set this to true if PC does not have https enabled
     endpoint: "" # eg: ip or fqdn
     username: "" # eg: admin or any other user with Kubernetes Infrastructure provision role
     password: ""
   k8sClusterName: ""
   k8sClusterUUID: ""
   k8sDistribution: "OCP" # eg: CAPX or NKE or OCP
   categoryMappings: "" # "one or more comma separated key=value" eg: "key1=value1" or "key1=value1,key2=value2"
   
   ---
   
   Helm 安裝
   
   $ helm install k8s-agent-1.0.0 /home/nutanix/k8s-agent-1.0.0  --set pc.endpoint=172.16.90.75,pc.username=ocpadmin,pc.password=password,pc.insecure=true,k8sClusterName=ocplab-qx6t5,k8sClusterUUID=25c8543a-3df0-442a-83eb-5af1ab1b7be3,k8sDistribution=OCP -n ntnx-system --create-namespace
   WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
   NAME: k8s-agent-594
   LAST DEPLOYED: Wed May 15 14:09:01 2024
   NAMESPACE: ntnx-system
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   
   
   ## 安裝新版本或是更改密碼可以使用helm upgrade (修改values.yaml)
   
   $ helm upgrade k8s-agent-594 /home/nutanix/k8s-agent-1.0.0 --set pc.endpoint=172.16.90.75,pc.username=ocpadmin,pc.password=password,pc.insecure=true,k8sClusterName=ocplab-qx6t5,k8sClusterUUID=25c8543a-3df0-442a-83eb-5af1ab1b7be3 -n ntnx-system
   WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
   Release "k8s-agent-594" has been upgraded. Happy Helming!
   NAME: k8s-agent-594
   LAST DEPLOYED: Wed May 15 14:13:56 2024
   NAMESPACE: ntnx-system
   STATUS: deployed
   REVISION: 2
   TEST SUITE: None
   
   $ helm upgrade k8s-agent-594 /home/nutanix/k8s-agent-1.0.0 --set pc.endpoint=172.16.90.75,pc.username=ocpadmin,pc.password=password,pc.insecure=true,k8sClusterName=ocplab-qx6t5,k8sClusterUUID=25c8543a-3df0-442a-83eb-5af1ab1b7be3 -n ntnx-system
   WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
   Release "k8s-agent-594" has been upgraded. Happy Helming!
   NAME: k8s-agent-594
   LAST DEPLOYED: Wed May 15 15:18:35 2024
   NAMESPACE: ntnx-system
   STATUS: deployed
   REVISION: 3
   TEST SUITE: None
   
   
   ## Check pods and deployments
   
   $ oc get all -n ntnx-system
   NAME                                 READY   STATUS    RESTARTS   AGE
   pod/nutanix-agent-75df9d879b-98smz   1/1     Running   0          80s
   
   NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/nutanix-agent   1/1     1            1           70m
   
   NAME                                       DESIRED   CURRENT   READY   AGE
   replicaset.apps/nutanix-agent-75df9d879b   1         1         1       80s
   replicaset.apps/nutanix-agent-7c4b644996   0         0         0       70m
   
   
   $ oc describe deployment.apps/nutanix-agent -n ntnx-system
   Name:                   nutanix-agent
   Namespace:              ntnx-system
   CreationTimestamp:      Wed, 15 May 2024 14:09:31 +0800
   Labels:                 app.kubernetes.io/managed-by=Helm
   Annotations:            deployment.kubernetes.io/revision: 2
                           meta.helm.sh/release-name: k8s-agent-594
                           meta.helm.sh/release-namespace: ntnx-system
   Selector:               app=nutanix-agent
   Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
   StrategyType:           RollingUpdate
   MinReadySeconds:        0
   RollingUpdateStrategy:  25% max unavailable, 25% max surge
   Pod Template:
     Labels:           app=nutanix-agent
     Service Account:  nutanix-agent
     Containers:
      nutanix-agent:
       Image:      docker.io/kenkennyinfo/nke-k8s-agent:668
       Port:       8080/TCP
       Host Port:  0/TCP
       Command:
         /k8s-agent/k8s-agent
       Args:
         -pc-endpoint=172.16.90.75
         -pc-port=9440
         -insecure=true
         -k8s-cluster-name=ocplab-qx6t5
         -k8s-cluster-uuid=25c8543a-3df0-442a-83eb-5af1ab1b7be3
         -k8s-distribution=OCP
         -k8s-cluster-category=
         -update-config-in-min=10
         -update-metrics-in-min=360
       Limits:
         cpu:     500m
         memory:  128Mi
       Requests:
         cpu:        250m
         memory:     64Mi
       Liveness:     http-get http://:8080/health delay=5s timeout=5s period=15s #success=1 #failure=3
       Readiness:    http-get http://:8080/readiness delay=5s timeout=1s period=10s #success=1 #failure=3
       Environment:  <none>
       Mounts:
         /certs from pc-creds-volume (ro)
     Volumes:
      pc-creds-volume:
       Type:        Secret (a volume populated by a Secret)
       SecretName:  nutanix-agent
       Optional:    false
   Conditions:
     Type           Status  Reason
     ----           ------  ------
     Available      True    MinimumReplicasAvailable
     Progressing    True    NewReplicaSetAvailable
   OldReplicaSets:  <none>
   NewReplicaSet:   nutanix-agent-75df9d879b (1/1 replicas created)
   Events:
     Type    Reason             Age   From                   Message
     ----    ------             ----  ----                   -------
     Normal  ScalingReplicaSet  70m   deployment-controller  Scaled up replica set nutanix-agent-7c4b644996 to 1
     Normal  ScalingReplicaSet  84s   deployment-controller  Scaled up replica set nutanix-agent-75df9d879b to 1
     Normal  ScalingReplicaSet  73s   deployment-controller  Scaled down replica set nutanix-agent-7c4b644996 to 0 from 1
   
   ```

   安裝後進入Prism Central 確認已 Onboard 完成

   ![image-20240515150336316](https://kenkenny.synology.me:5543/images/2024/05/image-20240515150336316.png)

   ![image-20240515150407687](https://kenkenny.synology.me:5543/images/2024/05/image-20240515150407687.png)

   ![](https://kenkenny.synology.me:5543/images/2024/05/image-20240515142010187.png)

   ![image-20240515142031619](https://kenkenny.synology.me:5543/images/2024/05/image-20240515142031619.png)

   ![image-20240515142042758](https://kenkenny.synology.me:5543/images/2024/05/image-20240515142042758.png)

   

### NDK Install without TLS



1. 下載 ndk-1.0.0.tar 並解壓縮

   ```
   $ wget https://download.nutanix.com/downloads/ndk/1.0.0/ndk-1.0.0.tar?Expires=1715793783203&Key-Pair-Id=APKAJTTNCWPEI42QKMSA&Signature=bT~JG3wJjEQ3dvNO85khPiJq1yVxAH6YPV7PGtJvKaf4iJVRDfQIockkqSyH3ZubXpSvF2loZAXqvueSef41SZa9vSzEC1zrc~IzwcCjAcXnAbXC1LGy566yIBvwNaFm1qSr0jONb3zwzckNHidS8QmTRRyscaiHdF8xQ~gD0MQfKBlrl4hej3Pnae0ckMZoT5yCPzAv8vQU5XhvL0YECmy2T6LhBH13mxHpijM70aACCjdE5KdlCHFrSZhsDYYy5FWRYT02q70mM4SAUNYYcBnW4FNmmKuPWPyq0r2lcJRktIOgfQOHvUv-qw0vyrEdRj7QmgWpiCAAoSfqWl-lKA__
   
   $ tar -xvf ndk-1.0.0.tar 
   
   $ tree ndk-1.0.0
   ndk-1.0.0
   ├── chart
   │   ├── Chart.yaml
   │   ├── crds
   │   │   ├── dataservices.nutanix.com_applicationsnapshotcontents.yaml
   │   │   ├── dataservices.nutanix.com_applicationsnapshotreplications.yaml
   │   │   ├── dataservices.nutanix.com_applicationsnapshotrestores.yaml
   │   │   ├── dataservices.nutanix.com_applicationsnapshots.yaml
   │   │   ├── dataservices.nutanix.com_applications.yaml
   │   │   ├── dataservices.nutanix.com_appprotectionplans.yaml
   │   │   ├── dataservices.nutanix.com_protectionplans.yaml
   │   │   ├── dataservices.nutanix.com_remotes.yaml
   │   │   ├── dataservices.nutanix.com_replicationtargets.yaml
   │   │   ├── dataservices.nutanix.com_storageclusters.yaml
   │   │   └── scheduler.nutanix.com_jobschedulers.yaml
   │   ├── Nutanix-core-k8s-job-scheduler-Disclosure.txt
   │   ├── Nutanix-core-k8s-juno-aos-pc-client-Disclosure.txt
   │   ├── Nutanix-core-k8s-juno-Disclosure.txt
   │   ├── Nutanix-kube-rbac-proxy-Disclosure.txt
   │   ├── templates
   │   │   ├── cert-manager.yaml
   │   │   ├── controller-manager-metrics-monitor.yaml
   │   │   ├── csi-precheck.yaml
   │   │   ├── deployment.yaml
   │   │   ├── _helpers.tpl
   │   │   ├── intercom-service.yaml
   │   │   ├── leader-election-rbac.yaml
   │   │   ├── manager-config.yaml
   │   │   ├── manager-rbac.yaml
   │   │   ├── metrics-reader-rbac.yaml
   │   │   ├── metrics-service.yaml
   │   │   ├── ndk-logs-pvc.yaml
   │   │   ├── ntnx-log-sc.yaml
   │   │   ├── proxy-rbac.yaml
   │   │   ├── registry-secret.yaml
   │   │   ├── scheduler-webhook-service.yaml
   │   │   ├── webhook-certificate.yaml
   │   │   └── webhook-service.yaml
   │   └── values.yaml
   └── ndk-1.0.0.tar
   
   3 directories, 36 files
   ```

   

2. 儲存Container image 並上傳至Docker hub or 私倉

   ```
   # Load Images
   
   $ podman load -i ndk-1.0.0/ndk-1.0.0.tar
   
   Writing manifest to image destination
   Storing signatures
   Loaded image: localhost/ndk/infra-manager:1.0.0
   Loaded image: localhost/ndk/job-scheduler:1.0.0
   Loaded image: localhost/ndk/kube-rbac-proxy:v0.8.0
   Loaded image: localhost/ndk/manager:1.0.0
   
   # Tag Images
   podman tag <ndk-image-name>:tag <image-registry-url>/ndk/<conatiner-name>:tag
   $ podman tag ndk/manager:1.0.0 kenkennyinfo/ndk-manager:1.0.0
   $ podman tag ndk/job-scheduler:1.0.0 kenkennyinfo/ndk-job-scheduler:1.0.0
   $ podman tag ndk/kube-rbac-proxy:v0.8.0 kenkennyinfo/ndk-kube-rbac-proxy:v0.8.0
   $ podman tag ndk/infra-manager:1.0.0 kenkennyinfo/ndk-infra-manager:1.0.0
   
   # batch tag images
   $ for img in ndk/manager:1.0.0 ndk/infra-manager:1.0.0 ndk/job-scheduler:1.0.0 ndk/kube-rbac-proxy:v0.8.0; do docker tag $img kenkennyinfo/${img}; done
   
   Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
   Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
   Error: ndk/infra-manager-client:1.0.0: image not known
   Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
   Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
   
   
   $ podman images
   [nutanix@ken-rhel9 ndk-1.0.0]$ podman images
   
   REPOSITORY                                                     TAG         IMAGE ID      CREATED       SIZE
   
   localhost/kenkennyinfo/ndk-manager                             1.0.0       ebffd7549ea5  2 months ago  69.2 MB
   localhost/ndk/manager                                          1.0.0       ebffd7549ea5  2 months ago  69.2 MB
   
   localhost/kenkennyinfo/ndk-infra-manager                       1.0.0       2982ed521b0e  2 months ago  60 MB
   localhost/ndk/infra-manager                                    1.0.0       2982ed521b0e  2 months ago  60 MB
   
   localhost/kenkennyinfo/nke-k8s-agent                           1.0.0       9bda464636b6  2 months ago  61.7 MB
   localhost/kenkennyinfo/nke-k8s-agent                           668         9bda464636b6  2 months ago  61.7 MB
   localhost/k8s-agent                                            1.0.0       9bda464636b6  2 months ago  61.7 MB
   docker.io/kenkennyinfo/nke-k8s-agent                           1.0.0       9bda464636b6  2 months ago  61.7 MB
   quay.io/karbon/k8s-agent                                       594         171ea3376596  3 months ago  58.4 MB
   
   localhost/ndk/job-scheduler                                    1.0.0       d594a3e567b7  3 months ago  59.1 MB
   localhost/kenkennyinfo/ndk-job-scheduler                       1.0.0       d594a3e567b7  3 months ago  59.1 MB
   
   
   localhost/kenkennyinfo/ndk-kube-rbac-proxy                     v0.8.0      ad393d6a4d1b  3 years ago   50.2 MB
   localhost/ndk/kube-rbac-proxy                                  v0.8.0      ad393d6a4d1b  3 years ago   50.2 MB
   
   # Push Images
   $ podman push kenkennyinfo/ndk-kube-rbac-proxy:v0.8.0
   $ podman push kenkennyinfo/ndk-job-scheduler:1.0.0
   $ podman push kenkennyinfo/ndk-infra-manager:1.0.0
   $ podman push kenkennyinfo/ndk-manager:1.0.0
   ```

   

3. 建立Secret，修改 chart/values.yaml

   ```
   # 建立Docker hub secret
   
   $ vim nutanix-k8s-agent-pull-secret.yaml 
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: nutanix-k8s-agent-pull-secret
   type: kubernetes.io/dockerconfigjson
   data:
     .dockerconfigjson: "dockerconfigVubnxxxxxxxxxxxxxxxx2"
   
   ---
   $ vim ntnx-pc-secret2.yaml
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: ntnx-pc-secret
     namespace: ntnx-system
   stringData:
     # prism-pc-ip:prism-port:admin:password
     key: 172.16.90.75:9440:ocpadmin:password
   ---
   
   $ oc apply -f nutanix-k8s-agent-pull-secret.yaml -n ntnx-system
   $ oc apply -f ntnx-pc-secret2.yaml -n ntnx-system
   
   $ oc get secret -n ntnx-system
   NAME                                  TYPE                                  DATA   AGE
   nutanix-k8s-agent-pull-secret         kubernetes.io/dockerconfigjson        1      12s
   
   ---
   $ vim /home/nutanix/ndk-1.0.0/chart/values.yaml
   
   manager:
     # -- Image Repository
     repository: kenkennyinfo/ndk-manager
     # -- Image tag
     # @default -- .Chart.AppVersion
     tag: 1.0.0
     # -- Image pull policy
     pullPolicy: Always
   
   infraManager:
     # -- Image Repository
     repository: kenkennyinfo/ndk-infra-manager
     # -- Image tag
     # @default -- .Chart.AppVersion
     tag: 1.0.0
     # -- Image pull policy
     pullPolicy: Always
   
   kubeRbacProxy:
     # -- Image Repository
     repository: kenkennyinfo/ndk-kube-rbac-proxy
     # -- Image tag
     # @default -- .Chart.AppVersion
     tag: v0.8.0
   
   jobScheduler:
     # -- Image Repository
     repository: kenkennyinfo/ndk-job-scheduler
     # -- Image tag
     # @default -- .Chart.AppVersion
     tag: 1.0.0
     # -- Image pull policy
     pullPolicy: Always
   
   imageCredentials:
     # Name of the secret containing the credentials to pull image from the container registry.
     imagePullSecretName: nutanix-k8s-agent-pull-secret
     
    clusterName: ocplab.nutanixlab.local
    
    intercomService:
     ports:
     - port: 2021
       protocol: TCP
       targetPort: 2021
     type: NodePort
     
     
    
    ---end
   ```

   

4. 安裝NDK

   ```
    helm install ndk -n ntnx-system <chart-location> --set tls.server.enable=false 
    
   $ helm install ndk -n ntnx-system /home/nutanix/ndk-1.0.0/chart/ --set tls.server.enable=false
    
   WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
   NAME: ndk
   LAST DEPLOYED: Wed May 15 17:14:37 2024
   NAMESPACE: ntnx-system
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   
   $ oc get pods -n ntnx-system
   NAME                                      READY   STATUS    RESTARTS   AGE
   ndk-controller-manager-55bf75f64b-blnfc   4/4     Running   0          3m57s
   nutanix-agent-75df9d879b-98smz            1/1     Running   0          120m
   
   $ oc adm policy add-scc-to-user anyuid -z ndk-controller-manager
   clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "ndk-controller-manager"
   
   $ helm upgrade ndk -n ntnx-system /home/nutanix/ndk-1.0.0/chart/ --set tls.server.enable=false
   WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/nutanix/.kube/config
   Release "ndk" has been upgraded. Happy Helming!
   NAME: ndk
   LAST DEPLOYED: Mon May 20 17:57:43 2024
   NAMESPACE: ntnx-system
   STATUS: deployed
   REVISION: 3
   TEST SUITE: None
   ```

   

5. 安裝完成

   ```
   $ oc get all -n ntnx-system
   
   NAME                                          READY   STATUS    RESTARTS   AGE
   pod/ndk-controller-manager-55bf75f64b-blnfc   4/4     Running   0          17h
   pod/nutanix-agent-75df9d879b-98smz            1/1     Running   0          19h
   
   NAME                                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
   service/ndk-controller-manager-metrics-service   ClusterIP   172.30.186.204   <none>        8443/TCP         17h
   service/ndk-intercom-service                     NodePort    172.30.237.126   <none>        2021:30998/TCP   17h
   service/ndk-scheduler-webhook-service            ClusterIP   172.30.64.110    <none>        9444/TCP         17h
   service/ndk-webhook-service                      ClusterIP   172.30.66.56     <none>        443/TCP          17h
   
   NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/ndk-controller-manager   1/1     1            1           17h
   deployment.apps/nutanix-agent            1/1     1            1           20h
   
   NAME                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/ndk-controller-manager-55bf75f64b   1         1         1       17h
   replicaset.apps/nutanix-agent-75df9d879b            1         1         1       19h
   replicaset.apps/nutanix-agent-7c4b644996            0         0         0       20h
   ```

   ![image-20240516110205960](https://kenkenny.synology.me:5543/images/2024/05/image-20240516110205960.png)

6. 測試NDK 連線 grpcurl

   ```
   $ podman pull fullstorydev/grpcurl:latest
   
   $ podman run  fullstorydev/grpcurl  -plaintext 172.16.90.180:30998 list
   grpc.reflection.v1.ServerReflection
   grpc.reflection.v1alpha.ServerReflection
   juno_interface.Juno
   ```

   

### Storage Cluster

1. 取得PC、PE的Cluster UUID

   ```
   on Prism Central CVM
   
   $ ncli cluster info
   
       Cluster Id                : 8c52e6be-697b-42a4-bacd-8e6ada7994f2::4237199414008583410
       Cluster Uuid              : 8c52e6be-697b-42a4-bacd-8e6ada7994f2
       Cluster Name              : NTNXSELABPC
       Cluster Version           : pc.2023.4
       Cluster Full Version      : el7.3-release-fraser-2023.4-stable-e10760423ef518a91beb02b784c7ff2e0cf3a45c
       Cluster FQDN              : NTNXSELABPC.nutanixlab.local
       Is LTS                    : false
       External Data Services... : 
       Support Verbosity Level   : BASIC_COREDUMP
       Lock Down Status          : Disabled
       Password Remote Login ... : Enabled
       Timezone                  : America/Los_Angeles
       NCC Version               : ncc-4.6.6.1
       Degraded Node Monitoring  : Enabled
   
   on Prism Element CVM
   
   $ ncli cluster info
   
       Cluster Id                : 00060f36-1379-a8ba-0000-000000028e95::167573
       Cluster Uuid              : 00060f36-1379-a8ba-0000-000000028e95
       Cluster Name              : NX1365G6PE
       Cluster Version           : 6.7.1
       Cluster Full Version      : el7.3-release-fraser-6.7.1-stable-104e6662e8b936a9225379d2d68efd93b5448289
       External IP address       : 172.16.90.74
       Node Count                : 3
       Block Count               : 1
       Shadow Clones Status      : Enabled
       Has Self Encrypting Disk  : no
       Cluster Masquerading I... : 
       Cluster Masquerading PORT : 
       Is registered to PC       : true
       Rf1 Container Support     : Enabled
       Rebuild Reservation       : Disabled
       Encryption In Transit     : Disabled
       Is LTS                    : false
       External Data Services... : 172.16.90.76
       Support Verbosity Level   : BASIC_COREDUMP
       Lock Down Status          : Disabled
       Password Remote Login ... : Enabled
       Timezone                  : Asia/Taipei
       NCC Version               : ncc-4.6.6.1
       Degraded Node Monitoring  : Enabled
   
   ```

   

2. 建立StorageCluster.yaml

   ```
   $ vim storagecluster-1365g6.yaml
   
   apiVersion: dataservices.nutanix.com/v1alpha1
   kind: StorageCluster
   metadata:
    name: nx1365g6pe
   spec:
    storageServerUuid: 00060f36-1379-a8ba-0000-000000028e95
    managementServerUuid: 8c52e6be-697b-42a4-bacd-8e6ada7994f2
    
   $ oc apply -f storagecluster-1365g6.yaml
   
   $ oc get storagecluster
   NAME                    AVAILABLE
   storagecluster-1365g6   true
   
   $ oc get storagecluster
   NAME         AVAILABLE
   nx1365g6pe   false
   ```




### Create Sample App

#### MySQL

1. Mysql 測試，建立mysql.yaml

   ```
   $ vim mysql-deploy.yaml
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysql-password
   type: opaque
   stringData:
     MYSQL_ROOT_PASSWORD: password
   ---
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: mysql-set
   spec:
     selector:
       matchLabels:
         app: mysql
     serviceName: "mysql"
     replicas: 3
     template:
       metadata:
         labels:
           app: mysql
       spec:
         terminationGracePeriodSeconds: 10
         containers:
         - name: mysql
           image: mysql:8.0
           ports:
           - containerPort: 3306
           volumeMounts:
           - name: mysql-store
             mountPath: /var/lib/mysql
           - name: mysql-data-1
             mountPath: /usr/data1
           env:
             - name: MYSQL_ROOT_PASSWORD
               valueFrom:
                 secretKeyRef:
                   name: mysql-password
                   key: MYSQL_ROOT_PASSWORD
     volumeClaimTemplates:
     - metadata:
         name: mysql-store
       spec:
         accessModes: ["ReadWriteOnce"]
         storageClassName: nutanixvolume
         resources:
           requests:
             storage: 5Gi
     - metadata:
         name: mysql-data-1
       spec:
         accessModes: ["ReadWriteOnce"]
         storageClassName: nutanixvolume
         resources:
           requests:
             storage: 3Gi
   ```

   

2. 建立Mysql app

   ```
   $ oc new-project ndk-mysql
   
   $ oc apply -f mysql-deploy.yaml -n ndk-mysql
   secret/mysql-password created
   statefulset.apps/mysql-set created
   
   
   $ oc get pods
   NAME          READY   STATUS              RESTARTS   AGE
   mysql-set-0   0/1     ContainerCreating   0          42s
   
   $ oc get pods
   NAME          READY   STATUS    RESTARTS   AGE
   mysql-set-0   1/1     Running   0          3m18s
   mysql-set-1   1/1     Running   0          2m22s
   mysql-set-2   1/1     Running   0          72s
   
   $ oc get pvc,pv
   NAME                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
   persistentvolumeclaim/mysql-data-1-mysql-set-0   Bound    pvc-276ea905-a815-4635-98c0-3a4fade1bc44   3Gi        RWO            nutanixvolume   6m24s
   persistentvolumeclaim/mysql-data-1-mysql-set-1   Bound    pvc-5638dc79-8271-4a49-94b9-d22c6212e232   3Gi        RWO            nutanixvolume   5m28s
   persistentvolumeclaim/mysql-data-1-mysql-set-2   Bound    pvc-a4e832f4-e99b-4bdc-bc5b-253731447ce4   3Gi        RWO            nutanixvolume   4m18s
   persistentvolumeclaim/mysql-store-mysql-set-0    Bound    pvc-262a52dc-1861-4992-b2aa-e41f448a6db5   5Gi        RWO            nutanixvolume   6m24s
   persistentvolumeclaim/mysql-store-mysql-set-1    Bound    pvc-1e809e1d-2190-4220-80ba-35712900d3a6   5Gi        RWO            nutanixvolume   5m28s
   persistentvolumeclaim/mysql-store-mysql-set-2    Bound    pvc-fafd94af-11be-4b02-aef1-4d959607c0cf   5Gi        RWO            nutanixvolume   4m18s
   
   NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS    REASON   AGE
   persistentvolume/pvc-127de2ed-452d-4fc2-a08e-f320cc4c85e0   1Gi        RWO            Delete           Bound    default/my-pvc                       nutanixvolume            6d23h
   persistentvolume/pvc-1e809e1d-2190-4220-80ba-35712900d3a6   5Gi        RWO            Delete           Bound    ndk-mysql/mysql-store-mysql-set-1    nutanixvolume            5m22s
   persistentvolume/pvc-262a52dc-1861-4992-b2aa-e41f448a6db5   5Gi        RWO            Delete           Bound    ndk-mysql/mysql-store-mysql-set-0    nutanixvolume            6m15s
   persistentvolume/pvc-276ea905-a815-4635-98c0-3a4fade1bc44   3Gi        RWO            Delete           Bound    ndk-mysql/mysql-data-1-mysql-set-0   nutanixvolume            6m14s
   persistentvolume/pvc-5638dc79-8271-4a49-94b9-d22c6212e232   3Gi        RWO            Delete           Bound    ndk-mysql/mysql-data-1-mysql-set-1   nutanixvolume            5m21s
   persistentvolume/pvc-8468392e-a564-4d05-81f1-175820eceba2   1Gi        RWO            Delete           Bound    default/test-pvc                     nutanixvolume            20h
   persistentvolume/pvc-a4e832f4-e99b-4bdc-bc5b-253731447ce4   3Gi        RWO            Delete           Bound    ndk-mysql/mysql-data-1-mysql-set-2   nutanixvolume            4m9s
   persistentvolume/pvc-fafd94af-11be-4b02-aef1-4d959607c0cf   5Gi        RWO            Delete           Bound    ndk-mysql/mysql-store-mysql-set-2    nutanixvolume            4m9s
   ```


   ![image-20240521094917993](https://kenkenny.synology.me:5543/images/2024/05/image-20240521094917993.png)
   ![image-20240521094932643](https://kenkenny.synology.me:5543/images/2024/05/image-20240521094932643.png)

3. 建立Application CR

   ```
   $ vim mysql-cr.yaml
   
   apiVersion: dataservices.nutanix.com/v1alpha1
   kind: Application
   metadata:
     name: mysql-cr
     namespace: ndk-mysql
   spec:
     applicationSelector:
   
               
   $ oc apply -f mysql-cr.yaml 
   
   $ oc get application
   
   NAME       AGE   LAST-STATUS-UPDATE
   mysql-cr   31s   31s
   ```
   



### NDK Snapshot

#### Manual Local Snapshot

1. Create Application Snapshot CR

   ```
   $ vim mysql-snap.yaml
   
   apiVersion: dataservices.nutanix.com/v1alpha1
   kind: ApplicationSnapshot
   metadata:
     name: mysql-snap-1
     namespace: ndk-mysql
   spec:
     source:
       applicationRef:
         name: mysql-cr
   ```

   

2. Apply Snapshot

   ```
   $ oc apply -f mysql-snap.yaml 
   applicationsnapshot.dataservices.nutanix.com/mysql-snap-1 created
   
   $ oc get applicationsnapshot -n ndk-mysql
   NAME           AGE   READY-TO-USE   BOUND-SNAPSHOTCONTENT                      SNAPSHOT-AGE
   mysql-snap-1   8s    false          asc-8889156a-43f2-4d95-9075-11ccd52c9457   
   $ oc get applicationsnapshot -n ndk-mysql
   NAME           AGE   READY-TO-USE   BOUND-SNAPSHOTCONTENT                      SNAPSHOT-AGE
   mysql-snap-1   58s   true           asc-8889156a-43f2-4d95-9075-11ccd52c9457   30s
   
   
   $ oc describe applicationsnapshot -n ndk-mysql
   
   Name:         mysql-snap-1
   Namespace:    ndk-mysql
   Labels:       <none>
   Annotations:  <none>
   API Version:  dataservices.nutanix.com/v1alpha1
   Kind:         ApplicationSnapshot
   Metadata:
     Creation Timestamp:  2024-05-22T10:10:24Z
     Finalizers:
       dataservices.nutanix.com/app-snap
     Generation:  1
     Managed Fields:
       API Version:  dataservices.nutanix.com/v1alpha1
       Fields Type:  FieldsV1
       fieldsV1:
         f:metadata:
           f:annotations:
             .:
             f:kubectl.kubernetes.io/last-applied-configuration:
         f:spec:
           .:
           f:source:
             .:
             f:applicationRef:
               .:
               f:name:
       Manager:      kubectl-client-side-apply
       Operation:    Update
       Time:         2024-05-22T10:10:24Z
       API Version:  dataservices.nutanix.com/v1alpha1
       Fields Type:  FieldsV1
       fieldsV1:
         f:metadata:
           f:finalizers:
             .:
             v:"dataservices.nutanix.com/app-snap":
       Manager:      manager
       Operation:    Update
       Time:         2024-05-22T10:10:24Z
       API Version:  dataservices.nutanix.com/v1alpha1
       Fields Type:  FieldsV1
       fieldsV1:
         f:status:
           .:
           f:boundApplicationSnapshotContentName:
           f:creationTime:
           f:readyToUse:
           f:summary:
             .:
             f:snapshotArtifacts:
               .:
               f:apps/v1/StatefulSet:
               f:authorization.openshift.io/v1/RoleBinding:
               f:rbac.authorization.k8s.io/v1/RoleBinding:
               f:v1/ConfigMap:
               f:v1/PersistentVolumeClaim:
               f:v1/Secret:
               f:v1/ServiceAccount:
       Manager:         manager
       Operation:       Update
       Subresource:     status
       Time:            2024-05-22T10:10:52Z
     Resource Version:  48052327
     UID:               8889156a-43f2-4d95-9075-11ccd52c9457
   Spec:
     Source:
       Application Ref:
         Name:  mysql-cr
   Status:
     Bound Application Snapshot Content Name:  asc-8889156a-43f2-4d95-9075-11ccd52c9457
     Creation Time:                            2024-05-22T10:10:52Z
     Ready To Use:                             true
     Summary:
       Snapshot Artifacts:
         apps/v1/StatefulSet:
           Name:  mysql-set
         authorization.openshift.io/v1/RoleBinding:
           Name:  admin
           Name:  system:image-builders
           Name:  system:deployers
           Name:  system:image-pullers
         rbac.authorization.k8s.io/v1/RoleBinding:
           Name:  system:image-builders
           Name:  admin
           Name:  system:image-pullers
           Name:  system:deployers
         v1/ConfigMap:
           Name:  openshift-service-ca.crt
           Name:  kube-root-ca.crt
         v1/PersistentVolumeClaim:
           Name:  mysql-store-mysql-set-1
           Name:  mysql-store-mysql-set-2
           Name:  mysql-data-1-mysql-set-0
           Name:  mysql-data-1-mysql-set-1
           Name:  mysql-data-1-mysql-set-2
           Name:  mysql-store-mysql-set-0
         v1/Secret:
           Name:  default-token-rpqq4
           Name:  mysql-password
           Name:  deployer-token-cc92m
           Name:  builder-token-xk5g4
         v1/ServiceAccount:
           Name:  deployer
           Name:  default
           Name:  builder
   Events:        <none>
   ```

   OCP 上的 Volume Snapshot

   ![image-20240523091131334](https://kenkenny.synology.me:5543/images/2024/05/image-20240523091131334.png)

   OCP Volume SnapshotContents

   ![image-20240523091201873](https://kenkenny.synology.me:5543/images/2024/05/image-20240523091201873.png)

   Nutanix Prism Central - VG Recovery Point

   ![image-20240523091508568](https://kenkenny.synology.me:5543/images/2024/05/image-20240523091508568.png)

### NDK Restore

#### Restore  Local Snapshot

1. Delete mysql.yaml

   ```
   $ ll
   total 12
   -rw-r--r--. 1 nutanix nutanix  393 May 16 13:45 mysql-cr.yaml
   -rw-r--r--. 1 nutanix nutanix 1213 May 16 11:18 mysql-deploy.yaml
   -rw-r--r--. 1 nutanix nutanix  183 May 20 15:45 mysql-snap.yaml
   
   $ oc get pvc -n ndk-mysql
   NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
   mysql-data-1-mysql-set-0   Bound    pvc-276ea905-a815-4635-98c0-3a4fade1bc44   3Gi        RWO            nutanixvolume   4d4h
   mysql-data-1-mysql-set-1   Bound    pvc-5638dc79-8271-4a49-94b9-d22c6212e232   3Gi        RWO            nutanixvolume   4d4h
   mysql-data-1-mysql-set-2   Bound    pvc-a4e832f4-e99b-4bdc-bc5b-253731447ce4   3Gi        RWO            nutanixvolume   4d4h
   mysql-store-mysql-set-0    Bound    pvc-262a52dc-1861-4992-b2aa-e41f448a6db5   5Gi        RWO            nutanixvolume   4d4h
   mysql-store-mysql-set-1    Bound    pvc-1e809e1d-2190-4220-80ba-35712900d3a6   5Gi        RWO            nutanixvolume   4d4h
   mysql-store-mysql-set-2    Bound    pvc-fafd94af-11be-4b02-aef1-4d959607c0cf   5Gi        RWO            nutanixvolume   4d4h
   
   $ oc get all
   NAME              READY   STATUS    RESTARTS   AGE
   pod/mysql-set-0   1/1     Running   0          4d4h
   pod/mysql-set-1   1/1     Running   0          4d4h
   pod/mysql-set-2   1/1     Running   0          4d4h
   
   NAME                         READY   AGE
   statefulset.apps/mysql-set   3/3     4d4h
   
   $ oc delete app mysql-cr -n ndk-mysql
   application.dataservices.nutanix.com "mysql-cr" deleted
   
   $ oc delete -f mysql-deploy.yaml -n ndk-mysql
   secret "mysql-password" deleted
   statefulset.apps "mysql-set" deleted
   
   $ oc delete pvc --all -n ndk-mysql
   persistentvolumeclaim "mysql-data-1-mysql-set-0" deleted
   persistentvolumeclaim "mysql-data-1-mysql-set-1" deleted
   persistentvolumeclaim "mysql-data-1-mysql-set-2" deleted
   persistentvolumeclaim "mysql-store-mysql-set-0" deleted
   persistentvolumeclaim "mysql-store-mysql-set-1" deleted
   persistentvolumeclaim "mysql-store-mysql-set-2" deleted
   
   $ oc get pvc -n ndk-mysql
   No resources found in ndk-mysql namespace.
   
   $ oc get all -n ndk-mysql
   No resources found in ndk-mysql namespace.
   ```

   <img src="https://kenkenny.synology.me:5543/images/2024/05/image-20240520160111632.png" alt="image-20240520160111632" style="zoom:50%;" />

   

   

2. Create Application Restore  CR

   ```
   $ vim mysql-restore.yaml
   
   apiVersion: dataservices.nutanix.com/v1alpha1
   kind: ApplicationSnapshotRestore
   metadata:
     name: restore-mysql-snap-1
     namespace: ndk-mysql
   spec:
     applicationSnapshotName: mysql-snap-1
   ```

   

3. oc apply restore CR

   ```
   $ oc apply -f mysql-restore.yaml 
   applicationsnapshotrestore.dataservices.nutanix.com/restore-mysql-snap-1 created
   
   $ oc get applicationsnapshotrestore -n ndk-mysql -o yaml -o wide
   NAME                   SNAPSHOT-NAME   COMPLETED
   restore-mysql-snap-1   mysql-snap-1    true
   
   $ oc describe applicationsnapshotrestore restore-mysql-snap-1 -n ndk-mysql
   Name:         restore-mysql-snap-1
   Namespace:    ndk-mysql
   Labels:       <none>
   Annotations:  <none>
   API Version:  dataservices.nutanix.com/v1alpha1
   Kind:         ApplicationSnapshotRestore
   Metadata:
     Creation Timestamp:  2024-05-22T10:14:26Z
     Finalizers:
       dataservices.nutanix.com/application-restore
     Generation:  1
     Managed Fields:
       API Version:  dataservices.nutanix.com/v1alpha1
       Fields Type:  FieldsV1
       fieldsV1:
         f:metadata:
           f:annotations:
             .:
             f:kubectl.kubernetes.io/last-applied-configuration:
         f:spec:
           .:
           f:applicationSnapshotName:
       Manager:      kubectl-client-side-apply
       Operation:    Update
       Time:         2024-05-22T10:14:26Z
       API Version:  dataservices.nutanix.com/v1alpha1
       Fields Type:  FieldsV1
       fieldsV1:
         f:metadata:
           f:finalizers:
             .:
             v:"dataservices.nutanix.com/application-restore":
       Manager:      manager
       Operation:    Update
       Time:         2024-05-22T10:14:26Z
       API Version:  dataservices.nutanix.com/v1alpha1
       Fields Type:  FieldsV1
       fieldsV1:
         f:status:
           .:
           f:completed:
           f:conditions:
       Manager:         manager
       Operation:       Update
       Subresource:     status
       Time:            2024-05-22T10:14:43Z
     Resource Version:  48054533
     UID:               7a10fa58-3aaa-4592-8bdf-f6db2e9c4a0d
   Spec:
     Application Snapshot Name:  mysql-snap-1
   Status:
     Completed:  true
     Conditions:
       Last Transition Time:  2024-05-22T10:14:26Z
       Message:               
       Observed Generation:   1
       Reason:                RequestCompleted
       Status:                False
       Type:                  Progressing
       Last Transition Time:  2024-05-22T10:14:27Z
       Message:               All prechecks passed and finalizers on dependent resources set. Skipped restoring resources since they already exist: ["/v1, Kind=ConfigMap, ndk-mysql/openshift-service-ca.crt","/v1, Kind=ServiceAccount, ndk-mysql/default","/v1, Kind=Secret, ndk-mysql/builder-token-xk5g4","/v1, Kind=ServiceAccount, ndk-mysql/deployer","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/admin","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/admin","/v1, Kind=Secret, ndk-mysql/default-token-rpqq4","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/system:image-pullers","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/system:image-pullers","/v1, Kind=Secret, ndk-mysql/deployer-token-cc92m","/v1, Kind=ServiceAccount, ndk-mysql/builder","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/system:image-builders","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/system:image-builders","/v1, Kind=ConfigMap, ndk-mysql/kube-root-ca.crt","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/system:deployers","rbac.authorization.k8s.io/v1, Kind=RoleBinding, ndk-mysql/system:deployers"]
       Observed Generation:   1
       Reason:                PrechecksPassed
       Status:                True
       Type:                  PrechecksPassed
       Last Transition Time:  2024-05-22T10:14:27Z
       Message:               All eligible volumes restored
       Observed Generation:   1
       Reason:                VolumesRestored
       Status:                True
       Type:                  VolumesRestored
       Last Transition Time:  2024-05-22T10:14:28Z
       Message:               All eligible application configs restored
       Observed Generation:   1
       Reason:                ApplicationConfigRestored
       Status:                True
       Type:                  ApplicationConfigRestored
       Last Transition Time:  2024-05-22T10:14:43Z
       Message:               Application restore successfully finalised
       Observed Generation:   1
       Reason:                ApplicationRestoreFinalised
       Status:                True
       Type:                  ApplicationRestoreFinalised
   Events:
     Type    Reason                 Age                From  Message
     ----    ------                 ----               ----  -------
     Normal  WaitingForVolumeBound  55s (x2 over 55s)  NDK   Waiting for PVCs to get Bound. Unbound PVCs: [mysql-store-mysql-set-2 mysql-data-1-mysql-set-0 mysql-store-mysql-set-1 mysql-data-1-mysql-set-2 mysql-data-1-mysql-set-1 mysql-store-mysql-set-0]
   ```

   

4. 確認Application 、PVC、PV 復原完成

   ```
   $ oc get all -n ndk-mysql
   NAME              READY   STATUS    RESTARTS   AGE
   pod/mysql-set-0   1/1     Running   0          2m9s
   pod/mysql-set-1   1/1     Running   0          106s
   pod/mysql-set-2   1/1     Running   0          92s
   
   NAME                         READY   AGE
   statefulset.apps/mysql-set   3/3     2m10s
   
   
   
   $ oc get pv,pvc -n ndk-mysql
   NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS    REASON   AGE
   persistentvolume/pvc-06e3f4c4-c4f1-415f-8194-2e4d356570d0   5Gi        RWO            Delete           Bound    ndk-mysql/mysql-store-mysql-set-1    nutanixvolume            3m35s
   persistentvolume/pvc-094bda7b-6a4d-44f1-a275-288d73b5545a   3Gi        RWO            Delete           Bound    ndk-mysql/mysql-data-1-mysql-set-2   nutanixvolume            3m35s
   persistentvolume/pvc-5cc840da-2fc9-431f-8c99-9948fe8b70f1   5Gi        RWO            Delete           Bound    ndk-mysql/mysql-store-mysql-set-2    nutanixvolume            3m39s
   persistentvolume/pvc-928e91ad-87db-41b6-96bd-e12d86608cbf   5Gi        RWO            Delete           Bound    ndk-mysql/mysql-store-mysql-set-0    nutanixvolume            3m35s
   persistentvolume/pvc-9b9aba57-c449-419f-9a09-5e4960604321   3Gi        RWO            Delete           Bound    ndk-mysql/mysql-data-1-mysql-set-1   nutanixvolume            3m40s
   persistentvolume/pvc-dccfcf6d-c9fd-45c0-9a93-53d94e96fdf6   3Gi        RWO            Delete           Bound    ndk-mysql/mysql-data-1-mysql-set-0   nutanixvolume            3m40s
   
   NAME                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
   persistentvolumeclaim/mysql-data-1-mysql-set-0   Bound    pvc-dccfcf6d-c9fd-45c0-9a93-53d94e96fdf6   3Gi        RWO            nutanixvolume   3m48s
   persistentvolumeclaim/mysql-data-1-mysql-set-1   Bound    pvc-9b9aba57-c449-419f-9a09-5e4960604321   3Gi        RWO            nutanixvolume   3m48s
   persistentvolumeclaim/mysql-data-1-mysql-set-2   Bound    pvc-094bda7b-6a4d-44f1-a275-288d73b5545a   3Gi        RWO            nutanixvolume   3m48s
   persistentvolumeclaim/mysql-store-mysql-set-0    Bound    pvc-928e91ad-87db-41b6-96bd-e12d86608cbf   5Gi        RWO            nutanixvolume   3m48s
   persistentvolumeclaim/mysql-store-mysql-set-1    Bound    pvc-06e3f4c4-c4f1-415f-8194-2e4d356570d0   5Gi        RWO            nutanixvolume   3m48s
   persistentvolumeclaim/mysql-store-mysql-set-2    Bound    pvc-5cc840da-2fc9-431f-8c99-9948fe8b70f1   5Gi        RWO            nutanixvolume   3m48s
   ```

   ![image-20240523091940333](https://kenkenny.synology.me:5543/images/2024/05/image-20240523091940333.png)

   ![image-20240523092014313](https://kenkenny.synology.me:5543/images/2024/05/image-20240523092014313.png)

   ![image-20240523092030321](https://kenkenny.synology.me:5543/images/2024/05/image-20240523092030321.png)

5. Prism Central - Kubernetes Clusters

   ![image-20240523091852050](https://kenkenny.synology.me:5543/images/2024/05/image-20240523091852050.png)
