---
title: Arcfra-Nodes
description: Arcfra-Nodes
slug: Arcfra-Nodes
date: 2026-07-24T03:49:12+08:00
categories:
    - Knowledge Base Category
tags:
    - arcfra
    - ai
    - kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# 	Arcfra-Notes



## 預設帳密

| 功能         | 帳號       | 密碼     | 備註          |
| ------------ | ---------- | -------- | ------------- |
| ake node     | admin      | HC!r0cks | 要切換成 root |
| ake registry | admin      | HC!r0cks |               |
| AOC          | cloudtower | HC!r0cks |               |







## AKE Management 問題



預設 AKE Management  Cluster 的網段如下，如果有衝突，需要進入 AOC 底層的 Kubernetes 去修改 Configmaps

![image-20260225115038599](https://kenkenny.synology.me:5543/images/2026/02/image-20260225115038599.png)



```
kubectl get cm sks-manager-api-env -n cloudtower-system
NAME                  DATA   AGE
sks-manager-api-env   7      21h

kubectl get deployment sks-manager-api -n cloudtower-system


kubectl get cm sks-manager-api-env -o yaml -n cloudtower-system
apiVersion: v1
data:
  ENABLE_LOG: "TRUE"
  KUBECONFIG: /app/.kube/config
  LOG_ENABLE_PRETTY_PRINT: "TRUE"
  LOG_LEVEL: trace
  NODE_ENV: production
  PACKAGE_VM_PREFIX: ake-managed
  PORT: "8080"
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: sks-manager
    meta.helm.sh/release-namespace: cloudtower-system
  creationTimestamp: "2026-02-24T06:36:32Z"
  labels:
    app.kubernetes.io/managed-by: Helm
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:ENABLE_LOG: {}
        f:KUBECONFIG: {}
        f:LOG_ENABLE_PRETTY_PRINT: {}
        f:LOG_LEVEL: {}
        f:NODE_ENV: {}
        f:PACKAGE_VM_PREFIX: {}
        f:PORT: {}
      f:metadata:
        f:annotations:
          .: {}
          f:meta.helm.sh/release-name: {}
          f:meta.helm.sh/release-namespace: {}
        f:labels:
          .: {}
          f:app.kubernetes.io/managed-by: {}
    manager: helm
    operation: Update
    time: "2026-02-24T06:36:32Z"
  name: sks-manager-api-env
  namespace: cloudtower-system
  resourceVersion: "3639"
  uid: 4994d0d6-6d0d-4c9b-a256-2621ce1b84b1

```

修改 data 的地方

```
data:
  SKS_POD_NETWORK_CIDR: "172.16.0.0/16"
  SKS_SERVICE_CIDR: "10.96.0.0/22"
  
  172.16.0.0/16 改成 10.42.0.0/16
  
修改完重啟 deployment
kubectl rollout restart deployment sks-manager-api -n cloudtower-system
```

![image-20260225120451868](https://kenkenny.synology.me:5543/images/2026/02/image-20260225120451868.png)

AOC 查看management 安裝 Log

kubectl logs -f sks-manager-api-79fb8c7d79-v5bpk -n cloudtower-system



### 部署出現 sks-fileserver 錯誤

源自 ake management cluster 創建 pvc 時出現錯誤，問AI 看起來是需要 2的倍數

```
kubectl describe pvc -n sks-system-core Name: sks-fileserver Namespace: sks-system-core StorageClass: elf-csi-driver Status: Pending Volume: Labels: app=sks-fileserver Annotations: volume.beta.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver volume.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver Finalizers: [kubernetes.io/pvc-protection] Capacity: Access Modes: VolumeMode: Filesystem Used By: sks-fileserver-645886fd9c-lnnjs Events: Type Reason Age From Message ---- ------ ---- ---- ------- Warning ProvisioningFailed 5m54s com.arcfra.elf-csi-driver_ake-mgmt-controlplane-d9krl_637cf4c4-fd8a-4c0e-ab91-8deac447c828 failed to provision volume with StorageClass "elf-csi-driver": rpc error: code = Unknown desc = [POST /create-vm-volume][500] createVmVolumeInternalServerError &{Code:<nil> Message:0xc0004e7530 OperationName:0xc0004e7520 Path:0xc0004e7510 Props:map[createVmVolume:map[code:VM_VOLUME_SIZE_INVALID message:failed to create vm because vm_disks with size not multiple of 2.]] Stack:0xc0004e7540 Status:0xc000783a18} Warning ProvisioningFailed 5m53s
```

從 AOC 底層查看

```
 kubectl get secret sh.helm.release.v1.sks-manager.v1 \
>   -n cloudtower-system \
>   -o jsonpath='{.data.release}' | base64 -d | base64 -d | gzip -d


{"name":"sks-manager","info":{"first_deployed":"2026-02-24T14:36:32.075004914+08:00","last_deployed":"2026-02-24T14:36:32.075004914+08:00","deleted":"","description":"Install complete","status":"deployed"},"chart":{"metadata":{"name":"sks-manager","version":"1.4.7","description":"A Helm chart for managing SKS deployment","apiVersion":"v2","appVersion":"1.4.7","type":"application"},"lock":null,"templates":[{"name":"templates/_helpers.tpl","data":"e3svKgpFeHBhbmQgdGhlIG5hbWUgb2YgdGhlIGNoYXJ0LgoqL319Cnt7LSBkZWZpbmUgInNrcy1tYW5hZ2VyLm5hbWUiIC19fQp7ey0gZGVmYXVsdCAuQ2hhcnQuTmFtZSAuVmFsdWVzLm5hbWVPdmVycmlkZSB8IHRydW5jIDYzIHwgdHJpbVN1ZmZpeCAiLSIgfX0Ke3stIGVuZCB9fQoKe3svKgpDcmVhdGUgYSBkZWZhdWx0IGZ1bGx5IHF1Y
...
dWRlICJza3MtbWFuYWdlci5zZWxlY3RvckxhYmVscyIgLiB8IG5pbmRlbnQgNCB9fQo="}],"values":{"PersistentVolumeClaim":{"opt":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"1Gi"}}},"upload":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"50Gi"}}}},"autoscaling":{"enabled":false,"maxReplicas":100,"minReplicas":1,"targetCPUUtilizationPercentage":80},"cloudtower":{"host":"http://cloudtower-cloudtower.cloudtower-system","password":"K5yt3hcjtUE4Teqe","username":"system-service"},"database":{"url":"postgresql://prisma:prisma@cloudtower-cloudtower-postgres.cloudtower-system:5432/sks-manager-prisma?schema=sks-manager\u0026connect_timeout=10"},"dnsConfig":{"options":[{"name":"single-request-reopen"}]},"environment":{"LOG_ENABLE_PRETTY_PRINT":"TRUE","LOG_LEVEL":"trace","PACKAGE_VM_PREFIX":"sks-managed"},"fullnameOverride":"sks-manager-api","image":{"pullPolicy":"IfNotPresent","repository":"registry.smtx.io/cloudtower/sks-manager","tag":"v0.1.3"},"nameOverride":"sks-manager-api","replicaCount":1,"service":{"http_listen_port":8080,"port":80,"type":"ClusterIP"},"towerConfig":{"haEnabled":false},"volumeMounts":[{"mountPath":"/etc/kubernetes/admin.conf","name":"kubeconfig","subPath":"admin.conf"},{"mountPath":"/app/uploads","name":"uploads-pv"},{"mountPath":"/opt","name":"opt-pv"}],"volumes":[{"hostPath":{"path":"/etc/kubernetes","type":"Directory"},"name":"kubeconfig"},{"name":"uploads-pv","persistentVolumeClaim":{"claimName":"uploads-pvc"}},{"name":"opt-pv","persistentVolumeClaim":{"claimName":"opt-pvc"}}]},"schema":null,"files":null},"config":{"PersistentVolumeClaim":{"opt":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"1Gi"}}},"upload":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"50Gi"}}}},"autoscaling":{"enabled":false,"maxReplicas":100,"minReplicas":1,"targetCPUUtilizationPercentage":80},"cloudtower":{"host":"http://cloudtower-cloudtower.cloudtower-system","password":"K5yt3hcjtUE4Teqe","username":"system-service"},"database":{"url":"postgresql://prisma:prisma@cloudtower-cloudtower-postgres.cloudtower-system:5432/sks-manager-prisma?schema=sks-manager\u0026connect_timeout=10"},"environment":{"LOG_ENABLE_PRETTY_PRINT":"TRUE","LOG_LEVEL":"trace","PACKAGE_VM_PREFIX":"ake-managed"},"fullnameOverride":"sks-manager-api","image":{"pullPolicy":"IfNotPresent","repository":"registry.local/cloudtower/sks-manager","tag":"v1.4.7-arcfra"},"nameOverride":"sks-manager-api","replicaCount":1,"service":{"http_listen_port":8080,"port":80,"type":"ClusterIP"},"towerConfig":{"haEnabled":false},"volumeMounts":[{"mountPath":"/run/containerd","name":"containerd-mount"},{"mountPath":"/etc/kubernetes/admin.conf","name":"kubeconfig","subPath":"admin.conf"},{"mountPath":"/app/uploads","name":"uploads-pv"},{"mountPath":"/opt","name":"opt-pv"}],"volumes":[{"hostPath":{"path":"/run/containerd"},"name":"containerd-mount"},{"hostPath":{"path":"/etc/kubernetes","type":"Directory"},"name":"kubeconfig"},{"name":"uploads-pv","persistentVolumeClaim":{"claimName":"uploads-pvc"}},{"name":"opt-pv","persistentVolumeClaim":{"claimName":"opt-pvc"}}]},"manifest":"---\n# Source: sks-manager/templates/secret.yaml\napiVersion: v1\ndata:\n  DATABASE_URL: \"cG9zdGdyZXNxbDovL3ByaXNtYTpwcmlzbWFAY2xvdWR0b3dlci1jbG91ZHRvd2VyLXBvc3RncmVzLmNsb3VkdG93ZXItc3lzdGVtOjU0MzIvc2tzLW1hbmFnZXItcHJpc21hP3NjaGVtYT1za3MtbWFuYWdlciZjb25uZWN0X3RpbWVvdXQ9MTA=\"\n  CLOUDTOWER_HOST: \"aHR0cDovL2Nsb3VkdG93ZXItY2xvdWR0b3dlci5jbG91ZHRvd2VyLXN5c3RlbQ==\"\n  CLOUDTOWER_USERNAME: \"c3lzdGVtLXNlcnZpY2U=\"\n  CLOUDTOWER_PASSWORD: \"SzV5dDNoY2p0VUU0VGVxZQ==\"\nkind: Secret\nmetadata:\n  name: sks-manager-api\n  namespace: cloudtower-system\ntype: Opaque\n---\n# Source: sks-manager/templates/configmap.yaml\napiVersion: v1\nkind: ConfigMap\nmetadata:\n  name: sks-manager-api-env\ndata:\n  NODE_ENV: production\n  PORT: \"8080\"\n  KUBECONFIG: \"/app/.kube/config\"\n  ENABLE_LOG: \"TRUE\"\n  LOG_ENABLE_PRETTY_PRINT: \"TRUE\"\n  LOG_LEVEL: trace\n  PACKAGE_VM_PREFIX: ake-managed\n---\n# Source: sks-manager/templates/pvc.yaml\napiVersion: \"v1\"\nkind: PersistentVolumeClaim\nmetadata:\n  name: uploads-pvc\nspec:\n  accessModes:\n    - ReadWriteOnce\n  resources:\n    requests:\n      storage: 50Gi\n---\n# Source: sks-manager/templates/pvc.yaml\napiVersion: \"v1\"\nkind: PersistentVolumeClaim\nmetadata:\n  name: opt-pvc\nspec:\n  accessModes:\n    - ReadWriteOnce\n  resources:\n    requests:\n      storage: 1Gi\n---\n# Source: sks-manager/templates/service.yaml\napiVersion: v1\nkind: Service\nmetadata:\n  name: sks-manager-api\n  labels:\n    helm.sh/chart: sks-manager-1.4.7\n    app.kubernetes.io/name: sks-manager-api\n    app.kubernetes.io/instance: sks-manager\n    app.kubernetes.io/version: \"1.4.7\"\n    app.kubernetes.io/managed-by: Helm\nspec:\n  type: ClusterIP\n  ports:\n    - port: 80\n      targetPort: 8080\n      protocol: TCP\n      name: http\n  selector:\n    app.kubernetes.io/name: sks-manager-api\n    app.kubernetes.io/instance: sks-manager\n---\n# Source: sks-manager/templates/deployment.yaml\napiVersion: apps/v1\nkind: Deployment\nmetadata:\n  name: sks-manager-api\n  labels:\n    helm.sh/chart: sks-manager-1.4.7\n    app.kubernetes.io/name: sks-manager-api\n    app.kubernetes.io/instance: sks-manager\n    app.kubernetes.io/version: \"1.4.7\"\n    app.kubernetes.io/managed-by: Helm\nspec:\n  replicas: 1\n  selector:\n    matchLabels:\n      app.kubernetes.io/name: sks-manager-api\n      app.kubernetes.io/instance: sks-manager\n  template:\n    metadata:\n      annotations:\n        checksum/config: 6862720ae995591bf916dc5e5b36b9920659214cb01433303ef4624da4fda583\n        checksum/secret: ce8bb8256745dd210dc3daefe3a60e501cb858c6a7172d5a064fbda90312cdef\n      labels:\n        app.kubernetes.io/name: sks-manager-api\n        app.kubernetes.io/instance: sks-manager\n    spec:\n      securityContext:\n        null\n      containers:\n        - name: sks-manager-api\n          securityContext:\n            null\n          image: \"registry.local/cloudtower/sks-manager:v1.4.7-arcfra\"\n          imagePullPolicy: IfNotPresent\n          ports:\n            - containerPort: 8080\n          command: [\"/bin/sh\", \"-c\"]\n          args:\n            - |-\n              set -e\n              export PATH=/opt/sks/bin:/app/bin:$PATH\n              sh ./scripts/copy-kubeconfig.sh\n              pnpm prisma migrate deploy\n              pnpm prisma generate\n              pnpm start:prod\n          envFrom:\n          - configMapRef:\n              name: sks-manager-api-env\n          - secretRef:\n              name: sks-server\n          - secretRef:\n              name: sks-manager-api\n          resources:\n            null\n          volumeMounts:\n            - mountPath: /run/containerd\n              name: containerd-mount\n            - mountPath: /etc/kubernetes/admin.conf\n              name: kubeconfig\n              subPath: admin.conf\n            - mountPath: /app/uploads\n              name: uploads-pv\n            - mountPath: /opt\n              name: opt-pv\n      dnsConfig:\n        options:\n        - name: single-request-reopen\n      volumes:\n        - hostPath:\n            path: /run/containerd\n          name: containerd-mount\n        - hostPath:\n            path: /etc/kubernetes\n            type: Directory\n          name: kubeconfig\n        - name: uploads-pv\n          persistentVolumeClaim:\n            claimName: uploads-pvc\n        - name: opt-pv\n          persistentVolumeClaim:\n            claimName: opt-pvc\n","version":1,"namespace":"cloudtower-system"}
```

最後使用在部署 management cluster 時 去刪除 pvc 並且修改成 6Gi

```
kubectl delete pvc sks-fileserver -n sks-system-core
persistentvolumeclaim "sks-fileserver" deleted
[root@ake-mgmt-controlplane-zsbcq ~]# vi pvc.yaml
[root@ake-mgmt-controlplane-zsbcq ~]# kubectl apply -f pvc.yaml 
persistentvolumeclaim/sks-fileserver created
[root@ake-mgmt-controlplane-zsbcq ~]# kubectl get pvc sks-fileserver -oyaml -n sks-system-core
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{"volume.beta.kubernetes.io/storage-provisioner":"com.arcfra.elf-csi-driver","volume.kubernetes.io/storage-provisioner":"com.arcfra.elf-csi-driver"},"creationTimestamp":"2026-02-26T08:59:12Z","finalizers":["kubernetes.io/pvc-protection"],"labels":{"app":"sks-fileserver"},"name":"sks-fileserver","namespace":"sks-system-core"},"spec":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"6Gi"}},"storageClassName":"elf-csi-driver","volumeMode":"Filesystem"}}
    volume.beta.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver
    volume.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver
  creationTimestamp: "2026-02-26T09:04:04Z"
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    app: sks-fileserver
  name: sks-fileserver
  namespace: sks-system-core
  resourceVersion: "5955"
  uid: c77670bb-f9df-49a9-a75d-1ff9919e9bc4
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 6Gi
  storageClassName: elf-csi-driver
  volumeMode: Filesystem
status:
  phase: Pending
[root@ake-mgmt-controlplane-zsbcq ~]# kubectl get pvc sks-fileserver -oyaml -n sks-system-core
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{"volume.beta.kubernetes.io/storage-provisioner":"com.arcfra.elf-csi-driver","volume.kubernetes.io/storage-provisioner":"com.arcfra.elf-csi-driver"},"creationTimestamp":"2026-02-26T08:59:12Z","finalizers":["kubernetes.io/pvc-protection"],"labels":{"app":"sks-fileserver"},"name":"sks-fileserver","namespace":"sks-system-core"},"spec":{"accessModes":["ReadWriteOnce"],"resources":{"requests":{"storage":"6Gi"}},"storageClassName":"elf-csi-driver","volumeMode":"Filesystem"}}
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver
    volume.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver
  creationTimestamp: "2026-02-26T09:04:04Z"
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    app: sks-fileserver
  name: sks-fileserver
  namespace: sks-system-core
  resourceVersion: "6019"
  uid: c77670bb-f9df-49a9-a75d-1ff9919e9bc4
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 6Gi
  storageClassName: elf-csi-driver
  volumeMode: Filesystem
  volumeName: pvc-c77670bb-f9df-49a9-a75d-1ff9919e9bc4
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 6Gi
  phase: Bound
[root@ake-mgmt-controlplane-zsbcq ~]# kubectl describe pvc sks-fileserver -n sks-system-core
Name:          sks-fileserver
Namespace:     sks-system-core
StorageClass:  elf-csi-driver
Status:        Bound
Volume:        pvc-c77670bb-f9df-49a9-a75d-1ff9919e9bc4
Labels:        app=sks-fileserver
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver
               volume.kubernetes.io/storage-provisioner: com.arcfra.elf-csi-driver
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      6Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       sks-fileserver-645886fd9c-797hz
Events:
  Type    Reason                 Age   From                                                                                        Message
  ----    ------                 ----  ----                                                                                        -------
  Normal  Provisioning           21s   com.arcfra.elf-csi-driver_ake-mgmt-controlplane-zsbcq_8c4b8562-aaf1-4c56-b843-86f3819b6327  External provisioner is provisioning volume for claim "sks-system-core/sks-fileserver"
  Normal  ExternalProvisioning   21s   persistentvolume-controller                                                                 waiting for a volume to be created, either by external provisioner "com.arcfra.elf-csi-driver" or manually created by system administrator
  Normal  ProvisioningSucceeded  14s   com.arcfra.elf-csi-driver_ake-mgmt-controlplane-zsbcq_8c4b8562-aaf1-4c56-b843-86f3819b6327  Successfully provisioned volume pvc-c77670bb-f9df-49a9-a75d-1ff9919e9bc4
```

![image-20260226171027910](https://kenkenny.synology.me:5543/images/2026/02/image-20260226171027910.png)

看起來是這個版本都有這個問題 AKE v1.5.1 , AOC 4.8.0



## AOC 改 DNS

./installer nameserver set --values "192.168.102.1,192.168.102.2,8.8.8.8"



## AOC 接 AD

1. AD 新增使用者不會自動同步，需要手動按同步按鈕
2. 一個使用者不能有兩個 Role，但是 ake workload cluster 又是另一個 role ，無法自定義，變成要建另一組 AD 帳號
3. 設定上分三個區域，第一個區域是名稱、第二個是連線資訊、第三個是對應規則
    第二區域 Configuration information : 
    Filter (這個filter指的是不要同步的項目): (|(objectClass=computer)(objectClass=group)(objectClass=container)(objectClass=organizationalUnit))
    第三區域 Mapping rule :  role mapping 建議要設定，不然會把全部搜尋到的User都拉進來，
    有設定好，下面的 sync For users outside criterion 選項就可以選 Do not sync 
    範例：
    (&(objectClass=person)(memberOf=CN=ITAdmin,OU=NutanixTeam,OU=NetfosTaipei,DC=nutanixlab,DC=local)) -> 選 Role
4. 無像Nutanix對基礎架構的資源配額、網段配置等分權功能

![Screenshot 2025-11-26 11.20.24](https://kenkenny.synology.me:5543/images/2026/02/Screenshot 2025-11-26 11.20.24.png)

![Screenshot 2025-11-26 11.20.21](https://kenkenny.synology.me:5543/images/2026/02/Screenshot 2025-11-26 11.20.21.png)

![Screenshot 2025-11-26 11.20.34](https://kenkenny.synology.me:5543/images/2026/02/Screenshot 2025-11-26 11.20.34.png)

```
BaseDN: DC=arcfralab,DC=local
BindDN: CN=Administrator,CN=Users,DC=arcfralab,DC=local
Filter: (|(objectClass=computer)(objectClass=group)(objectClass=container)(objectClass=organizationalUnit))

Arcfra@80670781


Name: cn
Username: sAMAccountName
Phone number: Telephone
Email: mail

(&(objectClass=person)(memberOf=CN=arcfra-admins,OU=Managers,OU=IT,OU=Netfos,DC=arcfralab,DC=local))
(&(objectClass=person)(memberOf=CN=arcfra-users,OU=Operators,OU=Netfos,DC=arcfralab,DC=local))

CN=Ken Wang,OU=Managers,OU=IT,OU=Netfos,DC=arcfralab,DC=local
CN=VPN User1,OU=Operators,OU=Netfos,DC=arcfralab,DC=local
```



![image-20260226111654794](https://kenkenny.synology.me:5543/images/2026/02/image-20260226111654794.png)

![image-20260226111704148](https://kenkenny.synology.me:5543/images/2026/02/image-20260226111704148.png)



## GPU

### VM MIG (不支援)

https://docs.nvidia.com/datacenter/tesla/mig-user-guide/virtualization.html

```
$ sudo nvidia-smi -i 0 -mig 1

Enabled MIG Mode for GPU 00000000:98:00.0
All done.

$ nvidia-smi -q

==============NVSMI LOG==============

Timestamp                                 : Fri Mar  6 10:26:53 2026
Driver Version                            : 550.127.06
CUDA Version                              : Not Found
vGPU Driver Capability
        Heterogenous Multi-vGPU           : Supported

Attached GPUs                             : 1
GPU 00000000:98:00.0
    Product Name                          : NVIDIA H100 NVL
    Product Brand                         : NVIDIA
    Product Architecture                  : Hopper
    Display Mode                          : Enabled
    Display Active                        : Disabled
    Persistence Mode                      : Enabled
    Addressing Mode                       : N/A
    vGPU Device Capability
        Fractional Multi-vGPU             : Not Supported
        Heterogeneous Time-Slice Profiles : Supported
        Heterogeneous Time-Slice Sizes    : Not Supported
    MIG Mode
        Current                           : Enabled
        Pending                           : Enabled


$ nvidia-smi mig -lgip

+-----------------------------------------------------------------------------+
| GPU instance profiles:                                                      |
| GPU   Name             ID    Instances   Memory     P2P    SM    DEC   ENC  |
|                              Free/Total   GiB              CE    JPEG  OFA  |
|=============================================================================|
|   0  MIG 1g.12gb       19     7/7        10.62      No     16     1     0   |
|                                                             1     1     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 1g.12gb+me    20     1/1        10.62      No     16     1     0   |
|                                                             1     1     1   |
+-----------------------------------------------------------------------------+
|   0  MIG 1g.24gb       15     4/4        21.50      No     26     1     0   |
|                                                             1     1     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 2g.24gb       14     3/3        21.50      No     32     2     0   |
|                                                             2     2     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 3g.47gb        9     2/2        46.12      No     60     3     0   |
|                                                             3     3     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 4g.47gb        5     1/1        46.12      No     64     4     0   |
|                                                             4     4     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 7g.94gb        0     1/1        92.62      No     132    7     0   |
|                                                             8     7     1   |
+-----------------------------------------------------------------------------+


$ sudo nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C

Successfully created GPU instance ID 13 on GPU  0 using profile MIG 1g.12gb (ID 19)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID 13 using profile MIG 1g.12gb (ID  0)
Successfully created GPU instance ID 11 on GPU  0 using profile MIG 1g.12gb (ID 19)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID 11 using profile MIG 1g.12gb (ID  0)
Successfully created GPU instance ID 12 on GPU  0 using profile MIG 1g.12gb (ID 19)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID 12 using profile MIG 1g.12gb (ID  0)
Successfully created GPU instance ID  7 on GPU  0 using profile MIG 1g.12gb (ID 19)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  7 using profile MIG 1g.12gb (ID  0)
Successfully created GPU instance ID  8 on GPU  0 using profile MIG 1g.12gb (ID 19)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  8 using profile MIG 1g.12gb (ID  0)
Successfully created GPU instance ID  9 on GPU  0 using profile MIG 1g.12gb (ID 19)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  9 using profile MIG 1g.12gb (ID  0)
Successfully created GPU instance ID 10 on GPU  0 using profile MIG 1g.12gb (ID 19)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID 10 using profile MIG 1g.12gb (ID  0)

$ nvidia-smi -L
GPU 0: NVIDIA H100 NVL (UUID: GPU-f4548330-27be-041a-ad8e-95b3005644b8)
  MIG 1g.12gb     Device  0: (UUID: MIG-1baf66e9-58ab-5d66-9f62-1adbdbe5c7c3)
  MIG 1g.12gb     Device  1: (UUID: MIG-3eb7b629-9a3c-550d-976d-86d336ed8d2e)
  MIG 1g.12gb     Device  2: (UUID: MIG-1284de0e-265e-5f21-b795-5e69e5c3bb86)
  MIG 1g.12gb     Device  3: (UUID: MIG-bfebf8fd-a0dc-57b1-b8aa-04d1bf0d5f4a)
  MIG 1g.12gb     Device  4: (UUID: MIG-2316e6e6-cec1-5a36-b9fe-e7b61f3890c4)
  MIG 1g.12gb     Device  5: (UUID: MIG-7218a604-141c-5a3d-ab27-6a5acc63dc52)
  MIG 1g.12gb     Device  6: (UUID: MIG-7a52b067-66ca-54eb-b2da-4bf4d9673bb7)

$ nvidia-smi
Fri Mar  6 10:55:12 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.127.06             Driver Version: 550.127.06     CUDA Version: N/A      |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H100 NVL                On  |   00000000:98:00.0 Off |                   On |
| N/A   45C    P0             65W /  400W |      88MiB /  95830MiB |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |                     Memory-Usage |        Vol|      Shared           |
|      ID  ID  Dev |                       BAR1-Usage | SM     Unc| CE ENC DEC OFA JPG    |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  0    7   0   0  |              13MiB / 10880MiB    | 16      0 |  1   0    1    0    1 |
|                  |                 0MiB / 16383MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
|  0    8   0   1  |              13MiB / 10880MiB    | 16      0 |  1   0    1    0    1 |
|                  |                 0MiB / 16383MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
|  0    9   0   2  |              13MiB / 10880MiB    | 16      0 |  1   0    1    0    1 |
|                  |                 0MiB / 16383MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
|  0   10   0   3  |              13MiB / 10880MiB    | 16      0 |  1   0    1    0    1 |
|                  |                 0MiB / 16383MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
|  0   11   0   4  |              13MiB / 10880MiB    | 16      0 |  1   0    1    0    1 |
|                  |                 0MiB / 16383MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
|  0   12   0   5  |              13MiB / 10880MiB    | 16      0 |  1   0    1    0    1 |
|                  |                 0MiB / 16383MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
|  0   13   0   6  |              13MiB / 10880MiB    | 16      0 |  1   0    1    0    1 |
|                  |                 0MiB / 16383MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+


$ dmesg | grep -i nvidia

$ sudo systemctl status nvidia-vgpud

$ lsmod | grep vfio

# destroy
$ sudo nvidia-smi mig -dci && sudo nvidia-smi mig -dgi
$ sudo nvidia-smi -i 0 -mig 0


```

![image-20260306115510542](https://kenkenny.synology.me:5543/images/2026/03/image-20260306115510542.png)

### AKE MIG & MPS

https://docs.nvidia.com/datacenter/tesla/mig-user-guide/supported-mig-profiles.html

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html

![img](https://kenkenny.synology.me:5543/images/2026/04/h100-profiles-v1.png)

![image-20260407112009186](https://kenkenny.synology.me:5543/images/2026/04/image-20260407112009186.png)



![image-20260309172244949](https://kenkenny.synology.me:5543/images/2026/03/image-20260309172244949.png)

![image-20260407140200427](https://kenkenny.synology.me:5543/images/2026/04/image-20260407140200427.png)

```
手動加上 migManager 那段等於 true
dcgmExporter:
  enabled: true
  serviceMonitor:
    enabled: true
driver:
  licensingConfig:
    nlsEnabled: false
migManager:
  enabled: true
# 修改此設定，預設是 single ，有需要 mixed 在這邊要修改，以及 config 有調整的話也要加上去  
mig:
  strategy: mixed
migManager:
  config: 
    default: all-disabled
    name: custom-mig-config   
  enabled: true
 
 ＃ MPS 設定
 
devicePlugin:
  config:
    name: nvidia-device-plugin-configs
    default: h100-mps
    
 # configmap 要先 apply
 
$ cat mps-configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-device-plugin-configs
  namespace: sks-system-nvidia-gpu
data:
  h100-mps: |-
    version: v1
    sharing:
      mps:
        resources:
        - name: nvidia.com/gpu
          replicas: 20
          
$ kubectl describe node ake-aic-gpu-workergroup-gpu-27r44-k9rj5

                    nvidia.com/gpu.present=true
                    nvidia.com/gpu.product=NVIDIA-H100-NVL-SHARED
                    nvidia.com/gpu.replicas=20
                    nvidia.com/gpu.sharing-strategy=mps
                    nvidia.com/mig.capable=true
                    nvidia.com/mig.config=all-disabled
                    nvidia.com/mig.config.state=success
                    nvidia.com/mig.strategy=single
                    nvidia.com/mps.capable=true
 
```

![image-20260309172401232](https://kenkenny.synology.me:5543/images/2026/03/image-20260309172401232.png)



```
$ kubectl get nodes

NAME                                      STATUS   ROLES           AGE    VERSION
ake-aic-gpu-controlplane-2v2n5            Ready    control-plane   3d2h   v1.30.13
ake-aic-gpu-workergroup-2-jq8kk-bblnj     Ready    <none>          3d1h   v1.30.13
ake-aic-gpu-workergroup-gpu-wx6mf-hcxnp   Ready    <none>          3d1h   v1.30.13

$ kubectl get node -o json | jq '.items[].metadata.labels'

  "nvidia.com/gpu-driver-upgrade-state": "upgrade-done",
  "nvidia.com/gpu.compute.major": "9",
  "nvidia.com/gpu.compute.minor": "0",
  "nvidia.com/gpu.count": "1",
  "nvidia.com/gpu.deploy.container-toolkit": "true",
  "nvidia.com/gpu.deploy.dcgm": "true",
  "nvidia.com/gpu.deploy.dcgm-exporter": "true",
  "nvidia.com/gpu.deploy.device-plugin": "true",
  "nvidia.com/gpu.deploy.driver": "true",
  "nvidia.com/gpu.deploy.gpu-feature-discovery": "true",
  "nvidia.com/gpu.deploy.mig-manager": "true",
  "nvidia.com/gpu.deploy.node-status-exporter": "true",
  "nvidia.com/gpu.deploy.nvsm": "",
  "nvidia.com/gpu.deploy.operator-validator": "true",
  "nvidia.com/gpu.family": "hopper",
  "nvidia.com/gpu.machine": "Arcfra-Virtual-Machine",
  "nvidia.com/gpu.memory": "95830",
  "nvidia.com/gpu.present": "true",
  "nvidia.com/gpu.product": "NVIDIA-H100-NVL",
  "nvidia.com/gpu.replicas": "1",
  "nvidia.com/mig.capable": "true",
  "nvidia.com/mig.strategy": "single"
 
# 調整 MIG 模式 Single 
$ kubectl patch clusterpolicies.nvidia.com/cluster-policy \
    --type='json' \
    -p='[{"op":"replace", "path":"/spec/mig/strategy", "value":"single"}]'
    

$ kubectl get cm -n sks-system-nvidia-gpu -o yaml  
  
# 調整 MIG Profile 成 1g.12gb  
$ kubectl label nodes ake-aic-gpu-workergroup-gpu-2xhsq-46pn4 nvidia.com/mig.config=all-1g.12gb --overwrite

# Node 有重開 MIG Profile 要重設
$ kubectl label nodes ake-aic-gpu-workergroup-gpu-2xhsq-46pn4 nvidia.com/mig.config=all-1g.12gb --overwrite

# 因為要跑 Extrahop 所以改 Profile 為 1g.24gb
$ kubectl label nodes ake-aic-gpu-workergroup-gpu-27r44-k9rj5  nvidia.com/mig.config=all-1g.24gb --overwrite

$ kubectl label nodes ake-aic-gpu-workergroup-gpu-27r44-k9rj5 nvidia.com/mig.config=all-3g.47gb --overwrite

# 確認 MIG Profile
$ kubectl get node -o json | jq '.items[].metadata.labels'
  "nvidia.com/gpu.family": "hopper",
  "nvidia.com/gpu.machine": "Arcfra-Virtual-Machine",
  "nvidia.com/gpu.memory": "11008",
  "nvidia.com/gpu.multiprocessors": "16",
  "nvidia.com/gpu.present": "true",
  "nvidia.com/gpu.product": "NVIDIA-H100-NVL-MIG-1g.12gb",
  "nvidia.com/gpu.replicas": "1",
  "nvidia.com/gpu.slices.ci": "1",
  "nvidia.com/gpu.slices.gi": "1",
  "nvidia.com/mig.capable": "true",
  "nvidia.com/mig.config": "all-1g.12gb",
  "nvidia.com/mig.config.state": "success",
  "nvidia.com/mig.strategy": "single"
}

$ kubectl get node ake-aic-gpu-workergroup-gpu-27r44-k9rj5 -o=jsonpath='{.metadata.labels}' | jq .

$ kubectl describe ds -l app=nvidia-driver-daemonset -n sks-system-nvidia-gpu

# 設定 MIG 之前
$ kubectl exec -it -n sks-system-nvidia-gpu ds/nvidia-driver-daemonset-4.18.0-553.54.1.el8.10-rocky8.10 -- nvidia-smi -L
$ kubectl exec -it -n sks-system-nvidia-gpu ds/nvidia-driver-daemonset-5.14.0-611.54.1.el9.7-rocky9.7 -- nvidia-smi -L

GPU 0: NVIDIA H100 NVL (UUID: GPU-f4548330-27be-041a-ad8e-95b3005644b8)

# 設定 MIG 之後

GPU 0: NVIDIA H100 NVL (UUID: GPU-f4548330-27be-041a-ad8e-95b3005644b8)
  MIG 1g.12gb     Device  0: (UUID: MIG-1baf66e9-58ab-5d66-9f62-1adbdbe5c7c3)
  MIG 1g.12gb     Device  1: (UUID: MIG-3eb7b629-9a3c-550d-976d-86d336ed8d2e)
  MIG 1g.12gb     Device  2: (UUID: MIG-1284de0e-265e-5f21-b795-5e69e5c3bb86)
  MIG 1g.12gb     Device  3: (UUID: MIG-bfebf8fd-a0dc-57b1-b8aa-04d1bf0d5f4a)
  MIG 1g.12gb     Device  4: (UUID: MIG-2316e6e6-cec1-5a36-b9fe-e7b61f3890c4)
  MIG 1g.12gb     Device  5: (UUID: MIG-7218a604-141c-5a3d-ab27-6a5acc63dc52)
  MIG 1g.12gb     Device  6: (UUID: MIG-7a52b067-66ca-54eb-b2da-4bf4d9673bb7)

# Mixed Mode
$ kubectl patch clusterpolicies.nvidia.com/cluster-policy \
    --type='json' \
    -p='[{"op":"replace", "path":"/spec/mig/strategy", "value":"mixed"}]'

# 各一種 Profile
$ kubectl label nodes ake-aic-gpu-workergroup-gpu-2xhsq-46pn4 nvidia.com/mig.config=all-balanced --overwrite
GPU 0: NVIDIA H100 NVL (UUID: GPU-f4548330-27be-041a-ad8e-95b3005644b8)
  MIG 3g.47gb     Device  0: (UUID: MIG-8afc9168-4c1d-55c7-895f-8a2ab1bc0af9)
  MIG 2g.24gb     Device  1: (UUID: MIG-a1996c99-4dc9-5e53-a1bb-0c0a7c09b29a)
  MIG 1g.12gb     Device  2: (UUID: MIG-1284de0e-265e-5f21-b795-5e69e5c3bb86)
  
# 調整 profile -> 2g.24gb *2 , 1g.12gb *3 ，要新增 configmaps
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-mig-config
  namespace: sks-system-nvidia-gpu
data:
  config.yaml: |
    version: v1
    mig-configs:
      all-disabled:
        - devices: all
          mig-enabled: false
      
      three-12g-two-24g:
        - devices: all
          mig-enabled: true
          mig-devices:
            "1g.12gb": 3
            "2g.24gb": 2
---
$ kubectl apply -n sks-system-nvidia-gpu -f mig-config.yaml 
configmap/custom-mig-config created

# 直接修改沒用，要去 AKE add-on 那邊去修改
$ kubectl patch clusterpolicies.nvidia.com/cluster-policy \
    --type='json' \
    -p='[{"op":"replace", "path":"/spec/migManager/config/name", "value":"custom-mig-config"}]'
    
$ kubectl label nodes ake-aic-gpu-workergroup-gpu-2xhsq-46pn4 nvidia.com/mig.config=three-12g-two-24g --overwrite
# 確認 migStrategy 都是 mixed，修改 config 名稱為 custom-mig-config
$ kubectl edit clusterpolicy cluster-policy

GPU 0: NVIDIA H100 NVL (UUID: GPU-f4548330-27be-041a-ad8e-95b3005644b8)
  MIG 2g.24gb     Device  0: (UUID: MIG-f1ae2989-3591-5586-8076-81b04d8581bd)
  MIG 2g.24gb     Device  1: (UUID: MIG-4fcec092-16f7-59cc-8633-af1390a1e90c)
  MIG 1g.12gb     Device  2: (UUID: MIG-1284de0e-265e-5f21-b795-5e69e5c3bb86)
  MIG 1g.12gb     Device  3: (UUID: MIG-bfebf8fd-a0dc-57b1-b8aa-04d1bf0d5f4a)
  MIG 1g.12gb     Device  4: (UUID: MIG-f2c1c2d7-686f-5073-a233-c24663911d00)

# 清除 MIG Profile
$ kubectl label nodes ake-aic-gpu-workergroup-gpu-27r44-k9rj5  nvidia.com/mig.config=all-disabled --overwrite

$ kubectl exec -it -n sks-system-nvidia-gpu ds/nvidia-driver-daemonset-5.14.0-611.54.1.el9.7-rocky9.7 -- nvidia-smi -mig 0

# 查看 mig log
$ kubectl logs -f  -n sks-system-nvidia-gpu -l app=nvidia-mig-manager

```

kubectl edit clusterpolicy cluster-policy -> 直接修改沒用，要去 AKE add-on 那邊去修改

![image-20260407135458426](https://kenkenny.synology.me:5543/images/2026/04/image-20260407135458426.png)

to

![image-20260407135544543](https://kenkenny.synology.me:5543/images/2026/04/image-20260407135544543.png)

驗證

```
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd
spec:
  restartPolicy: OnFailure
  containers:
  - name: vectoradd
    image: nvidia/samples:vectoradd-cuda11.2.1
    resources:
      limits:
        nvidia.com/gpu: 1
  nodeSelector:
    nvidia.com/gpu.product: NVIDIA-H100-NVL-MIG-1g.12gb
EOF

nvidia.com/mig-1g.12gb: 1
nvidia.com/mig-2g.24gb: 1


$ kubectl logs cuda-vectoradd
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

![image-20260309174212346](https://kenkenny.synology.me:5543/images/2026/03/image-20260309174212346.png)

![image-20260407142713782](https://kenkenny.synology.me:5543/images/2026/04/image-20260407142713782.png)



```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-runner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app-runner
  template:
    metadata:
      labels:
        app: hello-app-runner
    spec:
# Toleratation added so that the pod gets scheduled only on the GPU node
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: hello-app-runner
          image: public.ecr.aws/aws-containers/hello-app-runner:latest
          resources:
            limits:
              nvidia.com/gpu: 1
```



![image-20260309174850283](https://kenkenny.synology.me:5543/images/2026/03/image-20260309174850283.png)



![image-20260309164929155](https://kenkenny.synology.me:5543/images/2026/03/image-20260309164929155.png)

![image-20260309164646086](https://kenkenny.synology.me:5543/images/2026/03/image-20260309164646086.png)

![image-20260309173925966](https://kenkenny.synology.me:5543/images/2026/03/image-20260309173925966.png)

```
$ kubectl describe ds -l app=nvidia-driver-daemonset -n sks-system-nvidia-gpu
Name:           nvidia-driver-daemonset-4.18.0-553.54.1.el8.10-rocky8.10
Selector:       app=nvidia-driver-daemonset
Node-Selector:  feature.node.kubernetes.io/kernel-version.full=4.18.0-553.54.1.el8_10.x86_64,nvidia.com/gpu.deploy.driver=true
Labels:         app=nvidia-driver-daemonset
                app.kubernetes.io/managed-by=gpu-operator
                helm.sh/chart=gpu-operator-v23.6.2
                nvidia.com/precompiled=true
Annotations:    deprecated.daemonset.template.generation: 1
                nvidia.com/last-applied-hash: 9be043cb744b35b1
                openshift.io/scc: nvidia-driver
Desired Number of Nodes Scheduled: 1
Current Number of Nodes Scheduled: 1
Number of Nodes Scheduled with Up-to-date Pods: 1
Number of Nodes Scheduled with Available Pods: 1
Number of Nodes Misscheduled: 0
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           app=nvidia-driver-daemonset
                    app.kubernetes.io/managed-by=gpu-operator
                    helm.sh/chart=gpu-operator-v23.6.2
                    nvidia.com/precompiled=true
  Annotations:      kubectl.kubernetes.io/default-container: nvidia-driver-ctr
  Service Account:  nvidia-driver
  Init Containers:
   k8s-driver-manager:
    Image:      192.168.90.21/sks/nvidia/cloud-native/k8s-driver-manager:v0.6.2
    Port:       <none>
    Host Port:  <none>
    Command:
      driver-manager
    Args:
      uninstall_driver
    Environment:
      NODE_NAME:                    (v1:spec.nodeName)
      NVIDIA_VISIBLE_DEVICES:      void
      ENABLE_GPU_POD_EVICTION:     true
      ENABLE_AUTO_DRAIN:           false
      DRAIN_USE_FORCE:             false
      DRAIN_POD_SELECTOR_LABEL:    
      DRAIN_TIMEOUT_SECONDS:       0s
      DRAIN_DELETE_EMPTYDIR_DATA:  false
      OPERATOR_NAMESPACE:           (v1:metadata.namespace)
    Mounts:
      /host from host-root (ro)
      /run/nvidia from run-nvidia (rw)
      /sys from host-sys (rw)
  Containers:
   nvidia-driver-ctr:
    Image:      192.168.90.21/sks/nvidia/driver:535.216.01-rocky8.10
    Port:       <none>
    Host Port:  <none>
    Command:
      nvidia-driver
    Args:
      init
    Startup:      exec [sh -c nvidia-smi && touch /run/nvidia/validations/.driver-ctr-ready] delay=60s timeout=60s period=10s #success=1 #failure=120
    Environment:  <none>
    Mounts:
      /dev/log from dev-log (rw)
      /host-etc/os-release from host-os-release (ro)
      /run/mellanox/drivers from run-mellanox-drivers (rw)
      /run/mellanox/drivers/usr/src from mlnx-ofed-usr-src (rw)
      /run/nvidia from run-nvidia (rw)
      /run/nvidia-topologyd from run-nvidia-topologyd (rw)
      /var/log from var-log (rw)
  Volumes:
   run-nvidia:
    Type:          HostPath (bare host directory volume)
    Path:          /run/nvidia
    HostPathType:  DirectoryOrCreate
   var-log:
    Type:          HostPath (bare host directory volume)
    Path:          /var/log
    HostPathType:  
   dev-log:
    Type:          HostPath (bare host directory volume)
    Path:          /dev/log
    HostPathType:  
   host-os-release:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/os-release
    HostPathType:  
   run-nvidia-topologyd:
    Type:          HostPath (bare host directory volume)
    Path:          /run/nvidia-topologyd
    HostPathType:  DirectoryOrCreate
   mlnx-ofed-usr-src:
    Type:          HostPath (bare host directory volume)
    Path:          /run/mellanox/drivers/usr/src
    HostPathType:  DirectoryOrCreate
   run-mellanox-drivers:
    Type:          HostPath (bare host directory volume)
    Path:          /run/mellanox/drivers
    HostPathType:  DirectoryOrCreate
   run-nvidia-validations:
    Type:          HostPath (bare host directory volume)
    Path:          /run/nvidia/validations
    HostPathType:  DirectoryOrCreate
   host-root:
    Type:          HostPath (bare host directory volume)
    Path:          /
    HostPathType:  
   host-sys:
    Type:               HostPath (bare host directory volume)
    Path:               /sys
    HostPathType:       Directory
  Priority Class Name:  system-node-critical
Events:                 <none>
```

### Container Registry



ake robot account

8cNdzXSjG0XlAewORH9MHjZQCFvzZ6Ld

```
kubectl create secret docker-registry harbor-creds \
  --docker-server="ake-registry.arcfralab.local" \
  --docker-username="robot$ake" \
  --docker-password="P@ssw0rd123" \
  --docker-email="ake@arcfralab.local" \
  -n default
  
kubectl create secret docker-registry harbor-ken \
  --docker-server="ake-registry.arcfralab.local" \
  --docker-username="ken" \
  --docker-password="P@ssw0rd123" \
  --docker-email="ken.wang@netfos.com.tw" \
  -n default
```



## Neutree AI

https://docs.neutree.ai/getting-started/quick-start-install/

需要至少四個 Loadbalancer IP 或是可能要改用 Ingress，但文件沒寫到

預設帳號 admin@neutree.local

api key ken : sk_bxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

下載 CLI 、設定 Docker Registry 跟憑證、設定 kubeconfig、Helm Chart 

```
sudo mkdir -p /etc/docker/certs.d/ake-registry.arcfralab.local
sudo cp path/to/your-ca.crt /etc/docker/certs.d/ake-registry.arcfralab.local/ca.crt
sudo cp path/to/your-ca.crt /usr/local/share/ca-certificates/ca.crt
sudo update-ca-certificates
sudo systemctl restart docker
```

![image-20260310173309607](https://kenkenny.synology.me:5543/images/2026/03/image-20260310173309607.png)



![image-20260310172426589](https://kenkenny.synology.me:5543/images/2026/03/image-20260310172426589.png)

### Push Control Image

```
neutree-cli-amd64 import controlplane \
  --package neutree-controlplane-v1.0.0-enterprise-amd64.tar.gz \
  --mirror-registry ake-registry.arcfralab.local/neutreeai \
  --registry-username ken \
  --registry-password P@ssw0rd123

用 Harbor (AOC 建立的 User Registry) 出現 403 Forbidden
docker push ake-registry.arcfralab.local/neutreeai/neutree/license-server:v1.0.0-enterprise
The push refers to repository [ake-registry.arcfralab.local/neutreeai/neutree/license-server]
e6b45b0eca41: Waiting 
767e56ba346a: Waiting 
unknown: unexpected status from HEAD request to https://ake-registry.arcfralab.local/v2/neutreeai/neutree/license-server/blobs/sha256:3e8aaad2ba8dd1ffad56facfa490eb2f1e0e69eb3e35e147f5e1ad5c08dfba38: 403 Forbidden

SSH 進入 User Registry 那台，切換成 root
找到 Log 
$ cd /var/log/harbor
$ tail -f proxy.log

Mar 11 10:44:25 172.18.0.1 proxy[6200]: 2026/03/11 02:44:25 [error] 6#0: *20651 access forbidden by rule, client: 192.168.101.21, server: , request: "HEAD /v2/neutreeai/neutree/license-server/blobs/sha256:e6b45b0eca417658600fe5f740076c9593ecdab5e1b5b65b4bb8aeb0b7691fe8 HTTP/1.1", host: "ake-registry.arcfralab.local"
Mar 11 10:44:25 172.18.0.1 proxy[6200]: 192.168.101.21 - "HEAD /v2/neutreeai/neutree/license-server/blobs/sha256:e6b45b0eca417658600fe5f740076c9593ecdab5e1b5b65b4bb8aeb0b7691fe8 HTTP/1.1" 403 0 "-" "docker/29.2.1 go/go1.25.6 git-commit/6bc6209 kernel/6.8.0-101-generic os/linux arch/amd64 containerd-client/2.2.1+unknown storage-driver/overlayfs UpstreamClient(Docker-Client/29.2.1 \x5C(linux\x5C))" 0.000 - .


找到設定，確認有將含有 license 的字 blocked
$ cd /
$ find . -name "harbor.https.swagger.conf"
./var/lib/registry/harbor/common/config/nginx/conf.d/harbor.https.swagger.conf

$ docker exec -it nginx grep -r "license" /etc/nginx/
/etc/nginx/conf.d/harbor.https.swagger.conf:location ~* /license {

# 把 location ~* /license 註解掉
$ vi /var/lib/registry/harbor/common/config/nginx/conf.d/harbor.https.swagger.conf

location ~ ^(/devcenter-api-2.0|/swagger.*$) {
  auth_basic "Harbor Swagger API Docs";
  auth_basic_user_file /etc/nginx/.htpasswd;
  proxy_pass http://portal;
  proxy_set_header Host $http_host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $x_forwarded_proto;

  proxy_cookie_path / "/; HttpOnly; Secure";

  proxy_buffering off;
  proxy_request_buffering off;
}

#location ~* /license {
#  deny all;
#}
重啟 nginx
$ docker restart nginx

再次 push 成功
$ docker push ake-registry.arcfralab.local/neutreeai/neutree/license-server:v1.0.0-enterprise
The push refers to repository [ake-registry.arcfralab.local/neutreeai/neutree/license-server]
e6b45b0eca41: Mounted from neutreeai/neutree/l-server 
767e56ba346a: Mounted from neutreeai/neutree/neutree-db-scripts 
v1.0.0-enterprise: digest: sha256:c8973b7a81c78cca74c4c671554f70980f89719dfc987c76c0daeae4e7fcffea size: 556

再跑一次
neutree-cli-amd64 import controlplane \
  --package neutree-controlplane-v1.0.0-enterprise-amd64.tar.gz \
  --mirror-registry ake-registry.arcfralab.local/neutreeai \
  --registry-username ken \
  --registry-password P@ssw0rd123

Push 成功

✓ Successfully imported control plane package

Images Imported:
  • ake-registry.arcfralab.local/neutreeai/groundnuty/k8s-wait-for:v2.0
  • ake-registry.arcfralab.local/neutreeai/grafana/grafana:11.5.3
  • ake-registry.arcfralab.local/neutreeai/kong/kong:3.9
  • ake-registry.arcfralab.local/neutreeai/migrate/migrate:v4.18.3
  • ake-registry.arcfralab.local/neutreeai/neutree/jwt-cli:6.2.0
  • ake-registry.arcfralab.local/neutreeai/neutree/license-server:v1.0.0-enterprise
  • ake-registry.arcfralab.local/neutreeai/neutree/neutree-api:v1.0.0-enterprise
  • ake-registry.arcfralab.local/neutreeai/neutree/neutree-core:v1.0.0-enterprise
  • ake-registry.arcfralab.local/neutreeai/neutree/neutree-db-scripts:v1.0.0-enterprise
  • ake-registry.arcfralab.local/neutreeai/library/postgres:13
  • ake-registry.arcfralab.local/neutreeai/postgrest/postgrest:v14.0
  • ake-registry.arcfralab.local/neutreeai/supabase/gotrue:v2.170.0
  • ake-registry.arcfralab.local/neutreeai/supabase/postgres-meta:v0.86.0
  • ake-registry.arcfralab.local/neutreeai/timberio/vector:0.47.0-debian
  • ake-registry.arcfralab.local/neutreeai/victoriametrics/vmagent:v1.115.0
  • ake-registry.arcfralab.local/neutreeai/victoriametrics/vminsert:v1.115.0-cluster
  • ake-registry.arcfralab.local/neutreeai/victoriametrics/vmselect:v1.115.0-cluster
  • ake-registry.arcfralab.local/neutreeai/victoriametrics/vmstorage:v1.115.0-cluster
  

```

![image-20260310172504717](https://kenkenny.synology.me:5543/images/2026/03/image-20260310172504717.png)

![image-20260310173335786](https://kenkenny.synology.me:5543/images/2026/03/image-20260310173335786.png)

![image-20260311105623413](https://kenkenny.synology.me:5543/images/2026/03/image-20260311105623413.png)



### Helm Install

檔案下載

```
可以用 F12 -> Network -> 按下載的時候看 URL

$ curl -o "neutree-v1.0.0-enterprise.tgz" "https://ep-download.arcfra.com/ff/release-image/wTIrB9vOuJ-6kkmSjMV_E/neutree-v1.0.0-enterprise.tgz?Expires=1773212283&Key-Pair-Id=K8HIPCVS1VZEF&Signature=haOHWhm~cjH1OT7pJsyb~6sANziARilyFHeQdQAOjCu9z-viYdMJRR-m5SNYIwxwnFnBvii98DpHBxdJa7YqA~vpSUdbjORlM6pWcrTsiyZmvWM9J-Vvwpe8aQP2xXmwTc8NMano5IcWi0TxSKeHGzCvptq699nUmgL5iLKyxbZOjQZcG9xSqj7A-2O7hiA9XDhKEXbQVqkz71i2NLz6VcYumIZ0rA99CQBloDgl0kfv2S8R0sTx-GsuTptEy92AJd3X2PnPASXSQqsyTT6TP5il0M8xvCcSUxDf8F44dGfwsV5pNX~SLPjVY4PK4Ipfjs8Gt~bVCGkIhqNNi8j7NA__"
```

Helm 修改  values.yaml

```
$ helm show values neutree-v1.0.0-enterprise.tgz > values.yaml
$ cat values.yaml 

licenseServer:
  image:
    registry:
    repository: neutree/license-server
    tag: ""
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi

# Default values for neutree.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

global:
  # Global image registry for all components
  # When set, this registry will be used for:
  # - All main chart components (api, core, db, auth, etc.) via global.image.registry
  # - Victoria Metrics Cluster (via global.image.registry)
  # - Grafana (via global.imageRegistry)
  # Note: To use a unified registry, global.image.registry should equal global.imageRegistry.
  image:
    registry: ""
  imageRegistry: ""
  imagePullSecrets: []

jwtSecret: "mDCvM4zSk0ghmpyKhgqWb0g4igcOP0Lp"
# the password for the neutree admin user.
# it is valid when starting neutree core for the first time.
# It is recommended to change it quickly after installation.
adminPassword: ""


```

修改  registry , secret , pvc 大小不要是奇數 、API 的 Service Type 建議改 Loadbalancer ，然後我把 loadBalancerIP 都加上去變固定

```
JWT Secret 
$ openssl rand -base64 32 | tr '+/' '-_' | tr -d '='
_-WL6q9oloKngXFOoPrSFFvXvgAghqxHbM893sp8oMY


cat values.yaml 
licenseServer:
  image:
    registry: ake-registry.arcfralab.local/neutreeai
    repository: neutree/license-server
    tag: "v1.0.0-enterprise"
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi

# Default values for neutree.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

global:
  # Global image registry for all components
  # When set, this registry will be used for:
  # - All main chart components (api, core, db, auth, etc.) via global.image.registry
  # - Victoria Metrics Cluster (via global.image.registry)
  # - Grafana (via global.imageRegistry)
  # Note: To use a unified registry, global.image.registry should equal global.imageRegistry.
  image:
    registry: "ake-registry.arcfralab.local/neutreeai"
  imageRegistry: "ake-registry.arcfralab.local/neutreeai"
  imagePullSecrets: 
    - name: "harbor-ken"

jwtSecret: "_-WL6q9oloKngXFOoPrSFFvXvgAghqxHbM893sp8oMY"
# the password for the neutree admin user.
# it is valid when starting neutree core for the first time.
# It is recommended to change it quickly after installation.
adminPassword: "P@ssw0rd123"

system:
  grafana:
    url: ""

metrics:
  # Specify the URL for remote write if using an external metrics storage system.
  # Leave this empty to use the embedded Victoria Metrics cluster.
  remoteWriteUrl: ""

k8sWaitFor:
  image:
    registry: ghcr.io
    repository: groundnuty/k8s-wait-for
    tag: "v2.0"
    pullPolicy: IfNotPresent

dbScripts:
  image:
    registry: ""
    repository: neutree/neutree-db-scripts
    tag: ""
    pullPolicy: IfNotPresent

db:
  image:
    registry: ""
    repository: library/postgres
    tag: "13"
    pullPolicy: IfNotPresent
  user: postgres
  password: pgpassword
  name: neutree
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 2
      memory: 2Gi
  persistence:
    enabled: true
    size: 40Gi
    accessModes:
      - ReadWriteOnce
  service:
    type: ClusterIP
  nodeSelector: {}
  tolerations: []
  affinity: {}

auth:
  image:
    registry: ""
    repository: supabase/gotrue
    tag: "v2.170.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}
  service:
    type: ClusterIP

migration:
  image:
    registry: ""
    repository: migrate/migrate
    tag: v4.18.3
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}

postgrest:
  image:
    registry: ""
    repository: postgrest/postgrest
    tag: v14.0
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}
  service:
    type: ClusterIP

postgresmeta:
  image:
    registry: ""
    repository: supabase/postgres-meta
    tag: "v0.86.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}
  service:
    type: ClusterIP

core:
  image:
    registry: ""
    repository: neutree/neutree-core
    tag: ""
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}
  server:
    service:
      type: ClusterIP

api:
  image:
    registry: ""
    repository: neutree/neutree-api
    tag: ""
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  replicaCount: 1
  nodeSelector: {}
  tolerations: []
  affinity: {}
  service:
    type: LoadBalancer
    loadBalancerIP: 192.168.102.21

vmagent:
  image:
    registry: ""
    repository: victoriametrics/vmagent
    tag: "v1.115.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}
  persistence:
    enabled: true
    size: 2Gi
    accessModes:
      - ReadWriteOnce

kong:
  image:
    registry: ""
    repository: kong/kong
    tag: "3.9"
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 1
      memory: 2Gi
  nodeSelector: {}
  tolerations: []
  affinity: {}
  proxyService:
    type: LoadBalancer
    loadBalancerIP: 192.168.102.23

vector:
  image:
    registry: ""
    repository: timberio/vector
    tag: 0.47.0-debian
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}

jwtCli:
  image:
    registry: ""
    repository: neutree/jwt-cli
    tag: 6.2.0
    pullPolicy: IfNotPresent

victoria-metrics-cluster:
  enabled: true
  # global.image.registry is inherited from parent chart's global.image.registry
  vmselect:
    replicaCount: 1
  vminsert:
    replicaCount: 1
    service:
      type: LoadBalancer
      loadBalancerIP: 192.168.102.24
  vmstorage:
    replicaCount: 1
    persistentVolume:
      size: 40Gi

grafana:
  enabled: true
  # Grafana will automatically use global.imageRegistry from parent chart
  deploymentStrategy:
    type: Recreate
  image:
    # -- The Docker registry
    registry: "docker.io"
    # -- Docker image repository
    repository: grafana/grafana
    # Overrides the Grafana image tag whose default is the chart appVersion
    tag: "11.5.3"

  testFramework:
    enabled: false

  persistence:
    enabled: true
    size: 10Gi

  initChownData:
    enabled: false

  grafana.ini:
    auth.anonymous:
      enabled: true
    dashboards:
    # Path to the default home dashboard. If this value is empty, then Grafana uses StaticRootPath + "dashboards/home.json"
      default_home_dashboard_path: /var/lib/grafana/dashboards/default_grafana_dashboard.json
    analytics:
      check_for_plugin_updates: false
      check_for_updates: false
      enabled: false
      reporting_enabled: false
    plugins:
      public_key_retrieval_disabled: true
    security:
      preinstall_disabled: true
      allow_embedding: true
    grafana_net:
      url: ""
  dashboardProviders:
    dashboardproviders.yaml:
      - name: neutree
        orgId: 1
        folder: ''
        type: file
        options:
          path: /var/lib/grafana/dashboards
  assertNoLeakedSecrets: false

  adminUser: admin
  adminPassword: admin

  extraConfigmapMounts:
  - name: datasources
    mountPath: /etc/grafana/provisioning/datasources
    configMap: grafana-datasources
    readOnly: true
    optional: false
  - name: dashboards
    mountPath: /var/lib/grafana/dashboards
    configMap: grafana-dashboards
    readOnly: true
  service:
    enabled: true
    type: LoadBalancer
    loadBalancerIP: 192.168.102.22


```

#### Intall

```
# create namespace and secret

kubectl create ns neutree

kubectl create secret docker-registry harbor-ken \
  --docker-server="ake-registry.arcfralab.local" \
  --docker-username="ken" \
  --docker-password="P@ssw0rd123" \
  --docker-email="ken.wang@netfos.com.tw" \
  -n neutree
  
  
# helm install 
helm install neutree neutree-v1.0.0-enterprise.tgz -f values.yaml --namespace=neutree 

## helm upgrade
helm upgrade neutree neutree-v1.0.0-enterprise.tgz -f values.yaml --namespace=neutree 

## helm uninstall
helm uninstall neutree -n neutree

# check status 
kubectl get pods -n neutree
kubectl get pvc -n neutree
kubectl get svc -n neutree

# check admin password
$ kubectl -n neutree logs -l app.kubernetes.io/component=neutree-post-migration-hook-job
Defaulted container "post-migration-hook" out of: post-migration-hook, wait-postgresql (init), wait-migration-job (init), copy-seed-sql (init)
Executing seed file: /seed/001_default-role-and-users.sql
DO
psql:/seed/001_default-role-and-users.sql:148: NOTICE:  Created admin user: admin@neutree.local with password: P@ssw0rd123
DO
Executing seed file: /seed/002_default-workspaces.sql
DO
Executing seed file: /seed/999_notify_pgrst.sql
DO


```

預設帳號 ： admin@neutree.local 

密碼： P@ssw0rd

URL IP：192.168.102.21:3000

```
kubectl get pods -n neutree
NAME                                                         READY   STATUS      RESTARTS        AGE
license-server-df66bd465-5q6pd                               1/1     Running     4 (5m43s ago)   6m29s
neutree-api-69fdfcff44-bz8tt                                 1/1     Running     0               6m29s
neutree-auth-7664ddb94-2v86k                                 1/1     Running     0               6m29s
neutree-core-6df55d46fb-wkcd8                                1/1     Running     4 (5m32s ago)   6m29s
neutree-grafana-64f6b5dcb4-mxpk5                             1/1     Running     0               6m29s
neutree-kong-57db48b84c-4jrtk                                1/1     Running     0               6m29s
neutree-kong-init-migrations-1-zgfwj                         0/1     Completed   0               6m29s
neutree-kong-post-upgrade-migrations-1-4xctd                 0/1     Completed   0               6m29s
neutree-kong-pre-upgrade-migrations-1-8fvmr                  0/1     Completed   0               6m29s
neutree-migration-job-1-vjrlw                                0/1     Completed   0               6m29s
neutree-post-migration-hook-job-1-4kncs                      0/1     Completed   0               6m29s
neutree-postgresmeta-b4455b75d-xgwq4                         1/1     Running     0               6m29s
neutree-postgresql-8d6f685dc-2hz9q                           1/1     Running     0               6m29s
neutree-postgrest-c6d967f5f-gvjpc                            1/1     Running     0               6m29s
neutree-vector-7db7fbfbdd-zrqjr                              1/1     Running     0               6m29s
neutree-victoria-metrics-cluster-vminsert-6577b8cfb5-rzx4v   1/1     Running     0               6m29s
neutree-victoria-metrics-cluster-vmselect-8485c976c7-45s8j   1/1     Running     0               6m29s
neutree-victoria-metrics-cluster-vmstorage-0                 1/1     Running     0               6m29s
neutree-vmagent-78f47c7798-xng8w                             1/1     Running     0               6m29s


kubectl get svc -n neutree
NAME                                         TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
license-server                               ClusterIP      10.96.3.68    <none>           3002/TCP                     2m53s
neutree-api-service                          LoadBalancer   10.96.1.126   192.168.102.21   3000:31818/TCP               2m53s
neutree-auth-service                         ClusterIP      10.96.2.64    <none>           9999/TCP                     2m53s
neutree-core-server-service                  ClusterIP      10.96.1.173   <none>           3001/TCP                     2m53s
neutree-grafana                              LoadBalancer   10.96.3.212   192.168.102.22   80:32512/TCP                 2m53s
neutree-kong-admin                           ClusterIP      10.96.0.138   <none>           8001/TCP                     2m53s
neutree-kong-manager                         ClusterIP      10.96.3.65    <none>           8002/TCP                     2m53s
neutree-kong-proxy                           LoadBalancer   10.96.0.197   192.168.102.23   80:30554/TCP,443:31019/TCP   2m53s
neutree-postgresmeta-service                 ClusterIP      10.96.3.168   <none>           8080/TCP                     2m53s
neutree-postgresql-service                   ClusterIP      10.96.2.191   <none>           5432/TCP                     2m53s
neutree-postgrest-service                    ClusterIP      10.96.2.72    <none>           6432/TCP                     2m53s
neutree-vector                               ClusterIP      10.96.2.15    <none>           30122/TCP                    2m53s
neutree-victoria-metrics-cluster-vminsert    LoadBalancer   10.96.1.62    192.168.102.24   8480:30425/TCP               2m53s
neutree-victoria-metrics-cluster-vmselect    ClusterIP      10.96.1.155   <none>           8481/TCP                     2m53s
neutree-victoria-metrics-cluster-vmstorage   ClusterIP      None          <none>           8482/TCP,8401/TCP,8400/TCP   2m53s

neutree-api-service,neutree-kong-proxy  兩個都吃不到 value.yaml 設定的 IP
所以要進去 svc 內去調整，前提是他們都拿到你想要的IP，不然要重新部署看運氣

kubectl edit svc neutree-kong-proxy -n neutree
  type: LoadBalancer
  loadBalancerIP: 192.168.102.23

kubectl edit svc neutree-api-service -n neutree
  type: LoadBalancer
  loadBalancerIP: 192.168.102.23
  

```

![image-20260311135752251](https://kenkenny.synology.me:5543/images/2026/03/image-20260311135752251.png)

#### Upgrade

If you have deployed Neutree version 1.0.0, first [upgrade the CLI tool](https://docs.neutree.ai/1.0.1/user_guide/upgrade/upgrade-cli/), then [upgrade the management plane](https://docs.neutree.ai/1.0.1/user_guide/upgrade/upgrade-notes/upgrade.md). After the management plane upgrade is complete, you can [upgrade the cluster version](https://docs.neutree.ai/1.0.1/user_guide/upgrade/upgrade-cluster-version/) online as needed.

下載新版 CLI Tool

```
$ curl -o neutree-cli-amd64 "https://ep-download.arcfra.com/ff/release-image/FE4QpdQw5wNqaiANUALxR/neutree-cli-amd64?Expires=1777289237&Key-Pair-Id=K8HIPCVS1VZEF&Signature=AQA8s7nFiLnPgdv8Z97Ro5OZtuSkVa7gZWUlgFH5xKNgxyp~eTTkT3XTzyQ~jwzPGFfYbUySHP7xWh9a4nzhSg5oD4luiwXbsBQrj6UmIg7eRhkH-BHAKriJu9Fv8Lr2VpEIGUNKGCfToBdKztQIHBFya2D3VVvkFM1Mkb7JmpzyjT-QqwS477Bo2G3OKu63Ra2gc0-eknxpqjqcCAuyu-mfyVsKmqOuNhmCxG~KIvpyua5QafYmQ7kKroFHXUDA4Cn0MMbB04JzHw2ES55CQBn8m7h7ar05t~urcCI0-EV6iYD6w1mzpRmGKDHC~UyhlCNsFa9QB6pDNGJ9hpxo6A__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 53.3M  100 53.3M    0     0  13.3M      0  0:00:04  0:00:04 --:--:-- 13.3M

$ ll
total 54616
drwxrwxr-x 2 ken ken     4096 Apr 27 07:31 ./
drwxrwxr-x 5 ken ken     4096 Apr 27 07:30 ../
-rw-rw-r-- 1 ken ken 55915904 Apr 27 07:31 neutree-cli-amd64
$ chmod +x neutree-cli-amd64 
$ ./neutree-cli-amd64 version
Version: v1.0.1-enterprise
Git Commit: 0a87ec4
Build Time: 2026-04-08T09:17:17+00:00
Go Version: go1.23.0
Platform: linux/amd64
$ 

```

下載 controlplane 並import , upgrade

```
curl -o neutree-controlplane-v1.0.1-enterprise-amd64.tar.gz "https://ep-download.arcfra.com/ff/release-image/YQAjJHaoLqyqTm2uwuqZS/neutree-controlplane-v1.0.1-enterprise-amd64.tar.gz?Expires=1777289578&Key-Pair-Id=K8HIPCVS1VZEF&Signature=vC6ENs0Nt97jTvMZk~27j-8kytucrxhgL32F940d7lsBGTwiPj2SgPhAsjeRsQmu2Nq25EpcyR2hDBjtfdtOJ2Gy~H9YlXEIjqQqBV7HzN73toEjur8CGnP-0ghZSOPWYYK3s~vnkP2mNO517-9rRshs4Xc3W9NT6tEiomhz-kxzxznDlnLt8f01~snMfbMXu3Qy0ia1IhCbcA7LthcaE94YBSs7d7LUt5p6b1nS3n8y8H7JgUFCfKFsfQU9pWOpld3tuShLgQsxClCofW6o~wnHL-kX267OhN28j1N7Z8w5a5IacFXDefG~rYpfC4F1l-S1ckh8pPiQmjp4fOkgdw__"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 1354M  100 1354M    0     0  22.7M      0  0:00:59  0:00:59 --:--:-- 23.3M



neutree-cli-amd64 import controlplane \
  --package neutree-controlplane-v1.0.1-enterprise-amd64.tar.gz \
  --mirror-registry ake-registry.arcfralab.local \
  --registry-project neutreeai \
  --registry-username ken \
  --registry-password P@ssw0rd123

--- v1.1.0
neutree-cli-amd64 import controlplane \
  --package neutree-controlplane-v1.1.0-enterprise-amd64.tar.gz \
  --mirror-registry ake-registry.arcfralab.local \
  --registry-project neutreeai \
  --registry-username 'robot$neutree' \
  --registry-password "W5anl1O9DY0EvcJbt8mmQFouewrQwRvv"
  
helm show values ./neutree-v1.1.0-enterprise.tgz > values.yaml
此版本變動比較多，所以把舊版的一些資訊複製過來（image registry, pull secret, loadbalance ip 等等）
helm upgrade neutree neutree-v1.1.0-enterprise.tgz -f values.yaml \
  --namespace=neutree
----

檢查 value.yaml 跟 v1.0.0 版差不多，就差在 grafana 多了
    panels:
      disable_sanitize_html: true
所以把舊版拿來用，多上面這條

ken@ken-ubuntu-gpu:~/neutree/v1.0.1$ pwd
/home/ken/neutree/v1.0.1
ken@ken-ubuntu-gpu:~/neutree/v1.0.1$ ls -l
total 1575924
drwxrwxr-x 8 ken ken       4096 Apr 27 07:51 neutree
-rw-rw-r-- 1 ken ken  192847218 Apr 27 07:34 neutree-cluster-k8s-v1.0.1-amd64.tar.gz
-rw-rw-r-- 1 ken ken 1420747068 Apr 27 07:34 neutree-controlplane-v1.0.1-enterprise-amd64.tar.gz
-rw-rw-r-- 1 ken ken     128401 Apr 27 07:32 neutree-v1.0.1-enterprise.tgz
-rw-rw-r-- 1 ken ken       6748 Apr 27 07:58 values.yaml  

$  helm list -A
NAME   	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART                    	APP VERSION      
neutree	neutree  	1       	2026-03-11 06:27:43.807011811 +0000 UTC	deployed	neutree-v1.0.0-enterprise	v1.0.0-enterprise

$ helm upgrade neutree neutree-v1.0.1-enterprise.tgz -f values.yaml \
  --namespace=neutree
```

![image-20260427155018183](https://kenkenny.synology.me:5543/images/2026/04/image-20260427155018183.png)

![image-20260427160045195](https://kenkenny.synology.me:5543/images/2026/04/image-20260427160045195.png)

1.1.0

![image-20260716140243718](https://kenkenny.synology.me:5543/images/2026/07/image-20260716140243718.png)

v1.0.1 多了 External Endpoints 跟 Anthropic Compatible URL

![image-20260427160416809](https://kenkenny.synology.me:5543/images/2026/04/image-20260427160416809.png)

![image-20260427161440889](https://kenkenny.synology.me:5543/images/2026/04/image-20260427161440889.png)

![image-20260427160844816](https://kenkenny.synology.me:5543/images/2026/04/image-20260427160844816.png)

![image-20260427161528384](https://kenkenny.synology.me:5543/images/2026/04/image-20260427161528384.png)

Cluster 最後升級，上傳 k8s 包

```

Download neutree-cluster-k8s-v1.0.1-amd64.tar.gz

neutree-cli-amd64 import cluster \
--package neutree-cluster-k8s-v1.0.1-amd64.tar.gz \
--mirror-registry ake-registry.arcfralab.local/neutreeai \
--registry-username ken \
--registry-password P@ssw0rd123

neutree-cli-amd64 import cluster \
  --package neutree-cluster-k8s-v1.1.0-amd64.tar.gz \
  --mirror-registry ake-registry.arcfralab.local \
  --registry-project neutreeai \
  --registry-username 'robot$neutree' \
  --registry-password "W5anl1O9DY0EvcJbt8mmQFouewrQwRvv"
```

![image-20260427161546762](https://kenkenny.synology.me:5543/images/2026/04/image-20260427161546762.png)

UI 點選升級，做 rolling upgrade

![image-20260427161644389](https://kenkenny.synology.me:5543/images/2026/04/image-20260427161644389.png)

![image-20260427161704041](https://kenkenny.synology.me:5543/images/2026/04/image-20260427161704041.png)

升級完成，大約2分鐘

![image-20260427161835990](https://kenkenny.synology.me:5543/images/2026/04/image-20260427161835990.png)

![image-20260427161904174](https://kenkenny.synology.me:5543/images/2026/04/image-20260427161904174.png)



##### v1.1.0

新增 gpu 拆分功能

![image-20260716144341080](https://kenkenny.synology.me:5543/images/2026/07/image-20260716144341080.png)

![image-20260716144627059](https://kenkenny.synology.me:5543/images/2026/07/image-20260716144627059.png)

但是要先把原本 GPU Operator 的 devicePlugin 功能暫停

```
devicePlugin:
  enabled: false
```



![image-20260716145250379](https://kenkenny.synology.me:5543/images/2026/07/image-20260716145250379.png)

停用之後就多了 hami 的 pod 
**HAMi (Heterogeneous AI Multi-tenant Infrastructure)**（原名 `k8s-vgpu-scheduler`）是目前 CNCF 的沙盒（Sandbox）項目。它是一個開源的、**K8s 原生的 GPU 共享與虛擬化調度器**。

簡單來說，它的核心價值在於：讓多個 Pod 安全地共享同一張實體 GPU，且**不需要購買 NVIDIA 昂貴的商業版 vGPU（GRID）授權**，就能實現顯存（VRAM）和算力（vCore）的硬性隔離。

![image-20260716145940259](https://kenkenny.synology.me:5543/images/2026/07/image-20260716145940259.png)

catalog UI 可以直接部署

![image-20260716144656227](https://kenkenny.synology.me:5543/images/2026/07/image-20260716144656227.png)

部署畫面也變得更可視化

![image-20260716144735729](https://kenkenny.synology.me:5543/images/2026/07/image-20260716144735729.png)

monitoring 新增更多資訊

![image-20260716164305695](https://kenkenny.synology.me:5543/images/2026/07/image-20260716164305695.png)

![image-20260716164334642](https://kenkenny.synology.me:5543/images/2026/07/image-20260716164334642.png)

![image-20260716164347409](https://kenkenny.synology.me:5543/images/2026/07/image-20260716164347409.png)

![image-20260716164404552](https://kenkenny.synology.me:5543/images/2026/07/image-20260716164404552.png)

![image-20260716164416090](https://kenkenny.synology.me:5543/images/2026/07/image-20260716164416090.png)

Engine 多了 sglang，vllm 新版本也自動放上去了

![image-20260716144806450](https://kenkenny.synology.me:5543/images/2026/07/image-20260716144806450.png)

Model Gateway 多了 Usage, Access Log

![image-20260716144859404](https://kenkenny.synology.me:5543/images/2026/07/image-20260716144859404.png)

API Key 多了可以設定 usage, limits ，效能資訊

![image-20260716145018143](https://kenkenny.synology.me:5543/images/2026/07/image-20260716145018143.png)

![image-20260716145042712](https://kenkenny.synology.me:5543/images/2026/07/image-20260716145042712.png)



### Push Cluster Images

這部分 Image 比較大，所以會跑比較久

```
For Single Node
Download neutree-cluster-ssh-nvidia_gpu-v1.0.0-amd64.tar.gz

neutree-cli-amd64 import cluster \
--package neutree-cluster-ssh-nvidia_gpu-v1.0.0-amd64.tar.gz \
--mirror-registry ake-registry.arcfralab.local/neutreeai \
--registry-username ken \
--registry-password P@ssw0rd123
```

```
For K8s
Download neutree-cluster-k8s-v1.0.0-amd64.tar.gz

neutree-cli-amd64 import cluster \
--package neutree-cluster-k8s-v1.0.0-amd64.tar.gz \
--mirror-registry ake-registry.arcfralab.local/neutreeai \
--registry-username ken \
--registry-password P@ssw0rd123

```



![image-20260311151549321](https://kenkenny.synology.me:5543/images/2026/03/image-20260311151549321.png)

![image-20260311151802553](https://kenkenny.synology.me:5543/images/2026/03/image-20260311151802553.png)

![image-20260311152132196](https://kenkenny.synology.me:5543/images/2026/03/image-20260311152132196.png)

v1.1.0

![image-20260716140049240](https://kenkenny.synology.me:5543/images/2026/07/image-20260716140049240.png)

Inference Engine 的 Image 

有 vllm-v0.8.5 , vllm-v0.11.2 , llama-cpp-v0.3.7

FIle Share 有 vllm-v0.11.2 , llama-cpp-v0.3.7 也可以直接從 Docker Hub Pull 下來再 Push 到 Registry

前提是要先 Login docker.io

![image-20260313100127270](https://kenkenny.synology.me:5543/images/2026/03/image-20260313100127270.png)

```
$ docker pull vllm/vllm-openai:v0.11.2
$ docker images
$ docker tag vllm/vllm-openai:v0.11.2 ake-registry.arcfralab.local/neutreeai/vllm/vllm-openai:v0.11.2
$ docker push ake-registry.arcfralab.local/neutreeai/vllm/vllm-openai:v0.11.2
```

![image-20260313100101526](https://kenkenny.synology.me:5543/images/2026/03/image-20260313100101526.png)

![image-20260313101810875](https://kenkenny.synology.me:5543/images/2026/03/image-20260313101810875.png)

![image-20260313101752078](https://kenkenny.synology.me:5543/images/2026/03/image-20260313101752078.png)





### Config

#### Image Registries

![image-20260311152220972](https://kenkenny.synology.me:5543/images/2026/03/image-20260311152220972.png)

#### Cluster

可以選 Single or Cluster

![image-20260311152259155](https://kenkenny.synology.me:5543/images/2026/03/image-20260311152259155.png)

加入之後會在 K8s 自己建立 namespace 跟 Pod

![image-20260311153552188](https://kenkenny.synology.me:5543/images/2026/03/image-20260311153552188.png)

![image-20260311153618738](https://kenkenny.synology.me:5543/images/2026/03/image-20260311153618738.png)

![image-20260311153639463](https://kenkenny.synology.me:5543/images/2026/03/image-20260311153639463.png)

![image-20260311153653790](https://kenkenny.synology.me:5543/images/2026/03/image-20260311153653790.png)

![image-20260311153922788](https://kenkenny.synology.me:5543/images/2026/03/image-20260311153922788.png)

![image-20260311154129441](https://kenkenny.synology.me:5543/images/2026/03/image-20260311154129441.png)

![image-20260311154325173](https://kenkenny.synology.me:5543/images/2026/03/image-20260311154325173.png)

預設授權 90 天

![image-20260311160003149](https://kenkenny.synology.me:5543/images/2026/03/image-20260311160003149.png)

Hugging Face

arcfra-neutreeAI

hkeykeyk_xxxxxxxxxxxxxxxxxxxxxxxx

![image-20260311170505521](https://kenkenny.synology.me:5543/images/2026/03/image-20260311170505521.png)

![image-20260311170403066](https://kenkenny.synology.me:5543/images/2026/03/image-20260311170403066.png)



![image-20260311170421968](https://kenkenny.synology.me:5543/images/2026/03/image-20260311170421968.png)

AI 做的 Model catalog for llama-3-2-3b-instruct

```
apiVersion: v1
kind: ModelCatalog
metadata:
  name: llama-3-2-3b-instruct
  display_name: Meta/Llama-3.2-3B-Instruct
  labels:
    icon_url: https://mintlify.s3-us-west-1.amazonaws.com/llama/logo/light.png
    hf_repo_url: https://huggingface.co/meta-llama/Llama-3.2-3B-Instruct
spec:
  model:
    registry: ''
    name: meta-llama/Llama-3.2-3B-Instruct
    version: ''
    task: text-generation
  engine:
    engine: vllm
    version: v0.11.2
  resources: {}
  replicas:
    num: 1
  deployment_options:
    scheduler:
      type: consistent_hash
      virtual_nodes: 150
      load_factor: 1.25
  variables:
    RAY_SCHEDULER_TYPE: consistent_hash
    engine_args:
      tensor_parallel_size: 1
      max_model_len: 8192
      gpu_memory_utilization: 0.90
      enable_chunked_prefill: true
      served_model_name: meta-llama/Llama-3.2-3B-Instruct
```

API Key



sk_bbN1hG1qBD5I44TEBdxQJlJVcDwueDYvw_JrFfnpFlSqvpdG3xfXMqEdTx0R-uh-LKwGzJICa80NjQumgsib1w

測試成功

```
curl http://192.168.102.23/workspace/ake-aic-gpu/endpoint/llama-3.2-test/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk_bbN1hG1qBD5I44TEBdxQJlJVcDwueDYvw_JrFfnpFlSqvpdG3xfXMqEdTx0R-uh-LKwGzJICa80NjQumgsib1w" \
  -d '{
    "model": "meta-llama/Llama-3.2-3B-Instruct",
    "messages": [
      {"role": "user", "content": "你好，請自我介紹一下。"}
    ]
  }'
```

![image-20260313140358065](https://kenkenny.synology.me:5543/images/2026/03/image-20260313140358065.png)

![image-20260319154216379](https://kenkenny.synology.me:5543/images/2026/03/image-20260319154216379.png)

![image-20260319154229042](https://kenkenny.synology.me:5543/images/2026/03/image-20260319154229042.png)



公司內部測試

```
apiVersion: v1
kind: ModelCatalog
metadata:
  name: qwen35-27b-reasoning-distilled
  display_name: Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
  labels:
    icon_url: https://cdn-thumbnails.huggingface.co
    hf_repo_url: https://huggingface.co
spec:
  model:
    registry: ''
    name: Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
    version: ''
    task: text-generation
  engine:
    engine: vllm
    version: v0.11.2
  resources: {}
  replicas:
    num: 1
  deployment_options:
    scheduler:
      type: consistent_hash
      virtual_nodes: 150
      load_factor: 1.25
  variables:
    RAY_SCHEDULER_TYPE: consistent_hash
    engine_args:
      tensor_parallel_size: 1
      max_model_len: 4096
      gpu_memory_utilization: 0.90
      enable_chunked_prefill: true
      served_model_name: Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
      quantization: awq  # 必須使用量化版才能塞進 24GB
      enforce_eager: true
      trust_remote_code: true

```

### Model & Engine

#### 下載模型到 AFS File Server

```
# list model  
$ neutree-cli-amd64 model list -r afs-neutree \
  -w ake-aic-gpu \
  --api-key sk_xxxxxxxxxxxxxxxxxxxxxxx \
  --server-url http://192.168.102.21:3000
  
# login hugging face
$ hf login

pwd (NFS Path)
/mnt/neutree-nfs/qwen3.5-27b

# download
$ hf download Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled \
  --local-dir .
  
# upload (需要在 /mnt 建立資料夾)
$ sudo neutree-cli-amd64 model push /mnt/neutree-nfs/qwen3.5-27b  \
  -n Jackrong_Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled \
  -d extrahop \
  -r afs-neutree \
  -w ake-aic-gpu \
  --api-key sk_bbxxxxxxxxxxxxxxxxxxxxxxxvw_w \
  --server-url http://192.168.102.21:3000
  
# list model
$ neutree-cli-amd64 model list -r afs-neutree \
  -w ake-aic-gpu \
  --api-key sk_bbN1hG1xxxxxxxxxxxxxxxxxxxxxx3xfXMqEdTx0R-uh-LKwGzJICa80NjQumgsib1w \
  --server-url http://192.168.102.21:3000

```



![image-20260317103005214](https://kenkenny.synology.me:5543/images/2026/03/image-20260317103005214.png)

![image-20260317115648552](https://kenkenny.synology.me:5543/images/2026/03/image-20260317115648552.png)

![image-20260317120050521](https://kenkenny.synology.me:5543/images/2026/03/image-20260317120050521.png)

#### ModelCatalog For Extrahop

模型本身就需要約 58GB ，除非整張卡都使用，不然一定要量化

https://docs.vllm.ai/en/latest/configuration/engine_args/#cacheconfig

https://www.reddit.com/r/LocalLLaMA/comments/1rtggdo/trying_to_understand_vllm_kv_offloading_vs_hybrid/

url 

https://neutree-ai.skywebster.com/#/login

https://neutree-api.skywebster.com/workspace/ake-aic-gpu/endpoint/extrahop-27b-fp8

api key

extrahop

sk_bbN1hG1qBD5I44TEBdxQJlJFt-iYYxr6QLA2qWjo_VwRAH37PDCPKlev2gK3ma5mrdI0Ntgsq-A9ZoHEYu6CEw

測試 API

```
curl http://neutree-api.skywebster.com/workspace/ake-aic-gpu/endpoint/extrahop-27b-fp8/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk_bbN1hG1qBD5I44TEBdxQJlJFt-iYYxr6QLA2qWjo_VwRAH37PDCPKlev2gK3ma5mrdI0Ntgsq-A9ZoHEYu6CEw" \
  -d '{
    "model": "jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled",
    "messages": [
      {"role": "user", "content": "你好，請自我介紹一下。"}
    ]
  }'
```

![image-20260330164617826](https://kenkenny.synology.me:5543/images/2026/03/image-20260330164617826.png)

##### offload 設定說明

```
模型權重 (Weights):將模型的一部分層（Layers）放在 CPU RAM，只在需要計算時才載入 GPU。
模型太大，連 4-bit 量化後都塞不進 GPU 時（例如 24GB 顯卡跑 72B 模型）。
CPU 分兩種模式 UVA, Prefetch

UVA (Unified Virtual Addressing) : 零拷貝模式（預留記憶體）
原理： 讓 GPU 直接透過 PCIe 匯流排去讀取系統記憶體（RAM）裡的數據，不需要顯式的「搬運」動作。
功用： 統一增加虛擬的 GPU RAM，如果模型需要 34GB ，但是 GPU 只有 24GB，可以透過模擬的方式變成 34GB
優缺點： 設定最簡單，但每次運算都要經過 PCIe 讀取，速度最慢。
設定方式：
      #cpu-offload-gb: "10"

Prefetch : 異步搬運模式 (動態調整記憶體)
對應參數： --offload-group-size, --offload-num-in-group, --offload-prefetch-step
原理： 像流水線一樣，當 GPU 在算第 1 層時，後台偷偷把第 2 層從 RAM 搬進 GPU 備用。
功用： 透過「預先搬貨」來隱藏傳輸延遲。
優缺點： 速度比 UVA 快，但設定複雜，且需要 GPU 留出一部分空間來當緩衝區。
設定方式：
      offload_backend: "prefetch"
      offload-group-size: "6"
      offload-num-in-group: "1"
      offload-prefetch-step: "1"
      
對話緩存 (KV Cache):模型權重全在 GPU，但將「過去的對話紀錄」暫存到 CPU RAM。
上下文太長，GPU 放得下模型但放不下長達 32k/128k 的對話紀錄時。
設定方式：
      kv-offloading-backend: "native" or "lmcache"
      kv-offloading-size: "16"
      disable_hybrid_kv_cache_manager:
      kv-cache-dtype: "fp8"
```

##### 量化

```
      quantization: "bitsandbytes"

8bit
fp8

4bit
bitsandbytes
mxfp4
```



vllm-v0.17.1-u 版本

```
apiVersion: v1
kind: ModelCatalog
metadata:
  name: qwen35-27b-reasoning-distilled
  display_name: Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
  labels:
    icon_url: https://cdn-thumbnails.huggingface.co
    hf_repo_url: https://huggingface.co
spec:
  model:
    registry: 'afs-neutree'
    name: jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
    version: ''
    #task: text-generation
  engine:
    engine: vllm
    version: v0.17.1-u
  resources: {}
  replicas:
    num: 1
  deployment_options:
    scheduler:
      type: consistent_hash
      virtual_nodes: 150
      load_factor: 1.25
  variables:
    RAY_SCHEDULER_TYPE: consistent_hash
    engine_args:
      tensor_parallel_size: 1
      max_model_len: 2048
      gpu_memory_utilization: 0.85
      enable_chunked_prefill: true
      #--no-enable-chunked-prefill
      served_model_name: jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
      enforce_eager:
      tokenizer_mode: "auto"
      kv-cache-dtype: "fp8"
      dtype: "float16"

```

#RD 提供

```
apiVersion: v1
kind: ModelCatalog
metadata:
  name: qwen35-27b-reasoning-distilled
  display_name: Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
  labels:
    icon_url: https://cdn-thumbnails.huggingface.co
    hf_repo_url: https://huggingface.co
spec:
  model:
    registry: 'afs-neutree'
    name: jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
    version: ''
    #task: text-generation
  engine:
    engine: vllm
    version: v0.17.1-u
  resources: {}
  replicas:
    num: 1
  deployment_options:
    scheduler:
      type: consistent_hash
      virtual_nodes: 150
      load_factor: 1.25
  variables:
    RAY_SCHEDULER_TYPE: consistent_hash
    engine_args:
      tensor_parallel_size: 1
      max_model_len: 32768
      gpu_memory_utilization: 0.92
      enable_chunked_prefill: true
      served_model_name: jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
      tokenizer_mode: "auto"
      quantization: "fp8"
```

#加入 kv cache offload -> 用一些 記憶體來當作 cache，4bit 不管怎麼調整 24GB GPU 跑不起來，模型太大
配置 8 core , 32 GB RAM , 24GB GPU , bitsandbytes (4bit)

```
apiVersion: v1
kind: ModelCatalog
metadata:
  name: extrahop-27b-reasoning-distilled-4bit
  display_name: Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
  labels:
    icon_url: https://cdn-thumbnails.huggingface.co
    hf_repo_url: https://huggingface.co
spec:
  model:
    registry: 'afs-neutree'
    name: jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
    version: ''
  engine:
    engine: vllm
    version: v0.17.1-u
  resources: {}
  replicas:
    num: 1
  deployment_options:
    scheduler:
      type: consistent_hash
      virtual_nodes: 150
      load_factor: 1.25
  variables:
    RAY_SCHEDULER_TYPE: consistent_hash
    engine_args:
      tensor_parallel_size: 1
      max_model_len: 32768
      gpu_memory_utilization: 0.80
      enable_chunked_prefill: true
      #served_model_name: jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
      tokenizer_mode: "auto"
      quantization: "bitsandbytes"
      enforce_eager: ""
      #offload_backend: "prefetch"
      #offload-group-size: "4"
      #offload-num-in-group: "1"
      #offload-prefetch-step: "1"
      
      #kv-offloading-backend: "native"
      #kv-offloading-size: "16"
      #disable_hybrid_kv_cache_manager:
      
      
      #cpu-offload-gb: "10"
      
```

fp8 

配置 8 core ,  12GB RAM , 47GB GPU , fp8

```
apiVersion: v1
kind: ModelCatalog
metadata:
  name: extrahop-27b-reasoning-distilled-fp8
  display_name: Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled
  labels:
    icon_url: https://cdn-thumbnails.huggingface.co
    hf_repo_url: https://huggingface.co
spec:
  model:
    registry: 'afs-neutree'
    name: jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
    version: ''
  engine:
    engine: vllm
    version: v0.17.1-u
  resources: {}
  replicas:
    num: 1
  deployment_options:
    scheduler:
      type: consistent_hash
      virtual_nodes: 150
      load_factor: 1.25
  variables:
    RAY_SCHEDULER_TYPE: consistent_hash
    engine_args:
      tensor_parallel_size: 1
      max_model_len: 32768
      gpu_memory_utilization: 0.92
      enable_chunked_prefill: true
      tokenizer_mode: "auto"
      quantization: "fp8"
```



![image-20260317115833582](https://kenkenny.synology.me:5543/images/2026/03/image-20260317115833582.png)

![image-20260317115848034](https://kenkenny.synology.me:5543/images/2026/03/image-20260317115848034.png)

![image-20260317120527967](https://kenkenny.synology.me:5543/images/2026/03/image-20260317120527967.png)

```
kubectl get pods -n neutree-cluster-4617c098ac
NAME                                    READY   STATUS     RESTARTS   AGE
extrahop-qwen3.5-27b-75f75dc954-zwdw4   0/1     Init:0/1   0          2m21s
router-754bdfd4dd-h2wqm                 1/1     Running    0          5d20h
vmagent-78cfb7cfdc-42phm                1/1     Running    0          5d20h
ken@ken-ubuntu-gpu:/mnt$ kubectl logs -f extrahop-qwen3.5-27b-75f75dc954-zwdw4 -n neutree-cluster-4617c098ac
Defaulted container "vllm" out of: vllm, model-downloader (init)
Error from server (BadRequest): container "vllm" in pod "extrahop-qwen3.5-27b-75f75dc954-zwdw4" is waiting to start: PodInitializing
ken@ken-ubuntu-gpu:/mnt$ kubectl describe pod extrahop-qwen3.5-27b-75f75dc954-zwdw4 -n neutree-cluster-4617c098ac
Name:             extrahop-qwen3.5-27b-75f75dc954-zwdw4
Namespace:        neutree-cluster-4617c098ac
Priority:         0
Service Account:  default
Node:             ake-aic-gpu-workergroup-gpu-2xhsq-46pn4/192.168.102.14
Start Time:       Tue, 17 Mar 2026 03:58:53 +0000
Labels:           app=inference
                  cluster=ake-aic-gpu
                  endpoint=extrahop-qwen3.5-27b
                  engine=vllm
                  engine_version=v0.11.2
                  pod-template-hash=75f75dc954
                  routing_logic=consistent_hash
                  workspace=ake-aic-gpu
Annotations:      k8tz.io/injected: true
                  k8tz.io/timezone: Asia/Taipei
Status:           Pending
IP:               10.244.2.125
IPs:
  IP:           10.244.2.125
Controlled By:  ReplicaSet/extrahop-qwen3.5-27b-75f75dc954
Init Containers:
  model-downloader:
    Container ID:  containerd://4f33cabe1673d6c73cec7c3035521f1c1dcc50af1748c5529f2e4c9dd8d77e10
    Image:         192.168.122.11/neutreeai/neutree/neutree-runtime:v1.0.0
    Image ID:      192.168.122.11/neutreeai/neutree/neutree-runtime@sha256:fd5e90d1891eaca4132e01d007a702bdae96127f595d3fd227ea68994ae10a92
    Port:          <none>
    Host Port:     <none>
    Command:
      bash
      -c
    Args:
      python3 -m neutree.downloader --name="jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled" --registry_type="bentoml" --registry_path="/mnt/bentoml/models/jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled/n4o445rbwojggusu" --path="/models-cache/default/jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled/n4o445rbwojggusu" --version="n4o445rbwojggusu" --file="" --task="text-generation"
    State:          Running
      Started:      Tue, 17 Mar 2026 03:58:54 +0000
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /dev/shm from dshm (rw)
      /mnt/bentoml from bentoml-model-registry (rw)
      /models-cache from models-cache-tmp (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-k95hg (ro)
Containers:
  vllm:
    Container ID:  
    Image:         192.168.122.11/neutreeai/vllm/vllm-openai:v0.11.2
    Image ID:      
    Port:          8000/TCP
    Host Port:     0/TCP
    Command:
      vllm
      serve
      /models-cache/default/jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled/n4o445rbwojggusu
      --host
      0.0.0.0
      --port
      8000
      --served-model-name
      jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
      --task
      generate
      --enable_chunked_prefill
      --enforce_eager
      --gpu_memory_utilization
      0.9
      --max_model_len
      4096
      --quantization
      awq
      --served_model_name
      jackrong_qwen3.5-27b-claude-4.6-opus-reasoning-distilled
      --tensor_parallel_size
      1
      --trust_remote_code
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:             12
      memory:          40Gi
      nvidia.com/gpu:  1
    Requests:
      cpu:             12
      memory:          40Gi
      nvidia.com/gpu:  1
    Readiness:         http-get http://:8000/health delay=5s timeout=5s period=10s #success=1 #failure=3
    Startup:           http-get http://:8000/health delay=5s timeout=5s period=10s #success=1 #failure=120
    Environment:
      TZ:  Asia/Taipei
    Mounts:
      /dev/shm from dshm (rw)
      /etc/localtime from k8tz (ro,path="Asia/Taipei")
      /mnt/bentoml from bentoml-model-registry (rw)
      /models-cache from models-cache-tmp (rw)
      /usr/share/zoneinfo from k8tz (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-k95hg (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 False 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  models-cache-tmp:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  bentoml-model-registry:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    192.168.102.36
    Path:      /neutree
    ReadOnly:  false
  dshm:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  kube-api-access-k95hg:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
  k8tz:
    Type:          HostPath (bare host directory volume)
    Path:          /usr/share/zoneinfo
    HostPathType:  
QoS Class:         Burstable
Node-Selectors:    nvidia.com/gpu.product=NVIDIA-H100-NVL-MIG-1g.24gb
Tolerations:       node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                   node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m40s  default-scheduler  Successfully assigned neutree-cluster-4617c098ac/extrahop-qwen3.5-27b-75f75dc954-zwdw4 to ake-aic-gpu-workergroup-gpu-2xhsq-46pn4
  Normal  Pulled     2m39s  kubelet            spec.initContainers{model-downloader}: Container image "192.168.122.11/neutreeai/neutree/neutree-runtime:v1.0.0" already present on machine
  Normal  Created    2m39s  kubelet            spec.initContainers{model-downloader}: Created container: model-downloader
  Normal  Started    2m39s  kubelet            spec.initContainers{model-downloader}: Started container model-downloader
```

vllm 版本太舊

![image-20260317130905127](https://kenkenny.synology.me:5543/images/2026/03/image-20260317130905127.png)

更新後查看 log 

![image-20260318121031280](https://kenkenny.synology.me:5543/images/2026/03/image-20260318121031280.png)





#### 更新 vllm 版本

```
$ docker pull vllm/vllm-openai:v0.17.1
# Home 目錄
$ docker save vllm/vllm-openai:v0.17.1 | gzip > vllm-vllm-v0.17.1.tar

# 做好目錄跟 manifest.yaml 並打包
$ tree neutree-openai-vllm-v0.17.1/
neutree-openai-vllm-v0.17.1/
├── images
│   └── vllm-vllm-openai-v0.17.1.tar
└── manifest.yaml

2 directories, 2 files

$ cd /home/ken/neutree-openai-vllm-v0.17.1
$ tar -czvf /home/ken/neutree-openai-vllm-v0.17.1.tar.gz manifest.yaml images/
$ cd /home/ken
$ ls -l neutree-openai-vllm-v0.17.1.tar.gz
-rw-rw-r-- 1 ken ken 9220853350 Mar 18 03:16 neutree-openai-vllm-v0.17.1.tar.gz

$ neutree-cli-amd64 import engine  \
    --package /home/ken/neutree-openai-vllm-v0.17.1.tar.gz \
    --mirror-registry ake-registry.arcfralab.local/neutreeai \
    --registry-username ken \
    --registry-password P@ssw0rd123 \
    --api-key sk_bb1w \
    --server-url http://192.168.102.21:3000
    
# 指定 workspace
$ neutree-cli-amd64 import engine  \
    --package /home/ken/neutree-openai-vllm-v0.17.1.tar.gz \
    --mirror-registry ake-registry.arcfralab.local/neutreeai \
    --registry-username ken \
    --registry-password P@ssw0rd123 \
    --api-key sk_bxxxxxxxxxxxx \
    --workspace ake-aic-gpu \
    --server-url http://192.168.102.21:3000
    
    
# Arthur workspace
neutree-cli-amd64 import engine  \
--package /home/ken/vllm-gemma4.tar.gz \
--mirror-registry registry01.arcfralab.local \
--registry-project neutree-ai \
--registry-username admin \
--registry-password Arcfra@80670781 \
--api-key "sk_oKJxxxxxxxxxxxxxxobA" \
--server-url "http://192.168.102.21:3000" \
--workspace "arthur-ai-agent"

# delete 
neutree-cli-amd64 engine remove-version \
    --name vllm \
    --version gemma4 \
    --api-key sk_oKXjx3qBVN1dkNxxxxxxxxxxxxxxxxxxxxxxxxxn9JumXNTnJobA \
    --workspace arthur-ai-agent \
    --server-url http://192.168.102.21:3000

```

for  vllm/vllm-openai:gemma4，gemma4 模型建議

https://github.com/vllm-project/vllm/releases

- **Gemma 4 support**: Full Google Gemma 4 architecture support including MoE, multimodal, reasoning, and tool-use capabilities ([#38826](https://github.com/vllm-project/vllm/pull/38826), [#38847](https://github.com/vllm-project/vllm/pull/38847)). Requires `transformers>=5.5.0`. We recommend using pre-built docker image `vllm/vllm-openai:gemma4` for out of box usage.

```
$ docker pull vllm/vllm-openai:gemma4
$ docker save vllm/vllm-openai:gemma4 | gzip > vllm-vllm-vgemma4.tar
$ tree vllm-gemma4/
vllm-gemma4/
├── images
│   └── vllm-vllm-gemma4.tar
└── manifest.yaml

2 directories, 2 files
$ tar -czvf /home/ken/vllm-gemma4.tar.gz manifest.yaml images/
$ neutree-cli-amd64 import engine  \
    --package /home/ken/vllm-gemma4.tar.gz \
    --mirror-registry ake-registry.arcfralab.local/neutreeai \
    --registry-username ken \
    --registry-password P@ssw0rd123 \
    --api-key sk_bbN1hG1LKwGzJICa80NjQumgsib1w \
    --workspace ake-aic-gpu \
    --server-url http://192.168.102.21:3000

# 刪除 engine
$ neutree-cli-amd64 engine remove-version \
    --name vllm \
    --version gemma4 \
    --api-key sk_bbN1hGJICa80NjQumgsib1w \
    --workspace ake-aic-gpu \
    --server-url http://192.168.102.21:3000
    
    

```

![image-20260408144648715](https://kenkenny.synology.me:5543/images/2026/04/image-20260408144648715.png)

![image-20260408144715264](https://kenkenny.synology.me:5543/images/2026/04/image-20260408144715264.png)



![image-20260318100644546](https://kenkenny.synology.me:5543/images/2026/03/image-20260318100644546.png)

![image-20260318112451817](https://kenkenny.synology.me:5543/images/2026/03/image-20260318112451817.png)

![image-20260318112413486](https://kenkenny.synology.me:5543/images/2026/03/image-20260318112413486.png)

![image-20260318113040976](https://kenkenny.synology.me:5543/images/2026/03/image-20260318113040976.png)

![image-20260318113323575](https://kenkenny.synology.me:5543/images/2026/03/image-20260318113323575.png)

![image-20260318113355665](https://kenkenny.synology.me:5543/images/2026/03/image-20260318113355665.png)

![image-20260318114328513](https://kenkenny.synology.me:5543/images/2026/03/image-20260318114328513.png)

##### 更新 transfomer

```
pwd
/home/ken/vllm-v0.17.1
ken@ken-ubuntu-gpu:~/vllm-v0.17.1$ ll
total 16
drwxrwxr-x  2 ken ken 4096 Mar 25 05:56 ./
drwxr-x--- 16 ken ken 4096 Mar 25 05:53 ../
-rw-rw-r--  1 ken ken  145 Mar 25 05:53 Dockerfile

Dockerfile
---
cat Dockerfile 
FROM vllm/vllm-openai:v0.17.1

RUN pip install --no-cache-dir --upgrade \
    transformers \
    tokenizers \
    accelerate \
    sentencepiece
---
$ docker build -t vllm/vllm-openai:v0.17.1-u .

[+] Building 21.2s (6/6) FINISHED                                                                               docker:default
 => [internal] load build definition from Dockerfile                                                                      0.0s
 => => transferring dockerfile: 184B                                                                                      0.0s
 => [internal] load metadata for docker.io/vllm/vllm-openai:v0.17.1                                                       0.0s
 => [internal] load .dockerignore                                                                                         0.0s
 => => transferring context: 2B                                                                                           0.0s
 => [1/2] FROM docker.io/vllm/vllm-openai:v0.17.1@sha256:0dc46f74eb0e630675d83101dc66c6441c4475cceedcf9235ee42b87c3affd2  0.3s
 => => resolve docker.io/vllm/vllm-openai:v0.17.1@sha256:0dc46f74eb0e630675d83101dc66c6441c4475cceedcf9235ee42b87c3affd2  0.0s
 => [2/2] RUN pip install --no-cache-dir --upgrade     transformers     tokenizers     accelerate     sentencepiece      15.3s
 => exporting to image                                                                                                    5.3s 
 => => exporting layers                                                                                                   4.2s 
 => => exporting manifest sha256:a17bf687d99bfb37475a035a3aeddb142ed7441025f773076414d6b191cdbda7                         0.0s 
 => => exporting config sha256:851e1f9c17416a7839a04febf261ecaa76e11dcb4d939b73c8d02d62e41aae8e                           0.0s 
 => => exporting attestation manifest sha256:ce9eaf80d6474bc994da45f9cad5f8dab6ba2967b6df449214b994c63f2a2060             0.0s 
 => => exporting manifest list sha256:5acc11145107ba38545920fcb5ee1c06fe9aa5576a49c998aed05ffcca11979a                    0.0s 
 => => naming to docker.io/vllm/vllm-openai:v0.17.1-u                                                                       
 => => unpacking to docker.io/vllm/vllm-openai:v0.17.1-u 

---

$ docker save vllm/vllm-openai:v0.17.1-u | gzip > vllm-vllm-v0.17.1-u.tar

---
# 放到同目錄，並且修改 manifest.yaml
$ mkdir neutree-openai-vllm-v0.17.1-u/images
$ mv vllm-vllm-v0.17.1-u.tar neutree-openai-vllm-v0.17.1-u
$ cp manifest.yaml neutree-openai-vllm-v0.17.1-u 
$ tree neutree-openai-vllm-v0.17.1-u

tree  neutree-openai-vllm-v0.17.1-u
neutree-openai-vllm-v0.17.1-u
├── images
│   └── vllm-vllm-openai-v0.17.1-u.tar
└── manifest.yaml

2 directories, 2 files
---
# 打包
$ cd /home/ken/neutree-openai-vllm-v0.17.1-u
$ tar -czvf /home/ken/neutree-openai-vllm-v0.17.1-u.tar.gz manifest.yaml images/
---
# 重新上傳，指定 workspace
$ neutree-cli-amd64 import engine  \
    --package /home/ken/neutree-openai-vllm-v0.17.1-u.tar.gz \
    --mirror-registry ake-registry.arcfralab.local/neutreeai \
    --registry-username ken \
    --registry-password P@ssw0rd123 \
    --api-key sk_bbN1hGwGzJICa80NjQumgsib1w \
    --workspace ake-aic-gpu \
    --server-url http://192.168.102.21:3000
```

![image-20260325143729924](https://kenkenny.synology.me:5543/images/2026/03/image-20260325143729924.png)

![image-20260325144000221](https://kenkenny.synology.me:5543/images/2026/03/image-20260325144000221.png)

![image-20260325144027270](https://kenkenny.synology.me:5543/images/2026/03/image-20260325144027270.png)



#### v0.11.2 manifest.yaml

```
/vllm-v0.11.2$ ll
total 13594468
drwxrwxr-x  3 ken ken        4096 Mar 18 02:34 ./
drwxr-x--- 14 ken ken        4096 Mar 18 02:30 ../
drwxr-xr-x  2 ken ken        4096 Jan 30 03:44 images/
-rw-r--r--  1 ken ken       40472 Jan 30 03:44 manifest.yaml
-rw-rw-r--  1 ken ken 13920677351 Mar 18 02:30 vllm-v0.11.2.tar.gz
ken@ken-ubuntu-gpu:~/vllm-v0.11.2$ cat manifest.yaml 
manifest_version: "1.0"

metadata:
  description: "vLLM"
  author: "Neutree Team"
  created_at: "2026-01-30T03:44:44Z"
  version: v1.0.0-rc.4-1-gb078220
  tags:
    - "engine"
    - "vllm"
    - "v0.11.2"

images:
    - accelerator: "nvidia_gpu"
      image_name: "vllm/vllm-openai"
      tag: "v0.11.2"
      image_file: "images/vllm-vllm-openai-v0.11.2.tar"
      platform: "linux/amd64"
      size: 28893889024

engines:
- name: vllm
  engine_versions:
  - version: "v0.11.2"

    values_schema:
      values_schema_base64: "ewogICIkc2NoZW1hIjogImh0dHA6Ly9qc29uLXNjaGVtYS5vcmcvZHJhZnQtMDcvc2NoZW1hIyIsCiAgInR5cGUiOiAib2JqZWN0IiwKICAidGl0bGUiOiAidkxMTSB2MC4xMS4yIEVuZ2luZSBDb25maWd1cmF0aW9uIiwKICAiZGVzY3JpcHRpb24iOiAiQ29uZmlndXJhdGlvbiBzY2hlbWEgZm9yIHZMTE0gdjAuMTEuMiBlbmdpbmUgcGFyYW1ldGVycyIsCiAgInByb3BlcnRpZXMiOiB7CiAgICAibW9kZWwiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOYW1lIG9yIHBhdGggb2YgdGhlIEh1Z2dpbmcgRmFjZSBtb2RlbCB0byB1c2UiCiAgICB9LAogICAgInJ1bm5lciI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImF1dG8iLCAiZHJhZnQiLCAiZ2VuZXJhdGUiLCAicG9vbGluZyJdLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSB0eXBlIG9mIG1vZGVsIHJ1bm5lciB0byB1c2UiCiAgICB9LAogICAgImNvbnZlcnQiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgImNsYXNzaWZ5IiwgImVtYmVkIiwgIm5vbmUiLCAicmV3YXJkIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiQ29udmVydCB0aGUgbW9kZWwgdXNpbmcgYWRhcHRlcnMiCiAgICB9LAogICAgInRva2VuaXplciI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk5hbWUgb3IgcGF0aCBvZiB0aGUgSHVnZ2luZyBGYWNlIHRva2VuaXplciB0byB1c2UiCiAgICB9LAogICAgInRva2VuaXplcl9tb2RlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJzbG93IiwgIm1pc3RyYWwiLCAiY3VzdG9tIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIHRva2VuaXplciBtb2RlIHRvIHVzZSIKICAgIH0sCiAgICAidHJ1c3RfcmVtb3RlX2NvZGUiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVHJ1c3QgcmVtb3RlIGNvZGUgZnJvbSBIdWdnaW5nIEZhY2UiCiAgICB9LAogICAgImR0eXBlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJoYWxmIiwgImZsb2F0MTYiLCAiYmZsb2F0MTYiLCAiZmxvYXQiLCAiZmxvYXQzMiJdLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkRhdGEgdHlwZSBmb3IgbW9kZWwgd2VpZ2h0cyBhbmQgYWN0aXZhdGlvbnMiCiAgICB9LAogICAgInNlZWQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDAsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJSYW5kb20gc2VlZCBmb3IgcmVwcm9kdWNpYmlsaXR5IgogICAgfSwKICAgICJoZl9jb25maWdfcGF0aCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk5hbWUgb3IgcGF0aCBvZiB0aGUgSHVnZ2luZyBGYWNlIGNvbmZpZyB0byB1c2UiCiAgICB9LAogICAgImFsbG93ZWRfbG9jYWxfbWVkaWFfcGF0aCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlZmF1bHQiOiAiIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkFsbG93ZWQgbG9jYWwgbWVkaWEgcGF0aCBmb3Igc2VjdXJpdHkiCiAgICB9LAogICAgImFsbG93ZWRfbWVkaWFfZG9tYWlucyI6IHsKICAgICAgInR5cGUiOiAiYXJyYXkiLAogICAgICAiaXRlbXMiOiB7CiAgICAgICAgInR5cGUiOiAic3RyaW5nIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiQWxsb3dlZCBtZWRpYSBkb21haW5zIGZvciBtdWx0aS1tb2RhbCBpbnB1dHMiCiAgICB9LAogICAgInJldmlzaW9uIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIHNwZWNpZmljIG1vZGVsIHZlcnNpb24gdG8gdXNlIgogICAgfSwKICAgICJjb2RlX3JldmlzaW9uIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIHNwZWNpZmljIHJldmlzaW9uIHRvIHVzZSBmb3IgdGhlIG1vZGVsIGNvZGUiCiAgICB9LAogICAgInRva2VuaXplcl9yZXZpc2lvbiI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlJldmlzaW9uIG9mIHRoZSB0b2tlbml6ZXIgdG8gdXNlIgogICAgfSwKICAgICJtYXhfbW9kZWxfbGVuIjogewogICAgICAidHlwZSI6IFsiaW50ZWdlciIsICJzdHJpbmciXSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1vZGVsIGNvbnRleHQgbGVuZ3RoLiBTdXBwb3J0cyBodW1hbi1yZWFkYWJsZSBmb3JtYXQgbGlrZSAnMWsnLCAnMk0nIgogICAgfSwKICAgICJxdWFudGl6YXRpb24iOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNZXRob2QgdXNlZCB0byBxdWFudGl6ZSB0aGUgd2VpZ2h0cyIKICAgIH0sCiAgICAiZW5mb3JjZV9lYWdlciI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBbHdheXMgdXNlIGVhZ2VyLW1vZGUgUHlUb3JjaCIKICAgIH0sCiAgICAibWF4X2xvZ3Byb2JzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAyMCwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIGxvZyBwcm9iYWJpbGl0aWVzIHRvIHJldHVybiIKICAgIH0sCiAgICAibG9ncHJvYnNfbW9kZSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbInByb2Nlc3NlZF9sb2dpdHMiLCAicHJvY2Vzc2VkX2xvZ3Byb2JzIiwgInJhd19sb2dpdHMiLCAicmF3X2xvZ3Byb2JzIl0sCiAgICAgICJkZWZhdWx0IjogInJhd19sb2dwcm9icyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJJbmRpY2F0ZXMgdGhlIGNvbnRlbnQgcmV0dXJuZWQgaW4gbG9ncHJvYnMiCiAgICB9LAogICAgImRpc2FibGVfc2xpZGluZ193aW5kb3ciOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzYWJsZSBzbGlkaW5nIHdpbmRvdyBhdHRlbnRpb24iCiAgICB9LAogICAgImRpc2FibGVfY2FzY2FkZV9hdHRuIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgY2FzY2FkZSBhdHRlbnRpb24gZm9yIFYxIgogICAgfSwKICAgICJza2lwX3Rva2VuaXplcl9pbml0IjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlNraXAgaW5pdGlhbGl6YXRpb24gb2YgdG9rZW5pemVyIGFuZCBkZXRva2VuaXplciIKICAgIH0sCiAgICAiZW5hYmxlX3Byb21wdF9lbWJlZHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHBhc3NpbmcgdGV4dCBlbWJlZGRpbmdzIHZpYSBwcm9tcHRfZW1iZWRzIgogICAgfSwKICAgICJzZXJ2ZWRfbW9kZWxfbmFtZSI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJhcnJheSJdLAogICAgICAiaXRlbXMiOiB7CiAgICAgICAgInR5cGUiOiAic3RyaW5nIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIG1vZGVsIG5hbWUocykgdXNlZCBpbiB0aGUgQVBJIgogICAgfSwKICAgICJjb25maWdfZm9ybWF0IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJoZiIsICJtaXN0cmFsIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIGZvcm1hdCBvZiB0aGUgbW9kZWwgY29uZmlnIHRvIGxvYWQiCiAgICB9LAogICAgImhmX3Rva2VuIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgImJvb2xlYW4iXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkh1Z2dpbmcgRmFjZSB0b2tlbiBmb3IgYXV0aGVudGljYXRpb24iCiAgICB9LAogICAgImhmX292ZXJyaWRlcyI6IHsKICAgICAgInR5cGUiOiAib2JqZWN0IiwKICAgICAgImRlZmF1bHQiOiB7fSwKICAgICAgImRlc2NyaXB0aW9uIjogIkFyZ3VtZW50cyB0byBiZSBmb3J3YXJkZWQgdG8gSHVnZ2luZ0ZhY2UgY29uZmlnIgogICAgfSwKICAgICJwb29sZXJfY29uZmlnIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgIm9iamVjdCJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiUG9vbGVyIGNvbmZpZyBmb3Igb3V0cHV0IHBvb2xpbmcgYmVoYXZpb3IiCiAgICB9LAogICAgImxvZ2l0c19wcm9jZXNzb3JfcGF0dGVybiI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlJlZ2V4IHBhdHRlcm4gZm9yIHZhbGlkIGxvZ2l0cyBwcm9jZXNzb3IgbmFtZXMiCiAgICB9LAogICAgImdlbmVyYXRpb25fY29uZmlnIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlBhdGggdG8gZ2VuZXJhdGlvbiBjb25maWciCiAgICB9LAogICAgIm92ZXJyaWRlX2dlbmVyYXRpb25fY29uZmlnIjogewogICAgICAidHlwZSI6ICJvYmplY3QiLAogICAgICAiZGVmYXVsdCI6IHt9LAogICAgICAiZGVzY3JpcHRpb24iOiAiT3ZlcnJpZGUgZ2VuZXJhdGlvbiBjb25maWcgcGFyYW1ldGVycyIKICAgIH0sCiAgICAiZW5hYmxlX3NsZWVwX21vZGUiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHNsZWVwIG1vZGUgZm9yIHRoZSBlbmdpbmUiCiAgICB9LAogICAgIm1vZGVsX2ltcGwiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgInRlcnJhdG9yY2giLCAidHJhbnNmb3JtZXJzIiwgInZsbG0iXSwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJXaGljaCBpbXBsZW1lbnRhdGlvbiBvZiB0aGUgbW9kZWwgdG8gdXNlIgogICAgfSwKICAgICJvdmVycmlkZV9hdHRlbnRpb25fZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJPdmVycmlkZSBkdHlwZSBmb3IgYXR0ZW50aW9uIgogICAgfSwKICAgICJsb2dpdHNfcHJvY2Vzc29ycyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJhcnJheSJdLAogICAgICAiaXRlbXMiOiB7CiAgICAgICAgInR5cGUiOiAic3RyaW5nIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiTG9naXRzIHByb2Nlc3NvcnMgY2xhc3MgbmFtZXMiCiAgICB9LAogICAgImlvX3Byb2Nlc3Nvcl9wbHVnaW4iOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJJT1Byb2Nlc3NvciBwbHVnaW4gbmFtZSB0byBsb2FkIGF0IHN0YXJ0dXAiCiAgICB9LAogICAgImxvYWRfZm9ybWF0IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSBmb3JtYXQgb2YgbW9kZWwgd2VpZ2h0cyB0byBsb2FkIgogICAgfSwKICAgICJkb3dubG9hZF9kaXIiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEaXJlY3RvcnkgdG8gZG93bmxvYWQgYW5kIGxvYWQgd2VpZ2h0cyIKICAgIH0sCiAgICAic2FmZXRlbnNvcnNfbG9hZF9zdHJhdGVneSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImxhenkiLCAiZWFnZXIiLCAidG9yY2hhbyJdLAogICAgICAiZGVmYXVsdCI6ICJsYXp5IiwKICAgICAgImRlc2NyaXB0aW9uIjogIkxvYWRpbmcgc3RyYXRlZ3kgZm9yIHNhZmV0ZW5zb3JzIHdlaWdodHMiCiAgICB9LAogICAgIm1vZGVsX2xvYWRlcl9leHRyYV9jb25maWciOiB7CiAgICAgICJ0eXBlIjogIm9iamVjdCIsCiAgICAgICJkZWZhdWx0Ijoge30sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFeHRyYSBjb25maWcgZm9yIG1vZGVsIGxvYWRlciIKICAgIH0sCiAgICAiaWdub3JlX3BhdHRlcm5zIjogewogICAgICAidHlwZSI6ICJhcnJheSIsCiAgICAgICJpdGVtcyI6IHsKICAgICAgICAidHlwZSI6ICJzdHJpbmciCiAgICAgIH0sCiAgICAgICJkZWZhdWx0IjogWyJvcmlnaW5hbC8qKi8qIl0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQYXR0ZXJucyB0byBpZ25vcmUgd2hlbiBsb2FkaW5nIG1vZGVsIgogICAgfSwKICAgICJ1c2VfdHFkbV9vbl9sb2FkIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiB0cnVlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHRxZG0gcHJvZ3Jlc3MgYmFyIHdoZW4gbG9hZGluZyIKICAgIH0sCiAgICAicHRfbG9hZF9tYXBfbG9jYXRpb24iOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZWZhdWx0IjogImNwdSIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNYXAgbG9jYXRpb24gZm9yIGxvYWRpbmcgcHl0b3JjaCBjaGVja3BvaW50IgogICAgfSwKICAgICJyZWFzb25pbmdfcGFyc2VyIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiUmVhc29uaW5nIHBhcnNlciB0byB1c2UiCiAgICB9LAogICAgInJlYXNvbmluZ19wYXJzZXJfcGx1Z2luIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiUGF0aCB0byByZWFzb25pbmcgcGFyc2VyIHBsdWdpbiIKICAgIH0sCiAgICAiZGlzdHJpYnV0ZWRfZXhlY3V0b3JfYmFja2VuZCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImV4dGVybmFsX2xhdW5jaGVyIiwgIm1wIiwgInJheSIsICJ1bmkiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkJhY2tlbmQgZm9yIGRpc3RyaWJ1dGVkIG1vZGVsIHdvcmtlcnMiCiAgICB9LAogICAgInBpcGVsaW5lX3BhcmFsbGVsX3NpemUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBwaXBlbGluZSBwYXJhbGxlbCBncm91cHMiCiAgICB9LAogICAgIm1hc3Rlcl9hZGRyIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIxMjcuMC4wLjEiLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzdHJpYnV0ZWQgbWFzdGVyIGFkZHJlc3MiCiAgICB9LAogICAgIm1hc3Rlcl9wb3J0IjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAyOTUwMSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc3RyaWJ1dGVkIG1hc3RlciBwb3J0IgogICAgfSwKICAgICJubm9kZXMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBub2RlcyBmb3IgbXVsdGktbm9kZSBpbmZlcmVuY2UiCiAgICB9LAogICAgIm5vZGVfcmFuayI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJtaW5pbXVtIjogMCwKICAgICAgImRlZmF1bHQiOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzdHJpYnV0ZWQgbm9kZSByYW5rIgogICAgfSwKICAgICJ0ZW5zb3JfcGFyYWxsZWxfc2l6ZSI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJtaW5pbXVtIjogMSwKICAgICAgImRlZmF1bHQiOiAxLAogICAgICAiZGVzY3JpcHRpb24iOiAiTnVtYmVyIG9mIHRlbnNvciBwYXJhbGxlbCBncm91cHMiCiAgICB9LAogICAgImRlY29kZV9jb250ZXh0X3BhcmFsbGVsX3NpemUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBkZWNvZGUgY29udGV4dCBwYXJhbGxlbCBncm91cHMiCiAgICB9LAogICAgImRjcF9rdl9jYWNoZV9pbnRlcmxlYXZlX3NpemUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiS1YgY2FjaGUgaW50ZXJsZWF2ZSBzaXplIGZvciBkZWNvZGUgY29udGV4dCBwYXJhbGxlbGlzbSIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9zaXplIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgIm1pbmltdW0iOiAxLAogICAgICAiZGVmYXVsdCI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOdW1iZXIgb2YgZGF0YSBwYXJhbGxlbCByZXBsaWNhcyIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9yYW5rIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGF0YSBwYXJhbGxlbCByYW5rIG9mIHRoaXMgaW5zdGFuY2UiCiAgICB9LAogICAgImRhdGFfcGFyYWxsZWxfc3RhcnRfcmFuayI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJtaW5pbXVtIjogMCwKICAgICAgImRlc2NyaXB0aW9uIjogIlN0YXJ0aW5nIGRhdGEgcGFyYWxsZWwgcmFuayBmb3Igc2Vjb25kYXJ5IG5vZGVzIgogICAgfSwKICAgICJkYXRhX3BhcmFsbGVsX3NpemVfbG9jYWwiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOdW1iZXIgb2YgZGF0YSBwYXJhbGxlbCByZXBsaWNhcyBvbiB0aGlzIG5vZGUiCiAgICB9LAogICAgImRhdGFfcGFyYWxsZWxfYWRkcmVzcyI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkFkZHJlc3Mgb2YgZGF0YSBwYXJhbGxlbCBjbHVzdGVyIGhlYWQtbm9kZSIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9ycGNfcG9ydCI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQb3J0IGZvciBkYXRhIHBhcmFsbGVsIFJQQyBjb21tdW5pY2F0aW9uIgogICAgfSwKICAgICJkYXRhX3BhcmFsbGVsX2JhY2tlbmQiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJtcCIsICJyYXkiXSwKICAgICAgImRlZmF1bHQiOiAibXAiLAogICAgICAiZGVzY3JpcHRpb24iOiAiQmFja2VuZCBmb3IgZGF0YSBwYXJhbGxlbCIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9oeWJyaWRfbGIiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVXNlIGh5YnJpZCBEUCBsb2FkIGJhbGFuY2luZyBtb2RlIgogICAgfSwKICAgICJkYXRhX3BhcmFsbGVsX2V4dGVybmFsX2xiIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlVzZSBleHRlcm5hbCBEUCBsb2FkIGJhbGFuY2luZyBtb2RlIgogICAgfSwKICAgICJlbmFibGVfZXhwZXJ0X3BhcmFsbGVsIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBleHBlcnQgcGFyYWxsZWxpc20gZm9yIE1vRSBtb2RlbHMiCiAgICB9LAogICAgImFsbDJhbGxfYmFja2VuZCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImFsbGdhdGhlcl9yZWR1Y2VzY2F0dGVyIiwgImRlZXBlcF9oaWdoX3Rocm91Z2hwdXQiLCAiZGVlcGVwX2xvd19sYXRlbmN5IiwgImZsYXNoaW5mZXJfYWxsMmFsbHYiLCAibmFpdmUiLCAicHBseCIsICJOb25lIl0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBbGwyQWxsIGJhY2tlbmQgZm9yIE1vRSBleHBlcnQgcGFyYWxsZWwiCiAgICB9LAogICAgImVuYWJsZV9kYm8iOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGR1YWwgYmF0Y2ggb3ZlcmxhcCIKICAgIH0sCiAgICAiZGJvX2RlY29kZV90b2tlbl90aHJlc2hvbGQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVmYXVsdCI6IDMyLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9rZW4gdGhyZXNob2xkIGZvciBkdWFsIGJhdGNoIG92ZXJsYXAgZGVjb2RlIgogICAgfSwKICAgICJkYm9fcHJlZmlsbF90b2tlbl90aHJlc2hvbGQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVmYXVsdCI6IDUxMiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRva2VuIHRocmVzaG9sZCBmb3IgZHVhbCBiYXRjaCBvdmVybGFwIHByZWZpbGwiCiAgICB9LAogICAgImRpc2FibGVfbmNjbF9mb3JfZHBfc3luY2hyb25pemF0aW9uIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkZvcmNlIERQIHN5bmMgdG8gdXNlIEdsb28gaW5zdGVhZCBvZiBOQ0NMIgogICAgfSwKICAgICJlbmFibGVfZXBsYiI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFbmFibGUgZXhwZXJ0IHBhcmFsbGVsaXNtIGxvYWQgYmFsYW5jaW5nIgogICAgfSwKICAgICJlcGxiX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkV4cGVydCBwYXJhbGxlbGlzbSBjb25maWd1cmF0aW9uIgogICAgfSwKICAgICJleHBlcnRfcGxhY2VtZW50X3N0cmF0ZWd5IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsibGluZWFyIiwgInJvdW5kX3JvYmluIl0sCiAgICAgICJkZWZhdWx0IjogImxpbmVhciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFeHBlcnQgcGxhY2VtZW50IHN0cmF0ZWd5IGZvciBNb0UgbGF5ZXJzIgogICAgfSwKICAgICJtYXhfcGFyYWxsZWxfbG9hZGluZ193b3JrZXJzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIHBhcmFsbGVsIGxvYWRpbmcgd29ya2VycyIKICAgIH0sCiAgICAicmF5X3dvcmtlcnNfdXNlX25zaWdodCI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQcm9maWxlIFJheSB3b3JrZXJzIHdpdGggbnNpZ2h0IgogICAgfSwKICAgICJkaXNhYmxlX2N1c3RvbV9hbGxfcmVkdWNlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgY3VzdG9tIGFsbC1yZWR1Y2Uga2VybmVscyIKICAgIH0sCiAgICAid29ya2VyX2NscyI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJGdWxsIG5hbWUgb2YgdGhlIHdvcmtlciBjbGFzcyB0byB1c2UiCiAgICB9LAogICAgIndvcmtlcl9leHRlbnNpb25fY2xzIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiV29ya2VyIGV4dGVuc2lvbiBjbGFzcyBuYW1lIgogICAgfSwKICAgICJlbmFibGVfbXVsdGltb2RhbF9lbmNvZGVyX2RhdGFfcGFyYWxsZWwiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIG11bHRpbW9kYWwgZW5jb2RlciBkYXRhIHBhcmFsbGVsIgogICAgfSwKICAgICJibG9ja19zaXplIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImVudW0iOiBbMSwgOCwgMTYsIDMyLCA2NCwgMTI4LCAyNTZdLAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZSBvZiBhIGNvbnRpZ3VvdXMgY2FjaGUgYmxvY2siCiAgICB9LAogICAgImdwdV9tZW1vcnlfdXRpbGl6YXRpb24iOiB7CiAgICAgICJ0eXBlIjogIm51bWJlciIsCiAgICAgICJtaW5pbXVtIjogMCwKICAgICAgIm1heGltdW0iOiAxLjAsCiAgICAgICJkZWZhdWx0IjogMC45LAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIGZyYWN0aW9uIG9mIEdQVSBtZW1vcnkgdG8gYmUgdXNlZCIKICAgIH0sCiAgICAia3ZfY2FjaGVfbWVtb3J5X2J5dGVzIjogewogICAgICAidHlwZSI6IFsiaW50ZWdlciIsICJzdHJpbmciXSwKICAgICAgImRlc2NyaXB0aW9uIjogIlNpemUgb2YgS1YgQ2FjaGUgcGVyIEdQVSBpbiBieXRlcyIKICAgIH0sCiAgICAic3dhcF9zcGFjZSI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAiZGVmYXVsdCI6IDQsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJDUFUgc3dhcCBzcGFjZSBzaXplIChHaUIpIHBlciBHUFUiCiAgICB9LAogICAgImt2X2NhY2hlX2R0eXBlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJiZmxvYXQxNiIsICJmcDgiLCAiZnA4X2RzX21sYSIsICJmcDhfZTRtMyIsICJmcDhfZTVtMiIsICJmcDhfaW5jIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGF0YSB0eXBlIGZvciBLViBjYWNoZSBzdG9yYWdlIgogICAgfSwKICAgICJudW1fZ3B1X2Jsb2Nrc19vdmVycmlkZSI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJPdmVycmlkZSBwcm9maWxlZCBudW1fZ3B1X2Jsb2NrcyIKICAgIH0sCiAgICAiZW5hYmxlX3ByZWZpeF9jYWNoaW5nIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBwcmVmaXggY2FjaGluZyIKICAgIH0sCiAgICAicHJlZml4X2NhY2hpbmdfaGFzaF9hbGdvIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsic2hhMjU2IiwgInNoYTI1Nl9jYm9yIl0sCiAgICAgICJkZWZhdWx0IjogInNoYTI1NiIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJIYXNoIGFsZ29yaXRobSBmb3IgcHJlZml4IGNhY2hpbmciCiAgICB9LAogICAgImNwdV9vZmZsb2FkX2diIjogewogICAgICAidHlwZSI6ICJudW1iZXIiLAogICAgICAibWluaW11bSI6IDAsCiAgICAgICJkZWZhdWx0IjogMCwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSBzcGFjZSBpbiBHaUIgdG8gb2ZmbG9hZCB0byBDUFUiCiAgICB9LAogICAgImNhbGN1bGF0ZV9rdl9zY2FsZXMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGR5bmFtaWMgY2FsY3VsYXRpb24gb2Yga19zY2FsZSBhbmQgdl9zY2FsZSIKICAgIH0sCiAgICAia3Zfc2hhcmluZ19mYXN0X3ByZWZpbGwiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIEtWIHNoYXJpbmcgZmFzdCBwcmVmaWxsIG9wdGltaXphdGlvbiIKICAgIH0sCiAgICAibWFtYmFfY2FjaGVfZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgImZsb2F0MzIiXSwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEYXRhIHR5cGUgZm9yIE1hbWJhIGNhY2hlIgogICAgfSwKICAgICJtYW1iYV9zc21fY2FjaGVfZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgImZsb2F0MzIiXSwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEYXRhIHR5cGUgZm9yIE1hbWJhIFNTTSBzdGF0ZSIKICAgIH0sCiAgICAibWFtYmFfYmxvY2tfc2l6ZSI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJCbG9jayBzaXplIGZvciBtYW1iYSBjYWNoZSIKICAgIH0sCiAgICAia3Zfb2ZmbG9hZGluZ19zaXplIjogewogICAgICAidHlwZSI6ICJudW1iZXIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZSBvZiBLViBjYWNoZSBvZmZsb2FkaW5nIGJ1ZmZlciAoR2lCKSIKICAgIH0sCiAgICAia3Zfb2ZmbG9hZGluZ19iYWNrZW5kIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsibG1jYWNoZSIsICJuYXRpdmUiLCAiTm9uZSJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiQmFja2VuZCBmb3IgS1YgY2FjaGUgb2ZmbG9hZGluZyIKICAgIH0sCiAgICAibGltaXRfbW1fcGVyX3Byb21wdCI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlZmF1bHQiOiB7fSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIG11bHRpLW1vZGFsIGl0ZW1zIHBlciBwcm9tcHQiCiAgICB9LAogICAgImVuYWJsZV9tbV9lbWJlZHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHBhc3NpbmcgbXVsdGltb2RhbCBlbWJlZGRpbmdzIgogICAgfSwKICAgICJtZWRpYV9pb19rd2FyZ3MiOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZWZhdWx0Ijoge30sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBZGRpdGlvbmFsIGFyZ3MgZm9yIHByb2Nlc3NpbmcgbWVkaWEgaW5wdXRzIgogICAgfSwKICAgICJtbV9wcm9jZXNzb3Jfa3dhcmdzIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgIm9iamVjdCJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiQXJndW1lbnRzIGZvcndhcmRlZCB0byBtdWx0aS1tb2RhbCBwcm9jZXNzb3IiCiAgICB9LAogICAgIm1tX3Byb2Nlc3Nvcl9jYWNoZV9nYiI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgImRlZmF1bHQiOiA0LAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZSAoR2lCKSBvZiBtdWx0aS1tb2RhbCBwcm9jZXNzb3IgY2FjaGUiCiAgICB9LAogICAgImRpc2FibGVfbW1fcHJlcHJvY2Vzc29yX2NhY2hlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgbXVsdGktbW9kYWwgcHJlcHJvY2Vzc29yIGNhY2hlIgogICAgfSwKICAgICJtbV9wcm9jZXNzb3JfY2FjaGVfdHlwZSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImxydSIsICJzaG0iXSwKICAgICAgImRlZmF1bHQiOiAibHJ1IiwKICAgICAgImRlc2NyaXB0aW9uIjogIlR5cGUgb2YgY2FjaGUgZm9yIG11bHRpLW1vZGFsIHByb2Nlc3NvciIKICAgIH0sCiAgICAibW1fc2htX2NhY2hlX21heF9vYmplY3Rfc2l6ZV9tYiI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZWZhdWx0IjogMTI4LAogICAgICAiZGVzY3JpcHRpb24iOiAiTWF4IG9iamVjdCBzaXplIChNaUIpIGluIHNoYXJlZCBtZW1vcnkgY2FjaGUiCiAgICB9LAogICAgIm1tX2VuY29kZXJfdHBfbW9kZSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImRhdGEiLCAid2VpZ2h0cyJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiTXVsdGktbW9kYWwgZW5jb2RlciB0ZW5zb3IgcGFyYWxsZWwgbW9kZSIKICAgIH0sCiAgICAibW1fZW5jb2Rlcl9hdHRuX2JhY2tlbmQiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNdWx0aS1tb2RhbCBlbmNvZGVyIGF0dGVudGlvbiBiYWNrZW5kIgogICAgfSwKICAgICJpbnRlcmxlYXZlX21tX3N0cmluZ3MiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGZ1bGx5IGludGVybGVhdmVkIG11bHRpbW9kYWwgcHJvbXB0cyIKICAgIH0sCiAgICAic2tpcF9tbV9wcm9maWxpbmciOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiU2tpcCBtdWx0aW1vZGFsIG1lbW9yeSBwcm9maWxpbmciCiAgICB9LAogICAgInZpZGVvX3BydW5pbmdfcmF0ZSI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAibWF4aW11bSI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQcnVuaW5nIHJhdGUgZm9yIHZpZGVvIHZpYSBFZmZpY2llbnQgVmlkZW8gU2FtcGxpbmciCiAgICB9LAogICAgImVuYWJsZV9sb3JhIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBoYW5kbGluZyBvZiBMb1JBIGFkYXB0ZXJzIgogICAgfSwKICAgICJtYXhfbG9yYXMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIExvUkEgYWRhcHRlcnMiCiAgICB9LAogICAgIm1heF9sb3JhX3JhbmsiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZW51bSI6IFsxLCA4LCAxNiwgMzIsIDY0LCAxMjgsIDI1NiwgMzIwLCA1MTJdLAogICAgICAiZGVmYXVsdCI6IDE2LAogICAgICAiZGVzY3JpcHRpb24iOiAiTWF4aW11bSBMb1JBIHJhbmsiCiAgICB9LAogICAgImxvcmFfZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGF0YSB0eXBlIGZvciBMb1JBIgogICAgfSwKICAgICJtYXhfY3B1X2xvcmFzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIExvUkFzIHRvIHN0b3JlIGluIENQVSBtZW1vcnkiCiAgICB9LAogICAgImZ1bGx5X3NoYXJkZWRfbG9yYXMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVXNlIGZ1bGx5IHNoYXJkZWQgTG9SQSBsYXllcnMiCiAgICB9LAogICAgImRlZmF1bHRfbW1fbG9yYXMiOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEZWZhdWx0IExvUkEgcGF0aHMgZm9yIHNwZWNpZmljIG1vZGFsaXRpZXMiCiAgICB9LAogICAgInNob3dfaGlkZGVuX21ldHJpY3NfZm9yX3ZlcnNpb24iOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJTaG93IGhpZGRlbiBtZXRyaWNzIHNpbmNlIHNwZWNpZmllZCB2ZXJzaW9uIgogICAgfSwKICAgICJvdGxwX3RyYWNlc19lbmRwb2ludCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk9wZW5UZWxlbWV0cnkgdHJhY2VzIHRhcmdldCBVUkwiCiAgICB9LAogICAgImNvbGxlY3RfZGV0YWlsZWRfdHJhY2VzIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYWxsIiwgIm1vZGVsIiwgIndvcmtlciIsICJOb25lIiwgIm1vZGVsLHdvcmtlciIsICJtb2RlbCxhbGwiLCAid29ya2VyLG1vZGVsIiwgIndvcmtlcixhbGwiLCAiYWxsLG1vZGVsIiwgImFsbCx3b3JrZXIiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkNvbGxlY3QgZGV0YWlsZWQgdHJhY2VzIGZvciBzcGVjaWZpZWQgbW9kdWxlcyIKICAgIH0sCiAgICAibWF4X251bV9iYXRjaGVkX3Rva2VucyI6IHsKICAgICAgInR5cGUiOiBbImludGVnZXIiLCAic3RyaW5nIl0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNYXhpbXVtIG51bWJlciBvZiBiYXRjaGVkIHRva2VucyBwZXIgaXRlcmF0aW9uIgogICAgfSwKICAgICJtYXhfbnVtX3NlcXMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiTWF4aW11bSBudW1iZXIgb2Ygc2VxdWVuY2VzIHBlciBpdGVyYXRpb24iCiAgICB9LAogICAgIm1heF9udW1fcGFydGlhbF9wcmVmaWxscyI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heCBjb25jdXJyZW50IHBhcnRpYWwgcHJlZmlsbHMgZm9yIGNodW5rZWQgcHJlZmlsbCIKICAgIH0sCiAgICAibWF4X2xvbmdfcGFydGlhbF9wcmVmaWxscyI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heCBjb25jdXJyZW50IGxvbmcgcHJvbXB0cyBmb3IgY2h1bmtlZCBwcmVmaWxsIgogICAgfSwKICAgICJsb25nX3ByZWZpbGxfdG9rZW5fdGhyZXNob2xkIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9rZW4gdGhyZXNob2xkIGZvciBsb25nIHByZWZpbGwiCiAgICB9LAogICAgIm51bV9sb29rYWhlYWRfc2xvdHMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVmYXVsdCI6IDAsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJTbG90cyBmb3Igc3BlY3VsYXRpdmUgZGVjb2RpbmciCiAgICB9LAogICAgInNjaGVkdWxpbmdfcG9saWN5IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiZmNmcyIsICJwcmlvcml0eSJdLAogICAgICAiZGVmYXVsdCI6ICJmY2ZzIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSBzY2hlZHVsaW5nIHBvbGljeSB0byB1c2UiCiAgICB9LAogICAgImVuYWJsZV9jaHVua2VkX3ByZWZpbGwiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGNodW5rZWQgcHJlZmlsbCByZXF1ZXN0cyIKICAgIH0sCiAgICAiZGlzYWJsZV9jaHVua2VkX21tX2lucHV0IjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgcGFydGlhbCBzY2hlZHVsaW5nIG9mIG11bHRpbW9kYWwgaXRlbXMiCiAgICB9LAogICAgInNjaGVkdWxlcl9jbHMiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJUaGUgc2NoZWR1bGVyIGNsYXNzIHRvIHVzZSIKICAgIH0sCiAgICAiZGlzYWJsZV9oeWJyaWRfa3ZfY2FjaGVfbWFuYWdlciI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEaXNhYmxlIGh5YnJpZCBLViBjYWNoZSBtYW5hZ2VyIgogICAgfSwKICAgICJhc3luY19zY2hlZHVsaW5nIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlBlcmZvcm0gYXN5bmMgc2NoZWR1bGluZyIKICAgIH0sCiAgICAic3RyZWFtX2ludGVydmFsIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAxLAogICAgICAiZGVzY3JpcHRpb24iOiAiSW50ZXJ2YWwgZm9yIHN0cmVhbWluZyBpbiB0ZXJtcyBvZiB0b2tlbnMiCiAgICB9LAogICAgImN1ZGFncmFwaF9jYXB0dXJlX3NpemVzIjogewogICAgICAidHlwZSI6ICJhcnJheSIsCiAgICAgICJpdGVtcyI6IHsKICAgICAgICAidHlwZSI6ICJpbnRlZ2VyIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZXMgdG8gY2FwdHVyZSBjdWRhZ3JhcGgiCiAgICB9LAogICAgIm1heF9jdWRhZ3JhcGhfY2FwdHVyZV9zaXplIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gY3VkYWdyYXBoIGNhcHR1cmUgc2l6ZSIKICAgIH0sCiAgICAiY3VkYWdyYXBoX21vZGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJDVURBIGdyYXBoIG1vZGUgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiY3VkYWdyYXBoX251bV9vZl93YXJtdXBzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiTnVtYmVyIG9mIENVREEgZ3JhcGggd2FybXVwIGl0ZXJhdGlvbnMiCiAgICB9LAogICAgImN1ZGFncmFwaF9jb3B5X2lucHV0cyI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJDb3B5IENVREEgZ3JhcGggaW5wdXRzIgogICAgfSwKICAgICJjdWRhZ3JhcGhfc3BlY2lhbGl6ZV9sb3JhIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlNwZWNpYWxpemUgQ1VEQSBncmFwaHMgZm9yIExvUkEiCiAgICB9LAogICAgInNwZWN1bGF0aXZlX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIlNwZWN1bGF0aXZlIGRlY29kaW5nIGNvbmZpZ3VyYXRpb24iCiAgICB9LAogICAgImt2X3RyYW5zZmVyX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc3RyaWJ1dGVkIEtWIGNhY2hlIHRyYW5zZmVyIGNvbmZpZyIKICAgIH0sCiAgICAia3ZfZXZlbnRzX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkV2ZW50IHB1Ymxpc2hpbmcgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiZWNfdHJhbnNmZXJfY29uZmlnIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgIm9iamVjdCJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzdHJpYnV0ZWQgRUMgY2FjaGUgdHJhbnNmZXIgY29uZmlnIgogICAgfSwKICAgICJjb21waWxhdGlvbl9jb25maWciOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJ0b3JjaC5jb21waWxlIGFuZCBjdWRhZ3JhcGggY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiYWRkaXRpb25hbF9jb25maWciOiB7CiAgICAgICJ0eXBlIjogIm9iamVjdCIsCiAgICAgICJkZWZhdWx0Ijoge30sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBZGRpdGlvbmFsIHBsYXRmb3JtLXNwZWNpZmljIGNvbmZpZ3VyYXRpb24iCiAgICB9LAogICAgInN0cnVjdHVyZWRfb3V0cHV0c19jb25maWciOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJTdHJ1Y3R1cmVkIG91dHB1dHMgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiaGVhZGxlc3MiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiUnVuIGluIGhlYWRsZXNzIG1vZGUiCiAgICB9LAogICAgImFwaV9zZXJ2ZXJfY291bnQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBBUEkgc2VydmVyIHByb2Nlc3NlcyIKICAgIH0sCiAgICAiZGlzYWJsZV9sb2dfc3RhdHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzYWJsZSBsb2dnaW5nIHN0YXRpc3RpY3MiCiAgICB9LAogICAgImFnZ3JlZ2F0ZV9lbmdpbmVfbG9nZ2luZyI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJMb2cgYWdncmVnYXRlIHN0YXRpc3RpY3Mgd2l0aCBkYXRhIHBhcmFsbGVsIgogICAgfSwKICAgICJjaGF0X3RlbXBsYXRlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiQ2hhdCB0ZW1wbGF0ZSBmaWxlIHBhdGggb3IgaW5saW5lIHRlbXBsYXRlIgogICAgfSwKICAgICJjaGF0X3RlbXBsYXRlX2NvbnRlbnRfZm9ybWF0IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJzdHJpbmciLCAib3BlbmFpIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiQ2hhdCB0ZW1wbGF0ZSBjb250ZW50IGZvcm1hdCIKICAgIH0sCiAgICAidHJ1c3RfcmVxdWVzdF9jaGF0X3RlbXBsYXRlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlRydXN0IGNoYXQgdGVtcGxhdGUgZnJvbSByZXF1ZXN0IgogICAgfSwKICAgICJyZXNwb25zZV9yb2xlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICJhc3Npc3RhbnQiLAogICAgICAiZGVzY3JpcHRpb24iOiAiUm9sZSBuYW1lIHRvIHJldHVybiBmb3IgZ2VuZXJhdGlvbiIKICAgIH0sCiAgICAiZW5hYmxlX3Byb21wdF90b2tlbnNfZGV0YWlscyI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFbmFibGUgcHJvbXB0IHRva2VucyBkZXRhaWxzIGluIHJlc3BvbnNlIgogICAgfSwKICAgICJlbmFibGVfYXV0b190b29sX2Nob2ljZSI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFbmFibGUgYXV0byB0b29sIGNob2ljZSIKICAgIH0sCiAgICAiZXhjbHVkZV90b29sc193aGVuX3Rvb2xfY2hvaWNlX25vbmUiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRXhjbHVkZSB0b29scyB3aGVuIHRvb2xfY2hvaWNlIGlzIG5vbmUiCiAgICB9LAogICAgInRvb2xfY2FsbF9wYXJzZXIiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJUb29sIGNhbGwgcGFyc2VyIHRvIHVzZSIKICAgIH0sCiAgICAidG9vbF9wYXJzZXJfcGx1Z2luIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9vbCBwYXJzZXIgcGx1Z2luIHBhdGgiCiAgICB9LAogICAgInRvb2xfc2VydmVyIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9vbCBzZXJ2ZXIgaG9zdDpwb3J0IHBhaXJzIgogICAgfSwKICAgICJlbmFibGVfc2VydmVyX2xvYWRfdHJhY2tpbmciOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVHJhY2sgc2VydmVyX2xvYWRfbWV0cmljcyBpbiBhcHAgc3RhdGUiCiAgICB9LAogICAgImVuYWJsZV9mb3JjZV9pbmNsdWRlX3VzYWdlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkluY2x1ZGUgdXNhZ2Ugb24gZXZlcnkgcmVxdWVzdCIKICAgIH0sCiAgICAiZW5hYmxlX3Rva2VuaXplcl9pbmZvX2VuZHBvaW50IjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSAvZ2V0X3Rva2VuaXplcl9pbmZvIGVuZHBvaW50IgogICAgfSwKICAgICJlbmFibGVfbG9nX291dHB1dHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiTG9nIG1vZGVsIG91dHB1dHMiCiAgICB9LAogICAgImxvZ19lcnJvcl9zdGFjayI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJMb2cgc3RhY2sgdHJhY2Ugb2YgZXJyb3IgcmVzcG9uc2VzIgogICAgfSwKICAgICJ0b2tlbnNfb25seSI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJPbmx5IGVuYWJsZSBUb2tlbnMgSW48Pk91dCBlbmRwb2ludCIKICAgIH0sCiAgICAicHJlZW1wdGlvbl9tb2RlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsic3dhcCIsICJyZWNvbXB1dGUiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIlByZWVtcHRpb24gbW9kZSBkdXJpbmcgbWVtb3J5IHNob3J0YWdlIgogICAgfSwKICAgICJudW1fc2NoZWR1bGVyX3N0ZXBzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgIm1pbmltdW0iOiAxLAogICAgICAiZGVmYXVsdCI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOdW1iZXIgb2Ygc2NoZWR1bGVyIHN0ZXBzIgogICAgfSwKICAgICJtdWx0aV9zdGVwX3N0cmVhbV9vdXRwdXRzIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBtdWx0aS1zdGVwIHN0cmVhbSBvdXRwdXRzIgogICAgfSwKICAgICJyb3BlX3NjYWxpbmciOiB7CiAgICAgICJ0eXBlIjogIm9iamVjdCIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJSb1BFIHNjYWxpbmcgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAicm9wZV90aGV0YSI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiUm9QRSB0aGV0YSBwYXJhbWV0ZXIiCiAgICB9LAogICAgIm1heF9zZXFfbGVuX3RvX2NhcHR1cmUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogODE5MiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gc2VxdWVuY2UgbGVuZ3RoIGNvdmVyZWQgYnkgQ1VEQSBncmFwaHMiCiAgICB9CiAgfSwKICAiYWRkaXRpb25hbFByb3BlcnRpZXMiOiBmYWxzZQp9Cg=="

    deploy_template:
      kubernetes:
        default: "YXBpVmVyc2lvbjogYXBwcy92MQpraW5kOiBEZXBsb3ltZW50Cm1ldGFkYXRhOgogIG5hbWU6IHt7IC5FbmRwb2ludE5hbWUgfX0KICBuYW1lc3BhY2U6IHt7IC5OYW1lc3BhY2UgfX0KICBsYWJlbHM6CiAgICBlbmdpbmU6IHt7IC5FbmdpbmVOYW1lIH19CiAgICBlbmdpbmVfdmVyc2lvbjoge3sgLkVuZ2luZVZlcnNpb24gfX0KICAgIHJvdXRpbmdfbG9naWM6IHt7IC5Sb3V0aW5nTG9naWMgfX0KICAgIGFwcDogaW5mZXJlbmNlCnNwZWM6CiAgcmVwbGljYXM6IHt7IC5SZXBsaWNhcyB9fQogIHByb2dyZXNzRGVhZGxpbmVTZWNvbmRzOiAxMjAwCiAgc3RyYXRlZ3k6CiAgICB0eXBlOiBSb2xsaW5nVXBkYXRlCiAgICByb2xsaW5nVXBkYXRlOgogICAgICBtYXhVbmF2YWlsYWJsZTogMQogICAgICBtYXhTdXJnZTogMAogIHNlbGVjdG9yOgogICAgbWF0Y2hMYWJlbHM6CiAgICAgIGNsdXN0ZXI6IHt7IC5DbHVzdGVyTmFtZSB9fQogICAgICB3b3Jrc3BhY2U6IHt7IC5Xb3Jrc3BhY2UgfX0KICAgICAgZW5kcG9pbnQ6IHt7IC5FbmRwb2ludE5hbWUgfX0KICAgICAgYXBwOiBpbmZlcmVuY2UKICB0ZW1wbGF0ZToKICAgIG1ldGFkYXRhOgogICAgICBsYWJlbHM6CiAgICAgICAgZW5naW5lOiB7eyAuRW5naW5lTmFtZSB9fQogICAgICAgIGVuZ2luZV92ZXJzaW9uOiB7eyAuRW5naW5lVmVyc2lvbiB9fQogICAgICAgIGNsdXN0ZXI6IHt7IC5DbHVzdGVyTmFtZSB9fQogICAgICAgIHdvcmtzcGFjZToge3sgLldvcmtzcGFjZSB9fQogICAgICAgIGVuZHBvaW50OiB7eyAuRW5kcG9pbnROYW1lIH19CiAgICAgICAgcm91dGluZ19sb2dpYzoge3sgLlJvdXRpbmdMb2dpYyB9fQogICAgICAgIGFwcDogaW5mZXJlbmNlCiAgICBzcGVjOgogICAgICBhZmZpbml0eToKICAgICAgICBwb2RBbnRpQWZmaW5pdHk6CiAgICAgICAgICBwcmVmZXJyZWREdXJpbmdTY2hlZHVsaW5nSWdub3JlZER1cmluZ0V4ZWN1dGlvbjoKICAgICAgICAgICAgLSB3ZWlnaHQ6IDEwMAogICAgICAgICAgICAgIHBvZEFmZmluaXR5VGVybToKICAgICAgICAgICAgICAgIGxhYmVsU2VsZWN0b3I6CiAgICAgICAgICAgICAgICAgIG1hdGNoRXhwcmVzc2lvbnM6CiAgICAgICAgICAgICAgICAgICAgLSBrZXk6IGVuZHBvaW50CiAgICAgICAgICAgICAgICAgICAgICBvcGVyYXRvcjogSW4KICAgICAgICAgICAgICAgICAgICAgIHZhbHVlczoKICAgICAgICAgICAgICAgICAgICAgICAgLSB7eyAuRW5kcG9pbnROYW1lIH19CiAgICAgICAgICAgICAgICB0b3BvbG9neUtleTogImt1YmVybmV0ZXMuaW8vaG9zdG5hbWUiCiAgICAgIHt7LSBpZiAuTm9kZVNlbGVjdG9yIH19CiAgICAgIG5vZGVTZWxlY3RvcjoKICAgICAgICB7ey0gcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5Ob2RlU2VsZWN0b3IgfX0KICAgICAgICB7eyAka2V5IH19OiB7eyAkdmFsdWUgfX0KICAgICAgICB7ey0gZW5kIH19CiAgICAgIHt7LSBlbmQgfX0KICAgICAge3stIGlmIC5JbWFnZVB1bGxTZWNyZXQgfX0KICAgICAgaW1hZ2VQdWxsU2VjcmV0czoKICAgICAgICAtIG5hbWU6IHt7IC5JbWFnZVB1bGxTZWNyZXQgfX0KICAgICAge3stIGVuZCB9fQoKICAgICAge3stIGlmIC5Wb2x1bWVzIH19CiAgICAgIHZvbHVtZXM6Cnt7IC5Wb2x1bWVzIHwgdG9ZYW1sIHwgaW5kZW50IDYgfX0KICAgICAge3stIGVuZCB9fQogICAgICBpbml0Q29udGFpbmVyczoKICAgICAgICAtIG5hbWU6IG1vZGVsLWRvd25sb2FkZXIKICAgICAgICAgIGltYWdlOiB7eyAuSW1hZ2VQcmVmaXggfX0vbmV1dHJlZS9uZXV0cmVlLXJ1bnRpbWU6e3sgLk5ldXRyZWVWZXJzaW9uIH19CiAgICAgICAgICBjb21tYW5kOgogICAgICAgICAgICAtIGJhc2gKICAgICAgICAgICAgLSAtYwogICAgICAgICAgYXJnczoKICAgICAgICAgICAgLSA+LQogICAgICAgICAgICAgIHB5dGhvbjMgLW0gbmV1dHJlZS5kb3dubG9hZGVyCiAgICAgICAgICAgICAgLS1uYW1lPSJ7eyAuTW9kZWxBcmdzLm5hbWUgfX0iCiAgICAgICAgICAgICAgLS1yZWdpc3RyeV90eXBlPSJ7eyAuTW9kZWxBcmdzLnJlZ2lzdHJ5X3R5cGUgfX0iCiAgICAgICAgICAgICAgLS1yZWdpc3RyeV9wYXRoPSJ7eyAuTW9kZWxBcmdzLnJlZ2lzdHJ5X3BhdGggfX0iCiAgICAgICAgICAgICAgLS1wYXRoPSJ7eyAuTW9kZWxBcmdzLnBhdGggfX0iCiAgICAgICAgICAgICAgLS12ZXJzaW9uPSJ7eyAuTW9kZWxBcmdzLnZlcnNpb24gfX0iCiAgICAgICAgICAgICAgLS1maWxlPSJ7eyAuTW9kZWxBcmdzLmZpbGUgfX0iCiAgICAgICAgICAgICAgLS10YXNrPSJ7eyAuTW9kZWxBcmdzLnRhc2sgfX0iCiAgICAgICAgICBlbnY6CiAgICAgICAgICAge3sgcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5FbnYgfX0KICAgICAgICAgICAtIG5hbWU6IHt7ICRrZXkgfX0KICAgICAgICAgICAgIHZhbHVlOiAie3sgJHZhbHVlIH19IgogICAgICAgICAgIHt7IGVuZCB9fQogICAgICAgICAge3stIGlmIC5Wb2x1bWVNb3VudHMgfX0KICAgICAgICAgIHZvbHVtZU1vdW50czoKe3sgLlZvbHVtZU1vdW50cyB8IHRvWWFtbCB8IGluZGVudCAxMCB9fQogICAgICAgICAge3stIGVuZCB9fQoKICAgICAgY29udGFpbmVyczoKICAgICAgICAtIG5hbWU6IHt7IC5FbmdpbmVOYW1lIH19CiAgICAgICAgICBpbWFnZToge3sgLkltYWdlUHJlZml4IH19L3t7IC5JbWFnZVJlcG8gfX06e3sgLkltYWdlVGFnIH19CiAgICAgICAgICBjb21tYW5kOgogICAgICAgICAgLSB2bGxtCiAgICAgICAgICAtIHNlcnZlCiAgICAgICAgICAtIHt7IC5Nb2RlbEFyZ3MucGF0aCB9fQogICAgICAgICAgLSAtLWhvc3QKICAgICAgICAgIC0gIjAuMC4wLjAiCiAgICAgICAgICAtICItLXBvcnQiCiAgICAgICAgICAtICI4MDAwIgogICAgICAgICAgLSAtLXNlcnZlZC1tb2RlbC1uYW1lCiAgICAgICAgICAtIHt7IC5Nb2RlbEFyZ3Muc2VydmVfbmFtZSB9fQogICAgICAgICAgLSAtLXRhc2sKICAgICAgICAgIHt7LSBpZiBlcSAuTW9kZWxBcmdzLnRhc2sgInRleHQtZW1iZWRkaW5nIiB9fQogICAgICAgICAgLSBlbWJlZGRpbmcKICAgICAgICAgIHt7LSBlbHNlIGlmIGVxIC5Nb2RlbEFyZ3MudGFzayAidGV4dC1nZW5lcmF0aW9uIiB9fQogICAgICAgICAgLSBnZW5lcmF0ZQogICAgICAgICAge3stIGVsc2UgaWYgZXEgLk1vZGVsQXJncy50YXNrICJ0ZXh0LXJlcmFuayIgfX0KICAgICAgICAgIC0gc2NvcmUKICAgICAgICAgIHt7LSBlbHNlIH19CiAgICAgICAgICAtIHt7IC5Nb2RlbEFyZ3MudGFzayB9fQogICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAge3stIGlmIC5FbmdpbmVBcmdzIH19CiAgICAgICAgICB7ey0gcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5FbmdpbmVBcmdzIH19CiAgICAgICAgICAtIC0te3sgJGtleSB9fQogICAgICB7ey0gaWYgbmUgKHByaW50ZiAiJXYiICR2YWx1ZSkgInRydWUifX0KICAgICAgICAgIC0gInt7ICR2YWx1ZSB9fSIKICAgICAge3stIGVuZCB9fQogICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAgcmVzb3VyY2VzOgogICAgICAgICAgICBsaW1pdHM6CiAgICAgICAgICAgICAge3stIHJhbmdlICRrZXksICR2YWx1ZSA6PSAuUmVzb3VyY2VzIH19CiAgICAgICAgICAgICAge3sgJGtleSB9fToge3sgJHZhbHVlIH19CiAgICAgICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAgICByZXF1ZXN0czoKICAgICAgICAgICAgICB7ey0gcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5SZXNvdXJjZXMgfX0KICAgICAgICAgICAgICB7eyAka2V5IH19OiB7eyAkdmFsdWUgfX0KICAgICAgICAgICAgICB7ey0gZW5kIH19CiAgICAgICAgICBlbnY6CiAgICAgICAgICAge3sgcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5FbnYgfX0KICAgICAgICAgICAtIG5hbWU6IHt7ICRrZXkgfX0KICAgICAgICAgICAgIHZhbHVlOiAie3sgJHZhbHVlIH19IgogICAgICAgICAgIHt7IGVuZCB9fQogICAgICAgICAgcG9ydHM6CiAgICAgICAgICAgIC0gY29udGFpbmVyUG9ydDogODAwMAogICAgICAgICAgc3RhcnR1cFByb2JlOgogICAgICAgICAgICBodHRwR2V0OgogICAgICAgICAgICAgIHBhdGg6IC9oZWFsdGgKICAgICAgICAgICAgICBwb3J0OiA4MDAwCiAgICAgICAgICAgIGluaXRpYWxEZWxheVNlY29uZHM6IDUKICAgICAgICAgICAgdGltZW91dFNlY29uZHM6IDUKICAgICAgICAgICAgcGVyaW9kU2Vjb25kczogMTAKICAgICAgICAgICAgc3VjY2Vzc1RocmVzaG9sZDogMQogICAgICAgICAgICBmYWlsdXJlVGhyZXNob2xkOiAxMjAKICAgICAgICAgIHJlYWRpbmVzc1Byb2JlOgogICAgICAgICAgICBodHRwR2V0OgogICAgICAgICAgICAgIHBhdGg6IC9oZWFsdGgKICAgICAgICAgICAgICBwb3J0OiA4MDAwCiAgICAgICAgICAgIGluaXRpYWxEZWxheVNlY29uZHM6IDUKICAgICAgICAgICAgdGltZW91dFNlY29uZHM6IDUKICAgICAgICAgICAgcGVyaW9kU2Vjb25kczogMTAKICAgICAgICAgICAgc3VjY2Vzc1RocmVzaG9sZDogMQogICAgICAgICAgICBmYWlsdXJlVGhyZXNob2xkOiAzCiAgICAgICAgICB7ey0gaWYgLlZvbHVtZU1vdW50cyB9fQogICAgICAgICAgdm9sdW1lTW91bnRzOgp7eyAuVm9sdW1lTW91bnRzIHwgdG9ZYW1sIHwgaW5kZW50IDEwIH19CiAgICAgICAgICB7ey0gZW5kIH19"

    supported_tasks:
      - "generate"

    images:
      nvidia_gpu:
        image_name: "vllm/vllm-openai"
        tag: "v0.11.2"
```



values_schema_base64.v0.17.1.json 

```
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "vLLM v0.17.1 Engine Configuration",
  "description": "Configuration schema for vLLM v0.17.1 engine parameters",
  "properties": {
    "model": {
      "type": "string",
      "description": "Name or path of the Hugging Face model to use"
    },
    "runner": {
      "type": "string",
      "enum": ["auto", "draft", "generate", "pooling"],
      "default": "auto",
      "description": "The type of model runner to use"
    },
    "convert": {
      "type": "string",
      "enum": ["auto", "classify", "embed", "none", "reward"],
      "default": "auto",
      "description": "Convert the model using adapters"
    },
    "tokenizer": {
      "type": "string",
      "description": "Name or path of the Hugging Face tokenizer to use"
    },
    "tokenizer_mode": {
      "type": "string",
      "enum": ["auto", "slow", "mistral", "custom"],
      "default": "auto",
      "description": "The tokenizer mode to use"
    },
    "trust_remote_code": {
      "type": "boolean",
      "default": false,
      "description": "Trust remote code from Hugging Face"
    },
    "dtype": {
      "type": "string",
      "enum": ["auto", "half", "float16", "bfloat16", "float", "float32"],
      "default": "auto",
      "description": "Data type for model weights and activations"
    },
    "seed": {
      "type": "integer",
      "minimum": 0,
      "description": "Random seed for reproducibility"
    },
    "hf_config_path": {
      "type": "string",
      "description": "Name or path of the Hugging Face config to use"
    },
    "allowed_local_media_path": {
      "type": "string",
      "default": "",
      "description": "Allowed local media path for security"
    },
    "allowed_media_domains": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "Allowed media domains for multi-modal inputs"
    },
    "revision": {
      "type": "string",
      "description": "The specific model version to use"
    },
    "code_revision": {
      "type": "string",
      "description": "The specific revision to use for the model code"
    },
    "tokenizer_revision": {
      "type": "string",
      "description": "Revision of the tokenizer to use"
    },
    "max_model_len": {
      "type": ["integer", "string"],
      "description": "Model context length. Supports human-readable format like '1k', '2M'"
    },
    "quantization": {
      "type": "string",
      "description": "Method used to quantize the weights"
    },
    "enforce_eager": {
      "type": "boolean",
      "default": false,
      "description": "Always use eager-mode PyTorch"
    },
    "max_logprobs": {
      "type": "integer",
      "default": 20,
      "description": "Maximum number of log probabilities to return"
    },
    "logprobs_mode": {
      "type": "string",
      "enum": ["processed_logits", "processed_logprobs", "raw_logits", "raw_logprobs"],
      "default": "raw_logprobs",
      "description": "Indicates the content returned in logprobs"
    },
    "disable_sliding_window": {
      "type": "boolean",
      "default": false,
      "description": "Disable sliding window attention"
    },
    "disable_cascade_attn": {
      "type": "boolean",
      "default": false,
      "description": "Disable cascade attention for V1"
    },
    "skip_tokenizer_init": {
      "type": "boolean",
      "default": false,
      "description": "Skip initialization of tokenizer and detokenizer"
    },
    "enable_prompt_embeds": {
      "type": "boolean",
      "default": false,
      "description": "Enable passing text embeddings via prompt_embeds"
    },
    "served_model_name": {
      "type": ["string", "array"],
      "items": {
        "type": "string"
      },
      "description": "The model name(s) used in the API"
    },
    "config_format": {
      "type": "string",
      "enum": ["auto", "hf", "mistral"],
      "default": "auto",
      "description": "The format of the model config to load"
    },
    "hf_token": {
      "type": ["string", "boolean"],
      "description": "Hugging Face token for authentication"
    },
    "hf_overrides": {
      "type": "object",
      "default": {},
      "description": "Arguments to be forwarded to HuggingFace config"
    },
    "pooler_config": {
      "type": ["string", "object"],
      "description": "Pooler config for output pooling behavior"
    },
    "logits_processor_pattern": {
      "type": "string",
      "description": "Regex pattern for valid logits processor names"
    },
    "generation_config": {
      "type": "string",
      "default": "auto",
      "description": "Path to generation config"
    },
    "override_generation_config": {
      "type": "object",
      "default": {},
      "description": "Override generation config parameters"
    },
    "enable_sleep_mode": {
      "type": "boolean",
      "default": false,
      "description": "Enable sleep mode for the engine"
    },
    "model_impl": {
      "type": "string",
      "enum": ["auto", "terratorch", "transformers", "vllm"],
      "default": "auto",
      "description": "Which implementation of the model to use"
    },
    "override_attention_dtype": {
      "type": "string",
      "description": "Override dtype for attention"
    },
    "logits_processors": {
      "type": ["string", "array"],
      "items": {
        "type": "string"
      },
      "description": "Logits processors class names"
    },
    "io_processor_plugin": {
      "type": "string",
      "description": "IOProcessor plugin name to load at startup"
    },
    "load_format": {
      "type": "string",
      "default": "auto",
      "description": "The format of model weights to load"
    },
    "download_dir": {
      "type": "string",
      "description": "Directory to download and load weights"
    },
    "safetensors_load_strategy": {
      "type": "string",
      "enum": ["lazy", "eager", "torchao"],
      "default": "lazy",
      "description": "Loading strategy for safetensors weights"
    },
    "model_loader_extra_config": {
      "type": "object",
      "default": {},
      "description": "Extra config for model loader"
    },
    "ignore_patterns": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "default": ["original/**/*"],
      "description": "Patterns to ignore when loading model"
    },
    "use_tqdm_on_load": {
      "type": "boolean",
      "default": true,
      "description": "Enable tqdm progress bar when loading"
    },
    "pt_load_map_location": {
      "type": ["string", "object"],
      "default": "cpu",
      "description": "Map location for loading pytorch checkpoint"
    },
    "reasoning_parser": {
      "type": "string",
      "default": "",
      "description": "Reasoning parser to use"
    },
    "reasoning_parser_plugin": {
      "type": "string",
      "default": "",
      "description": "Path to reasoning parser plugin"
    },
    "distributed_executor_backend": {
      "type": "string",
      "enum": ["external_launcher", "mp", "ray", "uni"],
      "description": "Backend for distributed model workers"
    },
    "pipeline_parallel_size": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of pipeline parallel groups"
    },
    "master_addr": {
      "type": "string",
      "default": "127.0.0.1",
      "description": "Distributed master address"
    },
    "master_port": {
      "type": "integer",
      "default": 29501,
      "description": "Distributed master port"
    },
    "nnodes": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of nodes for multi-node inference"
    },
    "node_rank": {
      "type": "integer",
      "minimum": 0,
      "default": 0,
      "description": "Distributed node rank"
    },
    "tensor_parallel_size": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of tensor parallel groups"
    },
    "decode_context_parallel_size": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of decode context parallel groups"
    },
    "dcp_kv_cache_interleave_size": {
      "type": "integer",
      "description": "KV cache interleave size for decode context parallelism"
    },
    "data_parallel_size": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of data parallel replicas"
    },
    "data_parallel_rank": {
      "type": "integer",
      "minimum": 0,
      "description": "Data parallel rank of this instance"
    },
    "data_parallel_start_rank": {
      "type": "integer",
      "minimum": 0,
      "description": "Starting data parallel rank for secondary nodes"
    },
    "data_parallel_size_local": {
      "type": "integer",
      "minimum": 1,
      "description": "Number of data parallel replicas on this node"
    },
    "data_parallel_address": {
      "type": "string",
      "description": "Address of data parallel cluster head-node"
    },
    "data_parallel_rpc_port": {
      "type": "integer",
      "description": "Port for data parallel RPC communication"
    },
    "data_parallel_backend": {
      "type": "string",
      "enum": ["mp", "ray"],
      "default": "mp",
      "description": "Backend for data parallel"
    },
    "data_parallel_hybrid_lb": {
      "type": "boolean",
      "default": false,
      "description": "Use hybrid DP load balancing mode"
    },
    "data_parallel_external_lb": {
      "type": "boolean",
      "default": false,
      "description": "Use external DP load balancing mode"
    },
    "enable_expert_parallel": {
      "type": "boolean",
      "default": false,
      "description": "Enable expert parallelism for MoE models"
    },
    "all2all_backend": {
      "type": "string",
      "enum": ["allgather_reducescatter", "deepep_high_throughput", "deepep_low_latency", "flashinfer_all2allv", "naive", "pplx", "None"],
      "description": "All2All backend for MoE expert parallel"
    },
    "enable_dbo": {
      "type": "boolean",
      "default": false,
      "description": "Enable dual batch overlap"
    },
    "dbo_decode_token_threshold": {
      "type": "integer",
      "default": 32,
      "description": "Token threshold for dual batch overlap decode"
    },
    "dbo_prefill_token_threshold": {
      "type": "integer",
      "default": 512,
      "description": "Token threshold for dual batch overlap prefill"
    },
    "disable_nccl_for_dp_synchronization": {
      "type": "boolean",
      "default": false,
      "description": "Force DP sync to use Gloo instead of NCCL"
    },
    "enable_eplb": {
      "type": "boolean",
      "default": false,
      "description": "Enable expert parallelism load balancing"
    },
    "eplb_config": {
      "type": ["string", "object"],
      "description": "Expert parallelism configuration"
    },
    "expert_placement_strategy": {
      "type": "string",
      "enum": ["linear", "round_robin"],
      "default": "linear",
      "description": "Expert placement strategy for MoE layers"
    },
    "max_parallel_loading_workers": {
      "type": "integer",
      "description": "Maximum number of parallel loading workers"
    },
    "ray_workers_use_nsight": {
      "type": "boolean",
      "default": false,
      "description": "Profile Ray workers with nsight"
    },
    "disable_custom_all_reduce": {
      "type": "boolean",
      "default": false,
      "description": "Disable custom all-reduce kernels"
    },
    "worker_cls": {
      "type": "string",
      "default": "auto",
      "description": "Full name of the worker class to use"
    },
    "worker_extension_cls": {
      "type": "string",
      "default": "",
      "description": "Worker extension class name"
    },
    "enable_multimodal_encoder_data_parallel": {
      "type": "boolean",
      "default": false,
      "description": "Enable multimodal encoder data parallel"
    },
    "block_size": {
      "type": "integer",
      "enum": [1, 8, 16, 32, 64, 128, 256],
      "description": "Size of a contiguous cache block"
    },
    "gpu_memory_utilization": {
      "type": "number",
      "minimum": 0,
      "maximum": 1.0,
      "default": 0.9,
      "description": "The fraction of GPU memory to be used"
    },
    "kv_cache_memory_bytes": {
      "type": ["integer", "string"],
      "description": "Size of KV Cache per GPU in bytes"
    },
    "swap_space": {
      "type": "number",
      "minimum": 0,
      "default": 4,
      "description": "CPU swap space size (GiB) per GPU"
    },
    "kv_cache_dtype": {
      "type": "string",
      "enum": ["auto", "bfloat16", "fp8", "fp8_ds_mla", "fp8_e4m3", "fp8_e5m2", "fp8_inc"],
      "default": "auto",
      "description": "Data type for KV cache storage"
    },
    "num_gpu_blocks_override": {
      "type": "integer",
      "description": "Override profiled num_gpu_blocks"
    },
    "enable_prefix_caching": {
      "type": "boolean",
      "description": "Enable prefix caching"
    },
    "prefix_caching_hash_algo": {
      "type": "string",
      "enum": ["sha256", "sha256_cbor"],
      "default": "sha256",
      "description": "Hash algorithm for prefix caching"
    },
    "cpu_offload_gb": {
      "type": "number",
      "minimum": 0,
      "default": 0,
      "description": "The space in GiB to offload to CPU"
    },
    "calculate_kv_scales": {
      "type": "boolean",
      "default": false,
      "description": "Enable dynamic calculation of k_scale and v_scale"
    },
    "kv_sharing_fast_prefill": {
      "type": "boolean",
      "default": false,
      "description": "Enable KV sharing fast prefill optimization"
    },
    "mamba_cache_dtype": {
      "type": "string",
      "enum": ["auto", "float32"],
      "default": "auto",
      "description": "Data type for Mamba cache"
    },
    "mamba_ssm_cache_dtype": {
      "type": "string",
      "enum": ["auto", "float32"],
      "default": "auto",
      "description": "Data type for Mamba SSM state"
    },
    "mamba_block_size": {
      "type": "integer",
      "description": "Block size for mamba cache"
    },
    "kv_offloading_size": {
      "type": "number",
      "description": "Size of KV cache offloading buffer (GiB)"
    },
    "kv_offloading_backend": {
      "type": "string",
      "enum": ["lmcache", "native", "None"],
      "description": "Backend for KV cache offloading"
    },
    "limit_mm_per_prompt": {
      "type": ["string", "object"],
      "default": {},
      "description": "Maximum number of multi-modal items per prompt"
    },
    "enable_mm_embeds": {
      "type": "boolean",
      "default": false,
      "description": "Enable passing multimodal embeddings"
    },
    "media_io_kwargs": {
      "type": ["string", "object"],
      "default": {},
      "description": "Additional args for processing media inputs"
    },
    "mm_processor_kwargs": {
      "type": ["string", "object"],
      "description": "Arguments forwarded to multi-modal processor"
    },
    "mm_processor_cache_gb": {
      "type": "number",
      "default": 4,
      "description": "Size (GiB) of multi-modal processor cache"
    },
    "disable_mm_preprocessor_cache": {
      "type": "boolean",
      "default": false,
      "description": "Disable multi-modal preprocessor cache"
    },
    "mm_processor_cache_type": {
      "type": "string",
      "enum": ["lru", "shm"],
      "default": "lru",
      "description": "Type of cache for multi-modal processor"
    },
    "mm_shm_cache_max_object_size_mb": {
      "type": "integer",
      "default": 128,
      "description": "Max object size (MiB) in shared memory cache"
    },
    "mm_encoder_tp_mode": {
      "type": "string",
      "enum": ["data", "weights"],
      "description": "Multi-modal encoder tensor parallel mode"
    },
    "mm_encoder_attn_backend": {
      "type": "string",
      "description": "Multi-modal encoder attention backend"
    },
    "interleave_mm_strings": {
      "type": "boolean",
      "default": false,
      "description": "Enable fully interleaved multimodal prompts"
    },
    "skip_mm_profiling": {
      "type": "boolean",
      "default": false,
      "description": "Skip multimodal memory profiling"
    },
    "video_pruning_rate": {
      "type": "number",
      "minimum": 0,
      "maximum": 1,
      "description": "Pruning rate for video via Efficient Video Sampling"
    },
    "enable_lora": {
      "type": "boolean",
      "description": "Enable handling of LoRA adapters"
    },
    "max_loras": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Maximum number of LoRA adapters"
    },
    "max_lora_rank": {
      "type": "integer",
      "enum": [1, 8, 16, 32, 64, 128, 256, 320, 512],
      "default": 16,
      "description": "Maximum LoRA rank"
    },
    "lora_dtype": {
      "type": "string",
      "default": "auto",
      "description": "Data type for LoRA"
    },
    "max_cpu_loras": {
      "type": "integer",
      "description": "Maximum number of LoRAs to store in CPU memory"
    },
    "fully_sharded_loras": {
      "type": "boolean",
      "default": false,
      "description": "Use fully sharded LoRA layers"
    },
    "default_mm_loras": {
      "type": ["string", "object"],
      "description": "Default LoRA paths for specific modalities"
    },
    "show_hidden_metrics_for_version": {
      "type": "string",
      "description": "Show hidden metrics since specified version"
    },
    "otlp_traces_endpoint": {
      "type": "string",
      "description": "OpenTelemetry traces target URL"
    },
    "collect_detailed_traces": {
      "type": "string",
      "enum": ["all", "model", "worker", "None", "model,worker", "model,all", "worker,model", "worker,all", "all,model", "all,worker"],
      "description": "Collect detailed traces for specified modules"
    },
    "max_num_batched_tokens": {
      "type": ["integer", "string"],
      "description": "Maximum number of batched tokens per iteration"
    },
    "max_num_seqs": {
      "type": "integer",
      "description": "Maximum number of sequences per iteration"
    },
    "max_num_partial_prefills": {
      "type": "integer",
      "default": 1,
      "description": "Max concurrent partial prefills for chunked prefill"
    },
    "max_long_partial_prefills": {
      "type": "integer",
      "default": 1,
      "description": "Max concurrent long prompts for chunked prefill"
    },
    "long_prefill_token_threshold": {
      "type": "integer",
      "default": 0,
      "description": "Token threshold for long prefill"
    },
    "num_lookahead_slots": {
      "type": "integer",
      "default": 0,
      "description": "Slots for speculative decoding"
    },
    "scheduling_policy": {
      "type": "string",
      "enum": ["fcfs", "priority"],
      "default": "fcfs",
      "description": "The scheduling policy to use"
    },
    "enable_chunked_prefill": {
      "type": "boolean",
      "description": "Enable chunked prefill requests"
    },
    "disable_chunked_mm_input": {
      "type": "boolean",
      "default": false,
      "description": "Disable partial scheduling of multimodal items"
    },
    "scheduler_cls": {
      "type": "string",
      "description": "The scheduler class to use"
    },
    "disable_hybrid_kv_cache_manager": {
      "type": "boolean",
      "default": false,
      "description": "Disable hybrid KV cache manager"
    },
    "async_scheduling": {
      "type": "boolean",
      "default": false,
      "description": "Perform async scheduling"
    },
    "stream_interval": {
      "type": "integer",
      "default": 1,
      "description": "Interval for streaming in terms of tokens"
    },
    "cudagraph_capture_sizes": {
      "type": "array",
      "items": {
        "type": "integer"
      },
      "description": "Sizes to capture cudagraph"
    },
    "max_cudagraph_capture_size": {
      "type": "integer",
      "description": "Maximum cudagraph capture size"
    },
    "cudagraph_mode": {
      "type": "string",
      "description": "CUDA graph mode configuration"
    },
    "cudagraph_num_of_warmups": {
      "type": "integer",
      "default": 0,
      "description": "Number of CUDA graph warmup iterations"
    },
    "cudagraph_copy_inputs": {
      "type": "boolean",
      "description": "Copy CUDA graph inputs"
    },
    "cudagraph_specialize_lora": {
      "type": "boolean",
      "description": "Specialize CUDA graphs for LoRA"
    },
    "speculative_config": {
      "type": ["string", "object"],
      "description": "Speculative decoding configuration"
    },
    "kv_transfer_config": {
      "type": ["string", "object"],
      "description": "Distributed KV cache transfer config"
    },
    "kv_events_config": {
      "type": ["string", "object"],
      "description": "Event publishing configuration"
    },
    "ec_transfer_config": {
      "type": ["string", "object"],
      "description": "Distributed EC cache transfer config"
    },
    "compilation_config": {
      "type": ["string", "object"],
      "description": "torch.compile and cudagraph configuration"
    },
    "additional_config": {
      "type": "object",
      "default": {},
      "description": "Additional platform-specific configuration"
    },
    "structured_outputs_config": {
      "type": ["string", "object"],
      "description": "Structured outputs configuration"
    },
    "headless": {
      "type": "boolean",
      "default": false,
      "description": "Run in headless mode"
    },
    "api_server_count": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of API server processes"
    },
    "disable_log_stats": {
      "type": "boolean",
      "default": false,
      "description": "Disable logging statistics"
    },
    "aggregate_engine_logging": {
      "type": "boolean",
      "default": false,
      "description": "Log aggregate statistics with data parallel"
    },
    "chat_template": {
      "type": "string",
      "description": "Chat template file path or inline template"
    },
    "chat_template_content_format": {
      "type": "string",
      "enum": ["auto", "string", "openai"],
      "default": "auto",
      "description": "Chat template content format"
    },
    "trust_request_chat_template": {
      "type": "boolean",
      "default": false,
      "description": "Trust chat template from request"
    },
    "response_role": {
      "type": "string",
      "default": "assistant",
      "description": "Role name to return for generation"
    },
    "enable_prompt_tokens_details": {
      "type": "boolean",
      "default": false,
      "description": "Enable prompt tokens details in response"
    },
    "enable_auto_tool_choice": {
      "type": "boolean",
      "default": false,
      "description": "Enable auto tool choice"
    },
    "exclude_tools_when_tool_choice_none": {
      "type": "boolean",
      "default": false,
      "description": "Exclude tools when tool_choice is none"
    },
    "tool_call_parser": {
      "type": "string",
      "description": "Tool call parser to use"
    },
    "tool_parser_plugin": {
      "type": "string",
      "default": "",
      "description": "Tool parser plugin path"
    },
    "tool_server": {
      "type": "string",
      "description": "Tool server host:port pairs"
    },
    "enable_server_load_tracking": {
      "type": "boolean",
      "default": false,
      "description": "Track server_load_metrics in app state"
    },
    "enable_force_include_usage": {
      "type": "boolean",
      "default": false,
      "description": "Include usage on every request"
    },
    "enable_tokenizer_info_endpoint": {
      "type": "boolean",
      "default": false,
      "description": "Enable /get_tokenizer_info endpoint"
    },
    "enable_log_outputs": {
      "type": "boolean",
      "default": false,
      "description": "Log model outputs"
    },
    "log_error_stack": {
      "type": "boolean",
      "default": false,
      "description": "Log stack trace of error responses"
    },
    "tokens_only": {
      "type": "boolean",
      "default": false,
      "description": "Only enable Tokens In<>Out endpoint"
    },
    "preemption_mode": {
      "type": "string",
      "enum": ["swap", "recompute"],
      "description": "Preemption mode during memory shortage"
    },
    "num_scheduler_steps": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of scheduler steps"
    },
    "multi_step_stream_outputs": {
      "type": "boolean",
      "default": false,
      "description": "Enable multi-step stream outputs"
    },
    "rope_scaling": {
      "type": "object",
      "description": "RoPE scaling configuration"
    },
    "rope_theta": {
      "type": "number",
      "minimum": 0,
      "description": "RoPE theta parameter"
    },
    "max_seq_len_to_capture": {
      "type": "integer",
      "minimum": 1,
      "default": 8192,
      "description": "Maximum sequence length covered by CUDA graphs"
    }
  },
  "additionalProperties": false
}
```

to base64 然後貼到 manifest.yaml

```
base64 -w 0 values_schema_base64_v0.17.1.json > values_schema_base64_v0.17.1.txt
cat values_schema_base64_v0.17.1.txt 
```



#### v0.17.1 manifest.yaml

自我修改版本

```
manifest_version: "1.0"

metadata:
  description: "vLLM"
  author: "Ken Wang"
  created_at: "2026-03-18T03:44:44Z"
  version: v1.0.1
  tags:
    - "engine"
    - "vllm"
    - "v0.17.1"

images:
    - accelerator: "nvidia_gpu"
      image_name: "vllm/vllm-openai"
      tag: "v0.17.1"
      image_file: "images/vllm-vllm-openai-v0.17.1.tar"
      platform: "linux/amd64"
      size: 9269006336

engines:
- name: vllm
  engine_versions:
  - version: "v0.17.1"

    values_schema:
      values_schema_base64: "ewogICIkc2NoZW1hIjogImh0dHA6Ly9qc29uLXNjaGVtYS5vcmcvZHJhZnQtMDcvc2NoZW1hIyIsCiAgInR5cGUiOiAib2JqZWN0IiwKICAidGl0bGUiOiAidkxMTSB2MC4xNy4xIEVuZ2luZSBDb25maWd1cmF0aW9uIiwKICAiZGVzY3JpcHRpb24iOiAiQ29uZmlndXJhdGlvbiBzY2hlbWEgZm9yIHZMTE0gdjAuMTcuMSBlbmdpbmUgcGFyYW1ldGVycyIsCiAgInByb3BlcnRpZXMiOiB7CiAgICAibW9kZWwiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOYW1lIG9yIHBhdGggb2YgdGhlIEh1Z2dpbmcgRmFjZSBtb2RlbCB0byB1c2UiCiAgICB9LAogICAgInJ1bm5lciI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImF1dG8iLCAiZHJhZnQiLCAiZ2VuZXJhdGUiLCAicG9vbGluZyJdLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSB0eXBlIG9mIG1vZGVsIHJ1bm5lciB0byB1c2UiCiAgICB9LAogICAgImNvbnZlcnQiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgImNsYXNzaWZ5IiwgImVtYmVkIiwgIm5vbmUiLCAicmV3YXJkIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiQ29udmVydCB0aGUgbW9kZWwgdXNpbmcgYWRhcHRlcnMiCiAgICB9LAogICAgInRva2VuaXplciI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk5hbWUgb3IgcGF0aCBvZiB0aGUgSHVnZ2luZyBGYWNlIHRva2VuaXplciB0byB1c2UiCiAgICB9LAogICAgInRva2VuaXplcl9tb2RlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJzbG93IiwgIm1pc3RyYWwiLCAiY3VzdG9tIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIHRva2VuaXplciBtb2RlIHRvIHVzZSIKICAgIH0sCiAgICAidHJ1c3RfcmVtb3RlX2NvZGUiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVHJ1c3QgcmVtb3RlIGNvZGUgZnJvbSBIdWdnaW5nIEZhY2UiCiAgICB9LAogICAgImR0eXBlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJoYWxmIiwgImZsb2F0MTYiLCAiYmZsb2F0MTYiLCAiZmxvYXQiLCAiZmxvYXQzMiJdLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkRhdGEgdHlwZSBmb3IgbW9kZWwgd2VpZ2h0cyBhbmQgYWN0aXZhdGlvbnMiCiAgICB9LAogICAgInNlZWQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDAsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJSYW5kb20gc2VlZCBmb3IgcmVwcm9kdWNpYmlsaXR5IgogICAgfSwKICAgICJoZl9jb25maWdfcGF0aCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk5hbWUgb3IgcGF0aCBvZiB0aGUgSHVnZ2luZyBGYWNlIGNvbmZpZyB0byB1c2UiCiAgICB9LAogICAgImFsbG93ZWRfbG9jYWxfbWVkaWFfcGF0aCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlZmF1bHQiOiAiIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkFsbG93ZWQgbG9jYWwgbWVkaWEgcGF0aCBmb3Igc2VjdXJpdHkiCiAgICB9LAogICAgImFsbG93ZWRfbWVkaWFfZG9tYWlucyI6IHsKICAgICAgInR5cGUiOiAiYXJyYXkiLAogICAgICAiaXRlbXMiOiB7CiAgICAgICAgInR5cGUiOiAic3RyaW5nIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiQWxsb3dlZCBtZWRpYSBkb21haW5zIGZvciBtdWx0aS1tb2RhbCBpbnB1dHMiCiAgICB9LAogICAgInJldmlzaW9uIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIHNwZWNpZmljIG1vZGVsIHZlcnNpb24gdG8gdXNlIgogICAgfSwKICAgICJjb2RlX3JldmlzaW9uIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIHNwZWNpZmljIHJldmlzaW9uIHRvIHVzZSBmb3IgdGhlIG1vZGVsIGNvZGUiCiAgICB9LAogICAgInRva2VuaXplcl9yZXZpc2lvbiI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlJldmlzaW9uIG9mIHRoZSB0b2tlbml6ZXIgdG8gdXNlIgogICAgfSwKICAgICJtYXhfbW9kZWxfbGVuIjogewogICAgICAidHlwZSI6IFsiaW50ZWdlciIsICJzdHJpbmciXSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1vZGVsIGNvbnRleHQgbGVuZ3RoLiBTdXBwb3J0cyBodW1hbi1yZWFkYWJsZSBmb3JtYXQgbGlrZSAnMWsnLCAnMk0nIgogICAgfSwKICAgICJxdWFudGl6YXRpb24iOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNZXRob2QgdXNlZCB0byBxdWFudGl6ZSB0aGUgd2VpZ2h0cyIKICAgIH0sCiAgICAiZW5mb3JjZV9lYWdlciI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBbHdheXMgdXNlIGVhZ2VyLW1vZGUgUHlUb3JjaCIKICAgIH0sCiAgICAibWF4X2xvZ3Byb2JzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAyMCwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIGxvZyBwcm9iYWJpbGl0aWVzIHRvIHJldHVybiIKICAgIH0sCiAgICAibG9ncHJvYnNfbW9kZSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbInByb2Nlc3NlZF9sb2dpdHMiLCAicHJvY2Vzc2VkX2xvZ3Byb2JzIiwgInJhd19sb2dpdHMiLCAicmF3X2xvZ3Byb2JzIl0sCiAgICAgICJkZWZhdWx0IjogInJhd19sb2dwcm9icyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJJbmRpY2F0ZXMgdGhlIGNvbnRlbnQgcmV0dXJuZWQgaW4gbG9ncHJvYnMiCiAgICB9LAogICAgImRpc2FibGVfc2xpZGluZ193aW5kb3ciOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzYWJsZSBzbGlkaW5nIHdpbmRvdyBhdHRlbnRpb24iCiAgICB9LAogICAgImRpc2FibGVfY2FzY2FkZV9hdHRuIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgY2FzY2FkZSBhdHRlbnRpb24gZm9yIFYxIgogICAgfSwKICAgICJza2lwX3Rva2VuaXplcl9pbml0IjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlNraXAgaW5pdGlhbGl6YXRpb24gb2YgdG9rZW5pemVyIGFuZCBkZXRva2VuaXplciIKICAgIH0sCiAgICAiZW5hYmxlX3Byb21wdF9lbWJlZHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHBhc3NpbmcgdGV4dCBlbWJlZGRpbmdzIHZpYSBwcm9tcHRfZW1iZWRzIgogICAgfSwKICAgICJzZXJ2ZWRfbW9kZWxfbmFtZSI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJhcnJheSJdLAogICAgICAiaXRlbXMiOiB7CiAgICAgICAgInR5cGUiOiAic3RyaW5nIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIG1vZGVsIG5hbWUocykgdXNlZCBpbiB0aGUgQVBJIgogICAgfSwKICAgICJjb25maWdfZm9ybWF0IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJoZiIsICJtaXN0cmFsIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIGZvcm1hdCBvZiB0aGUgbW9kZWwgY29uZmlnIHRvIGxvYWQiCiAgICB9LAogICAgImhmX3Rva2VuIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgImJvb2xlYW4iXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkh1Z2dpbmcgRmFjZSB0b2tlbiBmb3IgYXV0aGVudGljYXRpb24iCiAgICB9LAogICAgImhmX292ZXJyaWRlcyI6IHsKICAgICAgInR5cGUiOiAib2JqZWN0IiwKICAgICAgImRlZmF1bHQiOiB7fSwKICAgICAgImRlc2NyaXB0aW9uIjogIkFyZ3VtZW50cyB0byBiZSBmb3J3YXJkZWQgdG8gSHVnZ2luZ0ZhY2UgY29uZmlnIgogICAgfSwKICAgICJwb29sZXJfY29uZmlnIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgIm9iamVjdCJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiUG9vbGVyIGNvbmZpZyBmb3Igb3V0cHV0IHBvb2xpbmcgYmVoYXZpb3IiCiAgICB9LAogICAgImxvZ2l0c19wcm9jZXNzb3JfcGF0dGVybiI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlJlZ2V4IHBhdHRlcm4gZm9yIHZhbGlkIGxvZ2l0cyBwcm9jZXNzb3IgbmFtZXMiCiAgICB9LAogICAgImdlbmVyYXRpb25fY29uZmlnIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlBhdGggdG8gZ2VuZXJhdGlvbiBjb25maWciCiAgICB9LAogICAgIm92ZXJyaWRlX2dlbmVyYXRpb25fY29uZmlnIjogewogICAgICAidHlwZSI6ICJvYmplY3QiLAogICAgICAiZGVmYXVsdCI6IHt9LAogICAgICAiZGVzY3JpcHRpb24iOiAiT3ZlcnJpZGUgZ2VuZXJhdGlvbiBjb25maWcgcGFyYW1ldGVycyIKICAgIH0sCiAgICAiZW5hYmxlX3NsZWVwX21vZGUiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHNsZWVwIG1vZGUgZm9yIHRoZSBlbmdpbmUiCiAgICB9LAogICAgIm1vZGVsX2ltcGwiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgInRlcnJhdG9yY2giLCAidHJhbnNmb3JtZXJzIiwgInZsbG0iXSwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJXaGljaCBpbXBsZW1lbnRhdGlvbiBvZiB0aGUgbW9kZWwgdG8gdXNlIgogICAgfSwKICAgICJvdmVycmlkZV9hdHRlbnRpb25fZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJPdmVycmlkZSBkdHlwZSBmb3IgYXR0ZW50aW9uIgogICAgfSwKICAgICJsb2dpdHNfcHJvY2Vzc29ycyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJhcnJheSJdLAogICAgICAiaXRlbXMiOiB7CiAgICAgICAgInR5cGUiOiAic3RyaW5nIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiTG9naXRzIHByb2Nlc3NvcnMgY2xhc3MgbmFtZXMiCiAgICB9LAogICAgImlvX3Byb2Nlc3Nvcl9wbHVnaW4iOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJJT1Byb2Nlc3NvciBwbHVnaW4gbmFtZSB0byBsb2FkIGF0IHN0YXJ0dXAiCiAgICB9LAogICAgImxvYWRfZm9ybWF0IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICJhdXRvIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSBmb3JtYXQgb2YgbW9kZWwgd2VpZ2h0cyB0byBsb2FkIgogICAgfSwKICAgICJkb3dubG9hZF9kaXIiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEaXJlY3RvcnkgdG8gZG93bmxvYWQgYW5kIGxvYWQgd2VpZ2h0cyIKICAgIH0sCiAgICAic2FmZXRlbnNvcnNfbG9hZF9zdHJhdGVneSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImxhenkiLCAiZWFnZXIiLCAidG9yY2hhbyJdLAogICAgICAiZGVmYXVsdCI6ICJsYXp5IiwKICAgICAgImRlc2NyaXB0aW9uIjogIkxvYWRpbmcgc3RyYXRlZ3kgZm9yIHNhZmV0ZW5zb3JzIHdlaWdodHMiCiAgICB9LAogICAgIm1vZGVsX2xvYWRlcl9leHRyYV9jb25maWciOiB7CiAgICAgICJ0eXBlIjogIm9iamVjdCIsCiAgICAgICJkZWZhdWx0Ijoge30sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFeHRyYSBjb25maWcgZm9yIG1vZGVsIGxvYWRlciIKICAgIH0sCiAgICAiaWdub3JlX3BhdHRlcm5zIjogewogICAgICAidHlwZSI6ICJhcnJheSIsCiAgICAgICJpdGVtcyI6IHsKICAgICAgICAidHlwZSI6ICJzdHJpbmciCiAgICAgIH0sCiAgICAgICJkZWZhdWx0IjogWyJvcmlnaW5hbC8qKi8qIl0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQYXR0ZXJucyB0byBpZ25vcmUgd2hlbiBsb2FkaW5nIG1vZGVsIgogICAgfSwKICAgICJ1c2VfdHFkbV9vbl9sb2FkIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiB0cnVlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHRxZG0gcHJvZ3Jlc3MgYmFyIHdoZW4gbG9hZGluZyIKICAgIH0sCiAgICAicHRfbG9hZF9tYXBfbG9jYXRpb24iOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZWZhdWx0IjogImNwdSIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNYXAgbG9jYXRpb24gZm9yIGxvYWRpbmcgcHl0b3JjaCBjaGVja3BvaW50IgogICAgfSwKICAgICJyZWFzb25pbmdfcGFyc2VyIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiUmVhc29uaW5nIHBhcnNlciB0byB1c2UiCiAgICB9LAogICAgInJlYXNvbmluZ19wYXJzZXJfcGx1Z2luIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiUGF0aCB0byByZWFzb25pbmcgcGFyc2VyIHBsdWdpbiIKICAgIH0sCiAgICAiZGlzdHJpYnV0ZWRfZXhlY3V0b3JfYmFja2VuZCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImV4dGVybmFsX2xhdW5jaGVyIiwgIm1wIiwgInJheSIsICJ1bmkiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkJhY2tlbmQgZm9yIGRpc3RyaWJ1dGVkIG1vZGVsIHdvcmtlcnMiCiAgICB9LAogICAgInBpcGVsaW5lX3BhcmFsbGVsX3NpemUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBwaXBlbGluZSBwYXJhbGxlbCBncm91cHMiCiAgICB9LAogICAgIm1hc3Rlcl9hZGRyIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIxMjcuMC4wLjEiLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzdHJpYnV0ZWQgbWFzdGVyIGFkZHJlc3MiCiAgICB9LAogICAgIm1hc3Rlcl9wb3J0IjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAyOTUwMSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc3RyaWJ1dGVkIG1hc3RlciBwb3J0IgogICAgfSwKICAgICJubm9kZXMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBub2RlcyBmb3IgbXVsdGktbm9kZSBpbmZlcmVuY2UiCiAgICB9LAogICAgIm5vZGVfcmFuayI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJtaW5pbXVtIjogMCwKICAgICAgImRlZmF1bHQiOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzdHJpYnV0ZWQgbm9kZSByYW5rIgogICAgfSwKICAgICJ0ZW5zb3JfcGFyYWxsZWxfc2l6ZSI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJtaW5pbXVtIjogMSwKICAgICAgImRlZmF1bHQiOiAxLAogICAgICAiZGVzY3JpcHRpb24iOiAiTnVtYmVyIG9mIHRlbnNvciBwYXJhbGxlbCBncm91cHMiCiAgICB9LAogICAgImRlY29kZV9jb250ZXh0X3BhcmFsbGVsX3NpemUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBkZWNvZGUgY29udGV4dCBwYXJhbGxlbCBncm91cHMiCiAgICB9LAogICAgImRjcF9rdl9jYWNoZV9pbnRlcmxlYXZlX3NpemUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiS1YgY2FjaGUgaW50ZXJsZWF2ZSBzaXplIGZvciBkZWNvZGUgY29udGV4dCBwYXJhbGxlbGlzbSIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9zaXplIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgIm1pbmltdW0iOiAxLAogICAgICAiZGVmYXVsdCI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOdW1iZXIgb2YgZGF0YSBwYXJhbGxlbCByZXBsaWNhcyIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9yYW5rIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGF0YSBwYXJhbGxlbCByYW5rIG9mIHRoaXMgaW5zdGFuY2UiCiAgICB9LAogICAgImRhdGFfcGFyYWxsZWxfc3RhcnRfcmFuayI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJtaW5pbXVtIjogMCwKICAgICAgImRlc2NyaXB0aW9uIjogIlN0YXJ0aW5nIGRhdGEgcGFyYWxsZWwgcmFuayBmb3Igc2Vjb25kYXJ5IG5vZGVzIgogICAgfSwKICAgICJkYXRhX3BhcmFsbGVsX3NpemVfbG9jYWwiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOdW1iZXIgb2YgZGF0YSBwYXJhbGxlbCByZXBsaWNhcyBvbiB0aGlzIG5vZGUiCiAgICB9LAogICAgImRhdGFfcGFyYWxsZWxfYWRkcmVzcyI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkFkZHJlc3Mgb2YgZGF0YSBwYXJhbGxlbCBjbHVzdGVyIGhlYWQtbm9kZSIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9ycGNfcG9ydCI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQb3J0IGZvciBkYXRhIHBhcmFsbGVsIFJQQyBjb21tdW5pY2F0aW9uIgogICAgfSwKICAgICJkYXRhX3BhcmFsbGVsX2JhY2tlbmQiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJtcCIsICJyYXkiXSwKICAgICAgImRlZmF1bHQiOiAibXAiLAogICAgICAiZGVzY3JpcHRpb24iOiAiQmFja2VuZCBmb3IgZGF0YSBwYXJhbGxlbCIKICAgIH0sCiAgICAiZGF0YV9wYXJhbGxlbF9oeWJyaWRfbGIiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVXNlIGh5YnJpZCBEUCBsb2FkIGJhbGFuY2luZyBtb2RlIgogICAgfSwKICAgICJkYXRhX3BhcmFsbGVsX2V4dGVybmFsX2xiIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlVzZSBleHRlcm5hbCBEUCBsb2FkIGJhbGFuY2luZyBtb2RlIgogICAgfSwKICAgICJlbmFibGVfZXhwZXJ0X3BhcmFsbGVsIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBleHBlcnQgcGFyYWxsZWxpc20gZm9yIE1vRSBtb2RlbHMiCiAgICB9LAogICAgImFsbDJhbGxfYmFja2VuZCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImFsbGdhdGhlcl9yZWR1Y2VzY2F0dGVyIiwgImRlZXBlcF9oaWdoX3Rocm91Z2hwdXQiLCAiZGVlcGVwX2xvd19sYXRlbmN5IiwgImZsYXNoaW5mZXJfYWxsMmFsbHYiLCAibmFpdmUiLCAicHBseCIsICJOb25lIl0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBbGwyQWxsIGJhY2tlbmQgZm9yIE1vRSBleHBlcnQgcGFyYWxsZWwiCiAgICB9LAogICAgImVuYWJsZV9kYm8iOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGR1YWwgYmF0Y2ggb3ZlcmxhcCIKICAgIH0sCiAgICAiZGJvX2RlY29kZV90b2tlbl90aHJlc2hvbGQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVmYXVsdCI6IDMyLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9rZW4gdGhyZXNob2xkIGZvciBkdWFsIGJhdGNoIG92ZXJsYXAgZGVjb2RlIgogICAgfSwKICAgICJkYm9fcHJlZmlsbF90b2tlbl90aHJlc2hvbGQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVmYXVsdCI6IDUxMiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRva2VuIHRocmVzaG9sZCBmb3IgZHVhbCBiYXRjaCBvdmVybGFwIHByZWZpbGwiCiAgICB9LAogICAgImRpc2FibGVfbmNjbF9mb3JfZHBfc3luY2hyb25pemF0aW9uIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkZvcmNlIERQIHN5bmMgdG8gdXNlIEdsb28gaW5zdGVhZCBvZiBOQ0NMIgogICAgfSwKICAgICJlbmFibGVfZXBsYiI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFbmFibGUgZXhwZXJ0IHBhcmFsbGVsaXNtIGxvYWQgYmFsYW5jaW5nIgogICAgfSwKICAgICJlcGxiX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkV4cGVydCBwYXJhbGxlbGlzbSBjb25maWd1cmF0aW9uIgogICAgfSwKICAgICJleHBlcnRfcGxhY2VtZW50X3N0cmF0ZWd5IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsibGluZWFyIiwgInJvdW5kX3JvYmluIl0sCiAgICAgICJkZWZhdWx0IjogImxpbmVhciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFeHBlcnQgcGxhY2VtZW50IHN0cmF0ZWd5IGZvciBNb0UgbGF5ZXJzIgogICAgfSwKICAgICJtYXhfcGFyYWxsZWxfbG9hZGluZ193b3JrZXJzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIHBhcmFsbGVsIGxvYWRpbmcgd29ya2VycyIKICAgIH0sCiAgICAicmF5X3dvcmtlcnNfdXNlX25zaWdodCI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQcm9maWxlIFJheSB3b3JrZXJzIHdpdGggbnNpZ2h0IgogICAgfSwKICAgICJkaXNhYmxlX2N1c3RvbV9hbGxfcmVkdWNlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgY3VzdG9tIGFsbC1yZWR1Y2Uga2VybmVscyIKICAgIH0sCiAgICAid29ya2VyX2NscyI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJGdWxsIG5hbWUgb2YgdGhlIHdvcmtlciBjbGFzcyB0byB1c2UiCiAgICB9LAogICAgIndvcmtlcl9leHRlbnNpb25fY2xzIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiV29ya2VyIGV4dGVuc2lvbiBjbGFzcyBuYW1lIgogICAgfSwKICAgICJlbmFibGVfbXVsdGltb2RhbF9lbmNvZGVyX2RhdGFfcGFyYWxsZWwiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIG11bHRpbW9kYWwgZW5jb2RlciBkYXRhIHBhcmFsbGVsIgogICAgfSwKICAgICJibG9ja19zaXplIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImVudW0iOiBbMSwgOCwgMTYsIDMyLCA2NCwgMTI4LCAyNTZdLAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZSBvZiBhIGNvbnRpZ3VvdXMgY2FjaGUgYmxvY2siCiAgICB9LAogICAgImdwdV9tZW1vcnlfdXRpbGl6YXRpb24iOiB7CiAgICAgICJ0eXBlIjogIm51bWJlciIsCiAgICAgICJtaW5pbXVtIjogMCwKICAgICAgIm1heGltdW0iOiAxLjAsCiAgICAgICJkZWZhdWx0IjogMC45LAogICAgICAiZGVzY3JpcHRpb24iOiAiVGhlIGZyYWN0aW9uIG9mIEdQVSBtZW1vcnkgdG8gYmUgdXNlZCIKICAgIH0sCiAgICAia3ZfY2FjaGVfbWVtb3J5X2J5dGVzIjogewogICAgICAidHlwZSI6IFsiaW50ZWdlciIsICJzdHJpbmciXSwKICAgICAgImRlc2NyaXB0aW9uIjogIlNpemUgb2YgS1YgQ2FjaGUgcGVyIEdQVSBpbiBieXRlcyIKICAgIH0sCiAgICAic3dhcF9zcGFjZSI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAiZGVmYXVsdCI6IDQsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJDUFUgc3dhcCBzcGFjZSBzaXplIChHaUIpIHBlciBHUFUiCiAgICB9LAogICAgImt2X2NhY2hlX2R0eXBlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJiZmxvYXQxNiIsICJmcDgiLCAiZnA4X2RzX21sYSIsICJmcDhfZTRtMyIsICJmcDhfZTVtMiIsICJmcDhfaW5jIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGF0YSB0eXBlIGZvciBLViBjYWNoZSBzdG9yYWdlIgogICAgfSwKICAgICJudW1fZ3B1X2Jsb2Nrc19vdmVycmlkZSI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJPdmVycmlkZSBwcm9maWxlZCBudW1fZ3B1X2Jsb2NrcyIKICAgIH0sCiAgICAiZW5hYmxlX3ByZWZpeF9jYWNoaW5nIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBwcmVmaXggY2FjaGluZyIKICAgIH0sCiAgICAicHJlZml4X2NhY2hpbmdfaGFzaF9hbGdvIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsic2hhMjU2IiwgInNoYTI1Nl9jYm9yIl0sCiAgICAgICJkZWZhdWx0IjogInNoYTI1NiIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJIYXNoIGFsZ29yaXRobSBmb3IgcHJlZml4IGNhY2hpbmciCiAgICB9LAogICAgImNwdV9vZmZsb2FkX2diIjogewogICAgICAidHlwZSI6ICJudW1iZXIiLAogICAgICAibWluaW11bSI6IDAsCiAgICAgICJkZWZhdWx0IjogMCwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSBzcGFjZSBpbiBHaUIgdG8gb2ZmbG9hZCB0byBDUFUiCiAgICB9LAogICAgImNhbGN1bGF0ZV9rdl9zY2FsZXMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGR5bmFtaWMgY2FsY3VsYXRpb24gb2Yga19zY2FsZSBhbmQgdl9zY2FsZSIKICAgIH0sCiAgICAia3Zfc2hhcmluZ19mYXN0X3ByZWZpbGwiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIEtWIHNoYXJpbmcgZmFzdCBwcmVmaWxsIG9wdGltaXphdGlvbiIKICAgIH0sCiAgICAibWFtYmFfY2FjaGVfZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgImZsb2F0MzIiXSwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEYXRhIHR5cGUgZm9yIE1hbWJhIGNhY2hlIgogICAgfSwKICAgICJtYW1iYV9zc21fY2FjaGVfZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJlbnVtIjogWyJhdXRvIiwgImZsb2F0MzIiXSwKICAgICAgImRlZmF1bHQiOiAiYXV0byIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEYXRhIHR5cGUgZm9yIE1hbWJhIFNTTSBzdGF0ZSIKICAgIH0sCiAgICAibWFtYmFfYmxvY2tfc2l6ZSI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJCbG9jayBzaXplIGZvciBtYW1iYSBjYWNoZSIKICAgIH0sCiAgICAia3Zfb2ZmbG9hZGluZ19zaXplIjogewogICAgICAidHlwZSI6ICJudW1iZXIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZSBvZiBLViBjYWNoZSBvZmZsb2FkaW5nIGJ1ZmZlciAoR2lCKSIKICAgIH0sCiAgICAia3Zfb2ZmbG9hZGluZ19iYWNrZW5kIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsibG1jYWNoZSIsICJuYXRpdmUiLCAiTm9uZSJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiQmFja2VuZCBmb3IgS1YgY2FjaGUgb2ZmbG9hZGluZyIKICAgIH0sCiAgICAibGltaXRfbW1fcGVyX3Byb21wdCI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlZmF1bHQiOiB7fSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIG11bHRpLW1vZGFsIGl0ZW1zIHBlciBwcm9tcHQiCiAgICB9LAogICAgImVuYWJsZV9tbV9lbWJlZHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIHBhc3NpbmcgbXVsdGltb2RhbCBlbWJlZGRpbmdzIgogICAgfSwKICAgICJtZWRpYV9pb19rd2FyZ3MiOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZWZhdWx0Ijoge30sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBZGRpdGlvbmFsIGFyZ3MgZm9yIHByb2Nlc3NpbmcgbWVkaWEgaW5wdXRzIgogICAgfSwKICAgICJtbV9wcm9jZXNzb3Jfa3dhcmdzIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgIm9iamVjdCJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiQXJndW1lbnRzIGZvcndhcmRlZCB0byBtdWx0aS1tb2RhbCBwcm9jZXNzb3IiCiAgICB9LAogICAgIm1tX3Byb2Nlc3Nvcl9jYWNoZV9nYiI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgImRlZmF1bHQiOiA0LAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZSAoR2lCKSBvZiBtdWx0aS1tb2RhbCBwcm9jZXNzb3IgY2FjaGUiCiAgICB9LAogICAgImRpc2FibGVfbW1fcHJlcHJvY2Vzc29yX2NhY2hlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgbXVsdGktbW9kYWwgcHJlcHJvY2Vzc29yIGNhY2hlIgogICAgfSwKICAgICJtbV9wcm9jZXNzb3JfY2FjaGVfdHlwZSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImxydSIsICJzaG0iXSwKICAgICAgImRlZmF1bHQiOiAibHJ1IiwKICAgICAgImRlc2NyaXB0aW9uIjogIlR5cGUgb2YgY2FjaGUgZm9yIG11bHRpLW1vZGFsIHByb2Nlc3NvciIKICAgIH0sCiAgICAibW1fc2htX2NhY2hlX21heF9vYmplY3Rfc2l6ZV9tYiI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZWZhdWx0IjogMTI4LAogICAgICAiZGVzY3JpcHRpb24iOiAiTWF4IG9iamVjdCBzaXplIChNaUIpIGluIHNoYXJlZCBtZW1vcnkgY2FjaGUiCiAgICB9LAogICAgIm1tX2VuY29kZXJfdHBfbW9kZSI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImVudW0iOiBbImRhdGEiLCAid2VpZ2h0cyJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiTXVsdGktbW9kYWwgZW5jb2RlciB0ZW5zb3IgcGFyYWxsZWwgbW9kZSIKICAgIH0sCiAgICAibW1fZW5jb2Rlcl9hdHRuX2JhY2tlbmQiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNdWx0aS1tb2RhbCBlbmNvZGVyIGF0dGVudGlvbiBiYWNrZW5kIgogICAgfSwKICAgICJpbnRlcmxlYXZlX21tX3N0cmluZ3MiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGZ1bGx5IGludGVybGVhdmVkIG11bHRpbW9kYWwgcHJvbXB0cyIKICAgIH0sCiAgICAic2tpcF9tbV9wcm9maWxpbmciOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiU2tpcCBtdWx0aW1vZGFsIG1lbW9yeSBwcm9maWxpbmciCiAgICB9LAogICAgInZpZGVvX3BydW5pbmdfcmF0ZSI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAibWF4aW11bSI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJQcnVuaW5nIHJhdGUgZm9yIHZpZGVvIHZpYSBFZmZpY2llbnQgVmlkZW8gU2FtcGxpbmciCiAgICB9LAogICAgImVuYWJsZV9sb3JhIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBoYW5kbGluZyBvZiBMb1JBIGFkYXB0ZXJzIgogICAgfSwKICAgICJtYXhfbG9yYXMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIExvUkEgYWRhcHRlcnMiCiAgICB9LAogICAgIm1heF9sb3JhX3JhbmsiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZW51bSI6IFsxLCA4LCAxNiwgMzIsIDY0LCAxMjgsIDI1NiwgMzIwLCA1MTJdLAogICAgICAiZGVmYXVsdCI6IDE2LAogICAgICAiZGVzY3JpcHRpb24iOiAiTWF4aW11bSBMb1JBIHJhbmsiCiAgICB9LAogICAgImxvcmFfZHR5cGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGF0YSB0eXBlIGZvciBMb1JBIgogICAgfSwKICAgICJtYXhfY3B1X2xvcmFzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gbnVtYmVyIG9mIExvUkFzIHRvIHN0b3JlIGluIENQVSBtZW1vcnkiCiAgICB9LAogICAgImZ1bGx5X3NoYXJkZWRfbG9yYXMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVXNlIGZ1bGx5IHNoYXJkZWQgTG9SQSBsYXllcnMiCiAgICB9LAogICAgImRlZmF1bHRfbW1fbG9yYXMiOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEZWZhdWx0IExvUkEgcGF0aHMgZm9yIHNwZWNpZmljIG1vZGFsaXRpZXMiCiAgICB9LAogICAgInNob3dfaGlkZGVuX21ldHJpY3NfZm9yX3ZlcnNpb24iOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJTaG93IGhpZGRlbiBtZXRyaWNzIHNpbmNlIHNwZWNpZmllZCB2ZXJzaW9uIgogICAgfSwKICAgICJvdGxwX3RyYWNlc19lbmRwb2ludCI6IHsKICAgICAgInR5cGUiOiAic3RyaW5nIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk9wZW5UZWxlbWV0cnkgdHJhY2VzIHRhcmdldCBVUkwiCiAgICB9LAogICAgImNvbGxlY3RfZGV0YWlsZWRfdHJhY2VzIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYWxsIiwgIm1vZGVsIiwgIndvcmtlciIsICJOb25lIiwgIm1vZGVsLHdvcmtlciIsICJtb2RlbCxhbGwiLCAid29ya2VyLG1vZGVsIiwgIndvcmtlcixhbGwiLCAiYWxsLG1vZGVsIiwgImFsbCx3b3JrZXIiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkNvbGxlY3QgZGV0YWlsZWQgdHJhY2VzIGZvciBzcGVjaWZpZWQgbW9kdWxlcyIKICAgIH0sCiAgICAibWF4X251bV9iYXRjaGVkX3Rva2VucyI6IHsKICAgICAgInR5cGUiOiBbImludGVnZXIiLCAic3RyaW5nIl0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJNYXhpbXVtIG51bWJlciBvZiBiYXRjaGVkIHRva2VucyBwZXIgaXRlcmF0aW9uIgogICAgfSwKICAgICJtYXhfbnVtX3NlcXMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiTWF4aW11bSBudW1iZXIgb2Ygc2VxdWVuY2VzIHBlciBpdGVyYXRpb24iCiAgICB9LAogICAgIm1heF9udW1fcGFydGlhbF9wcmVmaWxscyI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heCBjb25jdXJyZW50IHBhcnRpYWwgcHJlZmlsbHMgZm9yIGNodW5rZWQgcHJlZmlsbCIKICAgIH0sCiAgICAibWF4X2xvbmdfcGFydGlhbF9wcmVmaWxscyI6IHsKICAgICAgInR5cGUiOiAiaW50ZWdlciIsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heCBjb25jdXJyZW50IGxvbmcgcHJvbXB0cyBmb3IgY2h1bmtlZCBwcmVmaWxsIgogICAgfSwKICAgICJsb25nX3ByZWZpbGxfdG9rZW5fdGhyZXNob2xkIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9rZW4gdGhyZXNob2xkIGZvciBsb25nIHByZWZpbGwiCiAgICB9LAogICAgIm51bV9sb29rYWhlYWRfc2xvdHMiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAiZGVmYXVsdCI6IDAsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJTbG90cyBmb3Igc3BlY3VsYXRpdmUgZGVjb2RpbmciCiAgICB9LAogICAgInNjaGVkdWxpbmdfcG9saWN5IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiZmNmcyIsICJwcmlvcml0eSJdLAogICAgICAiZGVmYXVsdCI6ICJmY2ZzIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSBzY2hlZHVsaW5nIHBvbGljeSB0byB1c2UiCiAgICB9LAogICAgImVuYWJsZV9jaHVua2VkX3ByZWZpbGwiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVzY3JpcHRpb24iOiAiRW5hYmxlIGNodW5rZWQgcHJlZmlsbCByZXF1ZXN0cyIKICAgIH0sCiAgICAiZGlzYWJsZV9jaHVua2VkX21tX2lucHV0IjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc2FibGUgcGFydGlhbCBzY2hlZHVsaW5nIG9mIG11bHRpbW9kYWwgaXRlbXMiCiAgICB9LAogICAgInNjaGVkdWxlcl9jbHMiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJUaGUgc2NoZWR1bGVyIGNsYXNzIHRvIHVzZSIKICAgIH0sCiAgICAiZGlzYWJsZV9oeWJyaWRfa3ZfY2FjaGVfbWFuYWdlciI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJEaXNhYmxlIGh5YnJpZCBLViBjYWNoZSBtYW5hZ2VyIgogICAgfSwKICAgICJhc3luY19zY2hlZHVsaW5nIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlBlcmZvcm0gYXN5bmMgc2NoZWR1bGluZyIKICAgIH0sCiAgICAic3RyZWFtX2ludGVydmFsIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAxLAogICAgICAiZGVzY3JpcHRpb24iOiAiSW50ZXJ2YWwgZm9yIHN0cmVhbWluZyBpbiB0ZXJtcyBvZiB0b2tlbnMiCiAgICB9LAogICAgImN1ZGFncmFwaF9jYXB0dXJlX3NpemVzIjogewogICAgICAidHlwZSI6ICJhcnJheSIsCiAgICAgICJpdGVtcyI6IHsKICAgICAgICAidHlwZSI6ICJpbnRlZ2VyIgogICAgICB9LAogICAgICAiZGVzY3JpcHRpb24iOiAiU2l6ZXMgdG8gY2FwdHVyZSBjdWRhZ3JhcGgiCiAgICB9LAogICAgIm1heF9jdWRhZ3JhcGhfY2FwdHVyZV9zaXplIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gY3VkYWdyYXBoIGNhcHR1cmUgc2l6ZSIKICAgIH0sCiAgICAiY3VkYWdyYXBoX21vZGUiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJDVURBIGdyYXBoIG1vZGUgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiY3VkYWdyYXBoX251bV9vZl93YXJtdXBzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgImRlZmF1bHQiOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiTnVtYmVyIG9mIENVREEgZ3JhcGggd2FybXVwIGl0ZXJhdGlvbnMiCiAgICB9LAogICAgImN1ZGFncmFwaF9jb3B5X2lucHV0cyI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJDb3B5IENVREEgZ3JhcGggaW5wdXRzIgogICAgfSwKICAgICJjdWRhZ3JhcGhfc3BlY2lhbGl6ZV9sb3JhIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlc2NyaXB0aW9uIjogIlNwZWNpYWxpemUgQ1VEQSBncmFwaHMgZm9yIExvUkEiCiAgICB9LAogICAgInNwZWN1bGF0aXZlX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIlNwZWN1bGF0aXZlIGRlY29kaW5nIGNvbmZpZ3VyYXRpb24iCiAgICB9LAogICAgImt2X3RyYW5zZmVyX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkRpc3RyaWJ1dGVkIEtWIGNhY2hlIHRyYW5zZmVyIGNvbmZpZyIKICAgIH0sCiAgICAia3ZfZXZlbnRzX2NvbmZpZyI6IHsKICAgICAgInR5cGUiOiBbInN0cmluZyIsICJvYmplY3QiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIkV2ZW50IHB1Ymxpc2hpbmcgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiZWNfdHJhbnNmZXJfY29uZmlnIjogewogICAgICAidHlwZSI6IFsic3RyaW5nIiwgIm9iamVjdCJdLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzdHJpYnV0ZWQgRUMgY2FjaGUgdHJhbnNmZXIgY29uZmlnIgogICAgfSwKICAgICJjb21waWxhdGlvbl9jb25maWciOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJ0b3JjaC5jb21waWxlIGFuZCBjdWRhZ3JhcGggY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiYWRkaXRpb25hbF9jb25maWciOiB7CiAgICAgICJ0eXBlIjogIm9iamVjdCIsCiAgICAgICJkZWZhdWx0Ijoge30sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJBZGRpdGlvbmFsIHBsYXRmb3JtLXNwZWNpZmljIGNvbmZpZ3VyYXRpb24iCiAgICB9LAogICAgInN0cnVjdHVyZWRfb3V0cHV0c19jb25maWciOiB7CiAgICAgICJ0eXBlIjogWyJzdHJpbmciLCAib2JqZWN0Il0sCiAgICAgICJkZXNjcmlwdGlvbiI6ICJTdHJ1Y3R1cmVkIG91dHB1dHMgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAiaGVhZGxlc3MiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiUnVuIGluIGhlYWRsZXNzIG1vZGUiCiAgICB9LAogICAgImFwaV9zZXJ2ZXJfY291bnQiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogMSwKICAgICAgImRlc2NyaXB0aW9uIjogIk51bWJlciBvZiBBUEkgc2VydmVyIHByb2Nlc3NlcyIKICAgIH0sCiAgICAiZGlzYWJsZV9sb2dfc3RhdHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRGlzYWJsZSBsb2dnaW5nIHN0YXRpc3RpY3MiCiAgICB9LAogICAgImFnZ3JlZ2F0ZV9lbmdpbmVfbG9nZ2luZyI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJMb2cgYWdncmVnYXRlIHN0YXRpc3RpY3Mgd2l0aCBkYXRhIHBhcmFsbGVsIgogICAgfSwKICAgICJjaGF0X3RlbXBsYXRlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiQ2hhdCB0ZW1wbGF0ZSBmaWxlIHBhdGggb3IgaW5saW5lIHRlbXBsYXRlIgogICAgfSwKICAgICJjaGF0X3RlbXBsYXRlX2NvbnRlbnRfZm9ybWF0IjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsiYXV0byIsICJzdHJpbmciLCAib3BlbmFpIl0sCiAgICAgICJkZWZhdWx0IjogImF1dG8iLAogICAgICAiZGVzY3JpcHRpb24iOiAiQ2hhdCB0ZW1wbGF0ZSBjb250ZW50IGZvcm1hdCIKICAgIH0sCiAgICAidHJ1c3RfcmVxdWVzdF9jaGF0X3RlbXBsYXRlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIlRydXN0IGNoYXQgdGVtcGxhdGUgZnJvbSByZXF1ZXN0IgogICAgfSwKICAgICJyZXNwb25zZV9yb2xlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICJhc3Npc3RhbnQiLAogICAgICAiZGVzY3JpcHRpb24iOiAiUm9sZSBuYW1lIHRvIHJldHVybiBmb3IgZ2VuZXJhdGlvbiIKICAgIH0sCiAgICAiZW5hYmxlX3Byb21wdF90b2tlbnNfZGV0YWlscyI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFbmFibGUgcHJvbXB0IHRva2VucyBkZXRhaWxzIGluIHJlc3BvbnNlIgogICAgfSwKICAgICJlbmFibGVfYXV0b190b29sX2Nob2ljZSI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJFbmFibGUgYXV0byB0b29sIGNob2ljZSIKICAgIH0sCiAgICAiZXhjbHVkZV90b29sc193aGVuX3Rvb2xfY2hvaWNlX25vbmUiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiRXhjbHVkZSB0b29scyB3aGVuIHRvb2xfY2hvaWNlIGlzIG5vbmUiCiAgICB9LAogICAgInRvb2xfY2FsbF9wYXJzZXIiOiB7CiAgICAgICJ0eXBlIjogInN0cmluZyIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJUb29sIGNhbGwgcGFyc2VyIHRvIHVzZSIKICAgIH0sCiAgICAidG9vbF9wYXJzZXJfcGx1Z2luIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVmYXVsdCI6ICIiLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9vbCBwYXJzZXIgcGx1Z2luIHBhdGgiCiAgICB9LAogICAgInRvb2xfc2VydmVyIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZGVzY3JpcHRpb24iOiAiVG9vbCBzZXJ2ZXIgaG9zdDpwb3J0IHBhaXJzIgogICAgfSwKICAgICJlbmFibGVfc2VydmVyX2xvYWRfdHJhY2tpbmciOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiVHJhY2sgc2VydmVyX2xvYWRfbWV0cmljcyBpbiBhcHAgc3RhdGUiCiAgICB9LAogICAgImVuYWJsZV9mb3JjZV9pbmNsdWRlX3VzYWdlIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkluY2x1ZGUgdXNhZ2Ugb24gZXZlcnkgcmVxdWVzdCIKICAgIH0sCiAgICAiZW5hYmxlX3Rva2VuaXplcl9pbmZvX2VuZHBvaW50IjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSAvZ2V0X3Rva2VuaXplcl9pbmZvIGVuZHBvaW50IgogICAgfSwKICAgICJlbmFibGVfbG9nX291dHB1dHMiOiB7CiAgICAgICJ0eXBlIjogImJvb2xlYW4iLAogICAgICAiZGVmYXVsdCI6IGZhbHNlLAogICAgICAiZGVzY3JpcHRpb24iOiAiTG9nIG1vZGVsIG91dHB1dHMiCiAgICB9LAogICAgImxvZ19lcnJvcl9zdGFjayI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJMb2cgc3RhY2sgdHJhY2Ugb2YgZXJyb3IgcmVzcG9uc2VzIgogICAgfSwKICAgICJ0b2tlbnNfb25seSI6IHsKICAgICAgInR5cGUiOiAiYm9vbGVhbiIsCiAgICAgICJkZWZhdWx0IjogZmFsc2UsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJPbmx5IGVuYWJsZSBUb2tlbnMgSW48Pk91dCBlbmRwb2ludCIKICAgIH0sCiAgICAicHJlZW1wdGlvbl9tb2RlIjogewogICAgICAidHlwZSI6ICJzdHJpbmciLAogICAgICAiZW51bSI6IFsic3dhcCIsICJyZWNvbXB1dGUiXSwKICAgICAgImRlc2NyaXB0aW9uIjogIlByZWVtcHRpb24gbW9kZSBkdXJpbmcgbWVtb3J5IHNob3J0YWdlIgogICAgfSwKICAgICJudW1fc2NoZWR1bGVyX3N0ZXBzIjogewogICAgICAidHlwZSI6ICJpbnRlZ2VyIiwKICAgICAgIm1pbmltdW0iOiAxLAogICAgICAiZGVmYXVsdCI6IDEsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJOdW1iZXIgb2Ygc2NoZWR1bGVyIHN0ZXBzIgogICAgfSwKICAgICJtdWx0aV9zdGVwX3N0cmVhbV9vdXRwdXRzIjogewogICAgICAidHlwZSI6ICJib29sZWFuIiwKICAgICAgImRlZmF1bHQiOiBmYWxzZSwKICAgICAgImRlc2NyaXB0aW9uIjogIkVuYWJsZSBtdWx0aS1zdGVwIHN0cmVhbSBvdXRwdXRzIgogICAgfSwKICAgICJyb3BlX3NjYWxpbmciOiB7CiAgICAgICJ0eXBlIjogIm9iamVjdCIsCiAgICAgICJkZXNjcmlwdGlvbiI6ICJSb1BFIHNjYWxpbmcgY29uZmlndXJhdGlvbiIKICAgIH0sCiAgICAicm9wZV90aGV0YSI6IHsKICAgICAgInR5cGUiOiAibnVtYmVyIiwKICAgICAgIm1pbmltdW0iOiAwLAogICAgICAiZGVzY3JpcHRpb24iOiAiUm9QRSB0aGV0YSBwYXJhbWV0ZXIiCiAgICB9LAogICAgIm1heF9zZXFfbGVuX3RvX2NhcHR1cmUiOiB7CiAgICAgICJ0eXBlIjogImludGVnZXIiLAogICAgICAibWluaW11bSI6IDEsCiAgICAgICJkZWZhdWx0IjogODE5MiwKICAgICAgImRlc2NyaXB0aW9uIjogIk1heGltdW0gc2VxdWVuY2UgbGVuZ3RoIGNvdmVyZWQgYnkgQ1VEQSBncmFwaHMiCiAgICB9CiAgfSwKICAiYWRkaXRpb25hbFByb3BlcnRpZXMiOiBmYWxzZQp9Cg=="

    deploy_template:
      kubernetes:
        default: "YXBpVmVyc2lvbjogYXBwcy92MQpraW5kOiBEZXBsb3ltZW50Cm1ldGFkYXRhOgogIG5hbWU6IHt7IC5FbmRwb2ludE5hbWUgfX0KICBuYW1lc3BhY2U6IHt7IC5OYW1lc3BhY2UgfX0KICBsYWJlbHM6CiAgICBlbmdpbmU6IHt7IC5FbmdpbmVOYW1lIH19CiAgICBlbmdpbmVfdmVyc2lvbjoge3sgLkVuZ2luZVZlcnNpb24gfX0KICAgIHJvdXRpbmdfbG9naWM6IHt7IC5Sb3V0aW5nTG9naWMgfX0KICAgIGFwcDogaW5mZXJlbmNlCnNwZWM6CiAgcmVwbGljYXM6IHt7IC5SZXBsaWNhcyB9fQogIHByb2dyZXNzRGVhZGxpbmVTZWNvbmRzOiAxMjAwCiAgc3RyYXRlZ3k6CiAgICB0eXBlOiBSb2xsaW5nVXBkYXRlCiAgICByb2xsaW5nVXBkYXRlOgogICAgICBtYXhVbmF2YWlsYWJsZTogMQogICAgICBtYXhTdXJnZTogMAogIHNlbGVjdG9yOgogICAgbWF0Y2hMYWJlbHM6CiAgICAgIGNsdXN0ZXI6IHt7IC5DbHVzdGVyTmFtZSB9fQogICAgICB3b3Jrc3BhY2U6IHt7IC5Xb3Jrc3BhY2UgfX0KICAgICAgZW5kcG9pbnQ6IHt7IC5FbmRwb2ludE5hbWUgfX0KICAgICAgYXBwOiBpbmZlcmVuY2UKICB0ZW1wbGF0ZToKICAgIG1ldGFkYXRhOgogICAgICBsYWJlbHM6CiAgICAgICAgZW5naW5lOiB7eyAuRW5naW5lTmFtZSB9fQogICAgICAgIGVuZ2luZV92ZXJzaW9uOiB7eyAuRW5naW5lVmVyc2lvbiB9fQogICAgICAgIGNsdXN0ZXI6IHt7IC5DbHVzdGVyTmFtZSB9fQogICAgICAgIHdvcmtzcGFjZToge3sgLldvcmtzcGFjZSB9fQogICAgICAgIGVuZHBvaW50OiB7eyAuRW5kcG9pbnROYW1lIH19CiAgICAgICAgcm91dGluZ19sb2dpYzoge3sgLlJvdXRpbmdMb2dpYyB9fQogICAgICAgIGFwcDogaW5mZXJlbmNlCiAgICBzcGVjOgogICAgICBhZmZpbml0eToKICAgICAgICBwb2RBbnRpQWZmaW5pdHk6CiAgICAgICAgICBwcmVmZXJyZWREdXJpbmdTY2hlZHVsaW5nSWdub3JlZER1cmluZ0V4ZWN1dGlvbjoKICAgICAgICAgICAgLSB3ZWlnaHQ6IDEwMAogICAgICAgICAgICAgIHBvZEFmZmluaXR5VGVybToKICAgICAgICAgICAgICAgIGxhYmVsU2VsZWN0b3I6CiAgICAgICAgICAgICAgICAgIG1hdGNoRXhwcmVzc2lvbnM6CiAgICAgICAgICAgICAgICAgICAgLSBrZXk6IGVuZHBvaW50CiAgICAgICAgICAgICAgICAgICAgICBvcGVyYXRvcjogSW4KICAgICAgICAgICAgICAgICAgICAgIHZhbHVlczoKICAgICAgICAgICAgICAgICAgICAgICAgLSB7eyAuRW5kcG9pbnROYW1lIH19CiAgICAgICAgICAgICAgICB0b3BvbG9neUtleTogImt1YmVybmV0ZXMuaW8vaG9zdG5hbWUiCiAgICAgIHt7LSBpZiAuTm9kZVNlbGVjdG9yIH19CiAgICAgIG5vZGVTZWxlY3RvcjoKICAgICAgICB7ey0gcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5Ob2RlU2VsZWN0b3IgfX0KICAgICAgICB7eyAka2V5IH19OiB7eyAkdmFsdWUgfX0KICAgICAgICB7ey0gZW5kIH19CiAgICAgIHt7LSBlbmQgfX0KICAgICAge3stIGlmIC5JbWFnZVB1bGxTZWNyZXQgfX0KICAgICAgaW1hZ2VQdWxsU2VjcmV0czoKICAgICAgICAtIG5hbWU6IHt7IC5JbWFnZVB1bGxTZWNyZXQgfX0KICAgICAge3stIGVuZCB9fQoKICAgICAge3stIGlmIC5Wb2x1bWVzIH19CiAgICAgIHZvbHVtZXM6Cnt7IC5Wb2x1bWVzIHwgdG9ZYW1sIHwgaW5kZW50IDYgfX0KICAgICAge3stIGVuZCB9fQogICAgICBpbml0Q29udGFpbmVyczoKICAgICAgICAtIG5hbWU6IG1vZGVsLWRvd25sb2FkZXIKICAgICAgICAgIGltYWdlOiB7eyAuSW1hZ2VQcmVmaXggfX0vbmV1dHJlZS9uZXV0cmVlLXJ1bnRpbWU6e3sgLk5ldXRyZWVWZXJzaW9uIH19CiAgICAgICAgICBjb21tYW5kOgogICAgICAgICAgICAtIGJhc2gKICAgICAgICAgICAgLSAtYwogICAgICAgICAgYXJnczoKICAgICAgICAgICAgLSA+LQogICAgICAgICAgICAgIHB5dGhvbjMgLW0gbmV1dHJlZS5kb3dubG9hZGVyCiAgICAgICAgICAgICAgLS1uYW1lPSJ7eyAuTW9kZWxBcmdzLm5hbWUgfX0iCiAgICAgICAgICAgICAgLS1yZWdpc3RyeV90eXBlPSJ7eyAuTW9kZWxBcmdzLnJlZ2lzdHJ5X3R5cGUgfX0iCiAgICAgICAgICAgICAgLS1yZWdpc3RyeV9wYXRoPSJ7eyAuTW9kZWxBcmdzLnJlZ2lzdHJ5X3BhdGggfX0iCiAgICAgICAgICAgICAgLS1wYXRoPSJ7eyAuTW9kZWxBcmdzLnBhdGggfX0iCiAgICAgICAgICAgICAgLS12ZXJzaW9uPSJ7eyAuTW9kZWxBcmdzLnZlcnNpb24gfX0iCiAgICAgICAgICAgICAgLS1maWxlPSJ7eyAuTW9kZWxBcmdzLmZpbGUgfX0iCiAgICAgICAgICAgICAgLS10YXNrPSJ7eyAuTW9kZWxBcmdzLnRhc2sgfX0iCiAgICAgICAgICBlbnY6CiAgICAgICAgICAge3sgcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5FbnYgfX0KICAgICAgICAgICAtIG5hbWU6IHt7ICRrZXkgfX0KICAgICAgICAgICAgIHZhbHVlOiAie3sgJHZhbHVlIH19IgogICAgICAgICAgIHt7IGVuZCB9fQogICAgICAgICAge3stIGlmIC5Wb2x1bWVNb3VudHMgfX0KICAgICAgICAgIHZvbHVtZU1vdW50czoKe3sgLlZvbHVtZU1vdW50cyB8IHRvWWFtbCB8IGluZGVudCAxMCB9fQogICAgICAgICAge3stIGVuZCB9fQoKICAgICAgY29udGFpbmVyczoKICAgICAgICAtIG5hbWU6IHt7IC5FbmdpbmVOYW1lIH19CiAgICAgICAgICBpbWFnZToge3sgLkltYWdlUHJlZml4IH19L3t7IC5JbWFnZVJlcG8gfX06e3sgLkltYWdlVGFnIH19CiAgICAgICAgICBjb21tYW5kOgogICAgICAgICAgLSB2bGxtCiAgICAgICAgICAtIHNlcnZlCiAgICAgICAgICAtIHt7IC5Nb2RlbEFyZ3MucGF0aCB9fQogICAgICAgICAgLSAtLWhvc3QKICAgICAgICAgIC0gIjAuMC4wLjAiCiAgICAgICAgICAtICItLXBvcnQiCiAgICAgICAgICAtICI4MDAwIgogICAgICAgICAgLSAtLXNlcnZlZC1tb2RlbC1uYW1lCiAgICAgICAgICAtIHt7IC5Nb2RlbEFyZ3Muc2VydmVfbmFtZSB9fQogICAgICAgICAgLSAtLXRhc2sKICAgICAgICAgIHt7LSBpZiBlcSAuTW9kZWxBcmdzLnRhc2sgInRleHQtZW1iZWRkaW5nIiB9fQogICAgICAgICAgLSBlbWJlZGRpbmcKICAgICAgICAgIHt7LSBlbHNlIGlmIGVxIC5Nb2RlbEFyZ3MudGFzayAidGV4dC1nZW5lcmF0aW9uIiB9fQogICAgICAgICAgLSBnZW5lcmF0ZQogICAgICAgICAge3stIGVsc2UgaWYgZXEgLk1vZGVsQXJncy50YXNrICJ0ZXh0LXJlcmFuayIgfX0KICAgICAgICAgIC0gc2NvcmUKICAgICAgICAgIHt7LSBlbHNlIH19CiAgICAgICAgICAtIHt7IC5Nb2RlbEFyZ3MudGFzayB9fQogICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAge3stIGlmIC5FbmdpbmVBcmdzIH19CiAgICAgICAgICB7ey0gcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5FbmdpbmVBcmdzIH19CiAgICAgICAgICAtIC0te3sgJGtleSB9fQogICAgICB7ey0gaWYgbmUgKHByaW50ZiAiJXYiICR2YWx1ZSkgInRydWUifX0KICAgICAgICAgIC0gInt7ICR2YWx1ZSB9fSIKICAgICAge3stIGVuZCB9fQogICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAgcmVzb3VyY2VzOgogICAgICAgICAgICBsaW1pdHM6CiAgICAgICAgICAgICAge3stIHJhbmdlICRrZXksICR2YWx1ZSA6PSAuUmVzb3VyY2VzIH19CiAgICAgICAgICAgICAge3sgJGtleSB9fToge3sgJHZhbHVlIH19CiAgICAgICAgICAgICAge3stIGVuZCB9fQogICAgICAgICAgICByZXF1ZXN0czoKICAgICAgICAgICAgICB7ey0gcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5SZXNvdXJjZXMgfX0KICAgICAgICAgICAgICB7eyAka2V5IH19OiB7eyAkdmFsdWUgfX0KICAgICAgICAgICAgICB7ey0gZW5kIH19CiAgICAgICAgICBlbnY6CiAgICAgICAgICAge3sgcmFuZ2UgJGtleSwgJHZhbHVlIDo9IC5FbnYgfX0KICAgICAgICAgICAtIG5hbWU6IHt7ICRrZXkgfX0KICAgICAgICAgICAgIHZhbHVlOiAie3sgJHZhbHVlIH19IgogICAgICAgICAgIHt7IGVuZCB9fQogICAgICAgICAgcG9ydHM6CiAgICAgICAgICAgIC0gY29udGFpbmVyUG9ydDogODAwMAogICAgICAgICAgc3RhcnR1cFByb2JlOgogICAgICAgICAgICBodHRwR2V0OgogICAgICAgICAgICAgIHBhdGg6IC9oZWFsdGgKICAgICAgICAgICAgICBwb3J0OiA4MDAwCiAgICAgICAgICAgIGluaXRpYWxEZWxheVNlY29uZHM6IDUKICAgICAgICAgICAgdGltZW91dFNlY29uZHM6IDUKICAgICAgICAgICAgcGVyaW9kU2Vjb25kczogMTAKICAgICAgICAgICAgc3VjY2Vzc1RocmVzaG9sZDogMQogICAgICAgICAgICBmYWlsdXJlVGhyZXNob2xkOiAxMjAKICAgICAgICAgIHJlYWRpbmVzc1Byb2JlOgogICAgICAgICAgICBodHRwR2V0OgogICAgICAgICAgICAgIHBhdGg6IC9oZWFsdGgKICAgICAgICAgICAgICBwb3J0OiA4MDAwCiAgICAgICAgICAgIGluaXRpYWxEZWxheVNlY29uZHM6IDUKICAgICAgICAgICAgdGltZW91dFNlY29uZHM6IDUKICAgICAgICAgICAgcGVyaW9kU2Vjb25kczogMTAKICAgICAgICAgICAgc3VjY2Vzc1RocmVzaG9sZDogMQogICAgICAgICAgICBmYWlsdXJlVGhyZXNob2xkOiAzCiAgICAgICAgICB7ey0gaWYgLlZvbHVtZU1vdW50cyB9fQogICAgICAgICAgdm9sdW1lTW91bnRzOgp7eyAuVm9sdW1lTW91bnRzIHwgdG9ZYW1sIHwgaW5kZW50IDEwIH19CiAgICAgICAgICB7ey0gZW5kIH19"


    supported_tasks:
      - "generate"

    images:
      nvidia_gpu:
        image_name: "vllm/vllm-openai"
        tag: "v0.17.1"
```

 
