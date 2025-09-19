---
title: Nutanix_NKP-Notes
description: Nutanix_NKP-Notes
slug: Nutanix_NKP-Notes
date: 2025-09-19T06:07:16+08:00
categories:
    - Knowledge Base Category
tags:
    - NKP
    - Nutanix
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# NKP Notes



## Setting

### 取得 Kubeconfig

```
nkp get clusters  -A
WORKSPACE          	NAME        	KUBECONFIG                      	STATUS 
dev                	dev-cluster 	dev-cluster-kubeconfig          	Joined	
kommander-workspace	host-cluster	kommander-self-attach-kubeconfig	Joined

nkp get kubeconfig --cluster-name CLUSTER_NAME > CLUSTER_NAME.conf

nkp get kubeconfig --cluster-name dev-cluster -n dev > dev-cluster.confg
```



### Dedicated URL

```
kubectl get workspace

export WORKSPACE_NAME=kommander-workspace

echo https://$(kubectl get kommandercluster -n kommander host-cluster -o jsonpath='{ .status.ingress.address }')/token/landing/${WORKSPACE_NAME}
```

### Local Users

Create password

```
htpasswd -bnBC 10 "nkpadmin" 'Nutanix/4u' | tr -d ':\n' && echo

nkpadmin$2y$10$fYmKxCx8jeLFOfJHRLp8QOA.x6pGH/yhwMN6.Wqh06enN3B1SWi4i
```

Create Configmaps

dex-localusers-configmap.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex-localusers
  namespace: kommander
data:
  values.yaml: |
    config:
      staticPasswords:
      - email: nkpadmin
        hash: $2y$10$nYTbqF3UUP8arnDtfEpVP.UEL77EbkKRKn7EuFTo8LP13WXA0cr8W
      - email: nkpuser01
        hash: $2y$10$q9DT8341q.T4E/2RwjagGuC/ayxvWt7KWs1HPSrW3x/I9Heb2U/jO
      - email: nkpuser02
        hash: $2y$10$kblg6IUe1vRz4hi1OGrpFOZhrAUeGFw0xYi5RfZaxeTIjqbqlfWm.
      - email: nkpuser03
        hash: $2y$10$EHcX0VulcM2Xmwl3g6hjHuoN5vC/UJ79Oax.n2EbkwZY.DO6Yqs3W
      - email: nkpuser04
        hash: $2y$10$GlHMgNu922y0Onx5pf3eRe1rMb2/E3XT2LISLT0gPz8StrPbR.Cjy
```

kubectl apply -f dex-localusers-configmap.yaml

kubectl edit -n kommander appdeployment dex (修改 spec 的地方)

```
apiVersion: apps.kommander.d2iq.io/v1alpha3
kind: AppDeployment
metadata:
...
spec:
  appRef:
    kind: ClusterApp
    name: dex-2.11.1
  clusterConfigOverrides:
  - clusterSelector:
      matchExpressions:
      - key: kommander.d2iq.io/cluster-name
        operator: In
        values:
        - host-cluster
    configMapName: dex-kommander-overrides
  configOverrides: # Copy and paste this section.
    name: dex-localusers
status:
```

#### RBAC

都可以透過 yaml 去定義，或是先用 admin 的之後用 UI 來建立角色權限

##### Admin

dex-rbac-admin.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: nkpadmin
```

kubectl apply -f dex-rbac-admin.yaml

##### Kubernetes Dashboard

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: nkpadmin
```

##### UI (Developer)

針對特定 Workspace 及 Project 來設定

1. Create Group (workspace)

   ![image-20250715102606485](https://kenkenny.synology.me:5543/images/2025/07/image-20250715102606485.png)

2. Create RoleBindings (workspace)

   ![image-20250715102641426](https://kenkenny.synology.me:5543/images/2025/07/image-20250715102641426.png)

   Kommander Workspace View --> 可以登入這個 workspace ，但沒東西可以看
   Workspace Dashboard View --> 可以查看 workspace 的 Application 及 Dashboard

   Workspace KubernetesDashboard View Role --> 預設沒有 namespace 可以看，所以在 Project 層級要再加上 App Deployer

   Grafana 、 Prometheus、Traefik 給 View 權限就都可以看得到

3. Project RoleBinding 設定

   ![image-20250715102923562](https://kenkenny.synology.me:5543/images/2025/07/image-20250715102923562.png)

   developer 只能看 GitOps 不能編輯，看不到 Secret，針對平臺應用程式也只能看，不能新增

   ![image-20250715103510832](https://kenkenny.synology.me:5543/images/2025/07/image-20250715103510832.png)

   ![image-20250715103810986](https://kenkenny.synology.me:5543/images/2025/07/image-20250715103810986.png)

   ![image-20250715104052083](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104052083.png)

4. Kubernetes Dashboard 及 Kiali 只能看到該 Project 的資訊

   ![image-20250715103104835](https://kenkenny.synology.me:5543/images/2025/07/image-20250715103104835.png)

   ![image-20250715103219741](https://kenkenny.synology.me:5543/images/2025/07/image-20250715103219741.png)

5. Grafana & Traefik 都可以看得到

   ![image-20250715103921972](https://kenkenny.synology.me:5543/images/2025/07/image-20250715103921972.png)

   Grafana 可以再把權限調整成 View ，預設是給 admin

   ![image-20250715120412212](https://kenkenny.synology.me:5543/images/2025/07/image-20250715120412212.png)

   ![image-20250715103941262](https://kenkenny.synology.me:5543/images/2025/07/image-20250715103941262.png)

6. 取得 kubeconfig 方法，登入 Dedicated URL，可以選擇進入 NKP Dashboard 或是 Generate Kubectl Config 或是安裝插件

   ![image-20250715104158700](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104158700.png)

   選擇 Generate Kubectl Config

   ![image-20250715104312603](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104312603.png)

   需要再登入一次

   ![image-20250715104347559](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104347559.png)

   確認群組跟名稱，後面就依照他提供的設定在自己的電腦或跳板機執行完成就可以連線了

   ![image-20250715104509372](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104509372.png)

   ![image-20250715104416484](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104416484.png)

   

7. 從 Command Line 查看權限是否被正確套用，基本上除了 Project 內可以操作之外，其他都看不到也無法編輯

   ![image-20250715104811157](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104811157.png)

8. 補充 Pod exec Role 可以執行 kubectl -n namespace exec pod  -it -- /bin/bash

   ![image-20250722153023482](https://kenkenny.synology.me:5543/images/2025/07/image-20250722153023482.png)

9. 補充： 有安裝好插件的話 只需要輸入 kubectl get pods -n namespace 就會跳出 url ，把 url 貼到瀏覽器做登入就可以連線成功

   ![image-20250722153210936](https://kenkenny.synology.me:5543/images/2025/07/image-20250722153210936.png)

##### 確認  Token Expire

確認 Generate 出來的 kubeconfig 過期時間

```
cat ~/.kube/config | grep token | awk '{print $2}' | cut -d '.' -f2 | base64 -d | jq '.exp' | xargs -I{} date -d @{}
```





### Quota

![image-20250715104917757](https://kenkenny.synology.me:5543/images/2025/07/image-20250715104917757.png)

admin 從 UI 可以針對 Project 設定 Quotas 跟 Limit，要注意 project 如果 istio-injection: enabled 的話

啟動任何 Container 都會加上 istio-proxy ，預設會設定 Limits: cpu 2 , memory 1Gi，

所以 yaml 內要另外指定 container 的 resource，或是加上 sidecar.istio.io/inject: "false" ，讓 istio 不要偵測

![image-20250715105339778](https://kenkenny.synology.me:5543/images/2025/07/image-20250715105339778.png)

kubectl get resourcequota -n namespace

```
kubectl get resourcequota -n namespace
kubectl describe resourcequota -n namespace
```

![image-20250715112217564](https://kenkenny.synology.me:5543/images/2025/07/image-20250715112217564.png)

```
kubectl --kubeconfig=dev-cluster.conf get quota kommander -n demo-dev -o yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  creationTimestamp: "2025-07-14T05:25:04Z"
  labels:
    kubefed.io/managed: "true"
  name: kommander
  namespace: demo-dev
  resourceVersion: "259671"
  uid: 814a7df7-4891-4710-8b66-97b6a1edbf47
spec:
  hard:
    count/pods: "10"
    count/services: "5"
    count/services.loadbalancers: "0"
    count/services.nodeports: "0"
    limits.cpu: "8"
    limits.memory: 8000Mi
    requests.storage: 50Gi
status:
  hard:
    count/pods: "10"
    count/services: "5"
    count/services.loadbalancers: "0"
    count/services.nodeports: "0"
    limits.cpu: "8"
    limits.memory: 8000Mi
    requests.storage: 50Gi
  used:
    count/pods: "3"
    count/services: "2"
    limits.cpu: 4600m
    limits.memory: 3328Mi
    requests.storage: 17Gi
```

### Object Storage

會用到預設 Ceph Storage 的有 Grafana-Loki、Velero、Kubecost、NKP Insight，都可以改用 Nutanix Object 來節省 Ceph 所需的資源

Harbor 也需要 Object Storage ，不過可以用 Nutanix COSI 來安裝，所以設定上不用調整太多

Kubecost 跟 NKP Insight 應該都也可以透過 COSI 安裝

Grafana-Loki、Velero 這兩項是在 Kommander install yaml 裡面去修改參數後再重新部署

ps.  NKP 2.15 從 UI 刪除 rook-ceph 以及 velero、grafana-loki 時會自動長出來，必須要用 kommander-install.yaml 去修改才會生效



#### Grafana-Loki

Secret for object storage 必須要 base64 編碼後的值

```
-- Object Key
Username: nkpsa@nutanixlab.local
Access Key: _y1gdwax1EwopQXQo1EXM1egoBYijgF0
Secret Key: 5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5
Display Name: nkpsa
--

echo -n _y1gdwax1EwopQXQo1EXM1egoBYijgF0 |base64
X3kxZ2R3YXgxRXdvcFFYUW8xRVhNMWVnb0JZaWpnRjA=

echo -n 5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5 |base64
NWo4eEdDSzE2LTFfM0hIcmJtbVhWcDM5ZTFDaTk2YjU=
```

dkp-loki.yaml ， 名稱必須要是 dkp-loki

```
apiVersion: v1
kind: Secret
metadata:
  name: dkp-loki
  namespace: kommander
data:
  AWS_ACCESS_KEY_ID: X3kxZ2R3YXgxRXdvcFFYUW8xRVhNMWVnb0JZaWpnRjA=
  AWS_SECRET_ACCESS_KEY: NWo4eEdDSzE2LTFfM0hIcmJtbVhWcDM5ZTFDaTk2YjU=
```

部署及驗證 Secret

```
# 部署 secret 
kubectl apply -f dkp-loki-secret.yaml 

# 驗證 secret 內的 key 是否正確
export AWS_ACCESS_KEY_ID=`kubectl get secret -n kommander dkp-loki -o 'jsonpath={.data.AWS_ACCESS_KEY_ID}' | base64 --decode;echo`

export AWS_SECRET_ACCESS_KEY=`kubectl get secret -n kommander dkp-loki -o 'jsonpath={.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode;echo`
```

UI 部署 修改 yaml，url 可以是 IP 

```
loki:
  structuredConfig:
    storage_config:
      aws:
        s3: "s3://ntnxobject.nutanixlab.local/nkp-loki"
        bucketnames: "nkp-loki"
        endpoint: "http://ntnxobject.nutanixlab.local"
        insecure: true
        sse_encryption: false
        s3forcepathstyle: true
        region: "us-east-1"
    table_manager:
      retention_deletes_enabled: true
      retention_period: 90d
    limits_config:
      retention_period: 2160h  # 90days in hours
  extraEnvFrom:
    - secretRef:
        name: dkp-loki
```

CLI 部署 修改 kommander-install.yaml

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



#### Velero

Secret For Object Storage

```
-- Object Key
Username: nkpsa@nutanixlab.local
Access Key: _y1gdwax1EwopQXQo1EXM1egoBYijgF0
Secret Key: 5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5
Display Name: nkpsa
--

echo -n _y1gdwax1EwopQXQo1EXM1egoBYijgF0 |base64
X3kxZ2R3YXgxRXdvcFFYUW8xRVhNMWVnb0JZaWpnRjA=

echo -n 5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5 |base64
NWo4eEdDSzE2LTFfM0hIcmJtbVhWcDM5ZTFDaTk2YjU=

echo -n "[default]
aws_access_key_id=_y1gdwax1EwopQXQo1EXM1egoBYijgF0
aws_secret_access_key=5j8xGCK16-1_3HHrbmmXVp39e1Ci96b5" | base64 -w0
W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkPV95MWdkd2F4MUV3b3BRWFFvMUVYTTFlZ29CWWlqZ0YwCmF3c19zZWNyZXRfYWNjZXNzX2tleT01ajh4R0NLMTYtMV8zSEhyYm1tWFZwMzllMUNpOTZiNQ==
```

dkp-velero.yaml ， 名稱必須要是 dkp-velero

```
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
```

veler-secret.yaml，另外要再新增一個名為 velero 的 secert

```
apiVersion: v1
kind: Secret
metadata:
  name: velero
  namespace: kommander
type: Opaque
data:
  cloud: W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkPV95MWdkd2F4MUV3b3BRWFFvMUVYTTFlZ29CWWlqZ0YwCmF3c19zZWNyZXRfYWNjZXNzX2tleT01ajh4R0NLMTYtMV8zSEhyYm1tWFZwMzllMUNpOTZiNQ==
```

部署及驗證

```
# 部署 secret 
kubectl apply -f dkp-velero-secret.yaml

# 驗證 secret 內的 key 是否正確
export AWS_ACCESS_KEY_ID=`kubectl get secret -n kommander dkp-velero -o 'jsonpath={.data.AWS_ACCESS_KEY_ID}' | base64 --decode;echo`

export AWS_SECRET_ACCESS_KEY=`kubectl get secret -n kommander dkp-velero -o 'jsonpath={.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode;echo`

kubectl get secret velero -n kommander -o jsonpath="{.data}"
```

UI 部署 修改 yaml，url 可以是 IP 

```
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
        extraEnv:
          - name: AWS_ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: dkp-velero
                key: AWS_ACCESS_KEY_ID
          - name: AWS_SECRET_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: dkp-velero
                key: AWS_SECRET_ACCESS_KEY   
          - name: AWS_SDK_LOAD_CONFIG
            value: "true"
      defaultVolumesToFsBackup: true  # 是讓 Velero 預設對所有沒有快照的 PVC 使用 Kopia 備份
      extraEnvFrom:
        - secretRef:
            name: dkp-velero
```

CLI 部署 修改 kommander-install.yaml ，CLI 部署可以順便把不需要的刪除，例如 rook-ceph

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

![image-20250915104034092](https://kenkenny.synology.me:5543/images/2025/09/image-20250915104034092.png)

![image-20250915113839919](https://kenkenny.synology.me:5543/images/2025/09/image-20250915113839919.png)





#### Nutanix COSI

安裝方式是先建立好 Nutanix Object Storage 之後，透過 NKP 的 UI 填入所需的資訊之後就可以安裝連線完成

![image-20250903144326389](https://kenkenny.synology.me:5543/images/2025/09/image-20250903144326389.png)

![image-20250903144344482](https://kenkenny.synology.me:5543/images/2025/09/image-20250903144344482.png)

![image-20250903144359919](https://kenkenny.synology.me:5543/images/2025/09/image-20250903144359919.png)

相關的 Pod 以及 Secret

![image-20250903144516685](https://kenkenny.synology.me:5543/images/2025/09/image-20250903144516685.png)

資源解釋：

Bucket

```
透過 COSI 建立的 bucket，類似於 PV 的概念

$ kubectl get bucket
NAME                                                   AGE
cosi-nutanix-nkpcb51fd80-4957-4528-b758-95d572527386   133d
```

BucketClaim

```
類似於 PVC 的概念，建立後就可以產生 bucket

$ kubectl get bucketclaim -A
NAMESPACE    NAME                 AGE
kommander    cosi-kubecost        55d
ncr-system   cosi-harbor-bucket   133d
```

BucketClass

```
類似於 StorageClass，必須有 Class 才有辦法建立對應的 bucket,bucketclaim

$ kubectl get BucketClass
NAME               AGE
cosi-nutanix-nkp   133d
```

其他驗證相關的 BucketAccess、BucketAccessClass

```
Since Object Storage is always authenticated, and over the network, access credentials are required to access buckets. The two APIs, namely, BucketAccess and BucketAccessClass are used to denote access credentials and policies for authentication.

$ kubectl get BucketAccess -A
NAMESPACE    NAME                 AGE
kommander    cosi-kubecost        55d
ncr-system   cosi-harbor-bucket   133d

$ kubectl get BucketAccessClass -A
NAME               AGE
cosi-nutanix-nkp   133d
```



#### NKP Insight

NKP Insight 可以整合外部 Object Storage，在 UI 上面如果沒有 rook ceph 是無法直接點選安裝 (2.14)

必須要透過 CLI 方式安裝 secret + configmaps + appdeployments (online)

NKP 2.16 之後就可以透過 UI 點選安裝，只需要輸入 COSI 的資訊即可

![image-20250903145015330](https://kenkenny.synology.me:5543/images/2025/09/image-20250903145015330.png)

確認 Nutanix Object 已經安裝完成

```
[root@ken-rhel9 ~]# kubectl get BucketClass
NAME               AGE
cosi-nutanix-nkp   2m18s
[root@ken-rhel9 ~]# kubectl describe BucketClass
Name:             cosi-nutanix-nkp
Namespace:        
Labels:           app.kubernetes.io/managed-by=Helm
                  helm.toolkit.fluxcd.io/name=cosi-resources-nutanix
                  helm.toolkit.fluxcd.io/namespace=kommander
Annotations:      meta.helm.sh/release-name: cosi-resources-nutanix
                  meta.helm.sh/release-namespace: kommander
API Version:      objectstorage.k8s.io/v1alpha1
Deletion Policy:  Delete
Driver Name:      ntnx.objectstorage.k8s.io
Kind:             BucketClass
Metadata:
  Creation Timestamp:  2025-09-18T09:22:20Z
  Generation:          1
  Resource Version:    14804937
  UID:                 e9ad76b0-76c3-49a5-94d1-ee0a59f0eb2d
Events:                <none>
[root@ken-rhel9 ~]# kubectl get BucketAccessClass -A
NAME               AGE
cosi-nutanix-nkp   2m35s
```



Nutanix Object COSI 設定，TTL 預設是 168h 7 day，可以依照需求去更改

```
backend:
  s3:
    cosi:
      className: cosi-nutanix-nkp
      accessClassName: cosi-nutanix-nkp
cleanup:
    insightsTTL: "720h"
```

如果有先建立 Bucket 的話設定

```
backend:
  s3:
    bucketName: <existing bucket name>
    cosi:
      className: cosi-nutanix-nkp
      accessClassName: cosi-nutanix-nkp
      driverName: ntnx.objectstorage.k8s.io
cleanup:
    insightsTTL: "168h" # 7 day
```

2.16 截圖

![image-20250918173448102](https://kenkenny.synology.me:5543/images/2025/09/image-20250918173448102.png)

![image-20250918174130049](https://kenkenny.synology.me:5543/images/2025/09/image-20250918174130049.png)

已自動透過 COSI 創建 Bucket

![image-20250918174233554](https://kenkenny.synology.me:5543/images/2025/09/image-20250918174233554.png)

Insight 可以看到告警資訊

![image-20250918174323805](https://kenkenny.synology.me:5543/images/2025/09/image-20250918174323805.png)

點進 Detail 可以看到詳細資料，並且還有 Solutions

![image-20250918174403031](https://kenkenny.synology.me:5543/images/2025/09/image-20250918174403031.png)

也可以點到 Dashboard 直接導到 Kubernetes Dashboard

![image-20250918174704503](https://kenkenny.synology.me:5543/images/2025/09/image-20250918174704503.png)

Cluster 頁面會有前幾個 Insight 的 Alert

![image-20250918175010542](https://kenkenny.synology.me:5543/images/2025/09/image-20250918175010542.png)







### Prometheus

https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_15:top-monitoring-and-alerts-c.html

https://portal.nutanix.com/page/documents/kbs/details?targetId=kA0VO0000001mSP0AY

https://prometheus.io/docs/prometheus/latest/storage/

預設 prometheus 的 retention policy 是10天，NKP 預設安裝會開 100Gi 的 PVC ，如想要提升就要修改 Configmaps

順序：新增 configmaps --> 修改 appdeployments --> 擴充 pvc  擴充多少要參照原廠文件

原本預設值 

```
kubectl get appdeployment kube-prometheus-stack -n kommander -o yaml

  valuesFrom:
  - kind: ConfigMap
    name: kube-prometheus-stack-69.1.2-d2iq-defaults
  - kind: ConfigMap
    name: kube-prometheus-stack-mgmt-overrides
    optional: true
```

kube-prometheus-stack-69.1.2-d2iq-defaults

```
        externalUrl: "/dkp/prometheus"
        storageSpec:
          volumeClaimTemplate:
            metadata:
              name: db
            spec:
              accessModes: ["ReadWriteOnce"]
              # 100Gi is the default size for the chart
              resources:
                requests:
                  storage: 100Gi
        resources:
          limits:
            cpu: 2000m
            memory: 10922Mi
          requests:
            cpu: 1000m
            memory: 4000Mi
```

以下為從原廠 override configmaps 範例修改的內容，另外新增了 retention (天數)、retentionSize (容量大小)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-prometheus-stack-overrides
  namespace: kommander
data:
  values.yaml: |
    ---
    prometheus:
      prometheusSpec:
        resources:
          limits:
            cpu: "4"
            memory: "10Gi"
          requests:
            cpu: "2"
            memory: "4Gi"
        retention: 90d
        retentionSize: "816GiB"
      storageSpec:
        volumeClaimTemplate:
          spec:
            resources:
              requests:
                storage: "960GiB"
```

部署 configmaps 並讓 helm 重新套用

```
kubectl apply -f kube-prometheus-stack-overrides.yaml 

kubectl get appdeployment kube-prometheus-stack -n kommander

kubectl edit appdeployment kube-prometheus-stack -n kommander

# 新增 

spec:
  appRef:
    kind: ClusterApp
    name: kube-prometheus-stack-69.1.2
  configOverrides:
    name: kube-prometheus-stack-overrides

# 如有其他修改，要重新套用
kubectl annotate hr kube-prometheus-stack -n kommander reconcile.fluxcd.io/requestedAt="$(date +%FT%T%Z)" --overwrite

```

patch pvc 擴充至 960Gi

```
kubectl patch pvc db-prometheus-kube-prometheus-stack-prometheus-0 -n kommander -p '{"spec":{"resources":{"requests":{"storage":"960Gi"}}}}'


kubectl describe pvc db-prometheus-kube-prometheus-stack-prometheus-0 -n kommander
```

![image-20250814103450236](https://kenkenny.synology.me:5543/images/2025/08/image-20250814103450236.png)

![image-20250814103516913](https://kenkenny.synology.me:5543/images/2025/08/image-20250814103516913.png)

確認 kube-prometheus-stack 有設定保留時間 90 天

```
kubectl describe appdeployment kube-prometheus-stack -n kommander

Spec:
  App Ref:
    Kind:  ClusterApp
    Name:  kube-prometheus-stack-69.1.2
  Config Overrides:
    Name:  kube-prometheus-stack-overrides


kubectl get hr -n kommander |grep kube-prometheus-stack

kubectl describe hr kube-prometheus-stack -n kommander

kubectl get hr kube-prometheus-stack -n kommander -o yaml |grep -A10 valuesFrom

kubectl get hr kube-prometheus-stack -n kommander -o yaml | grep kube-prometheus-stack-overrides

kubectl get pods -n kommander |grep kube-prometheus-stack

kubectl top pod prometheus-kube-prometheus-stack-prometheus-0  -n kommander

kubectl get pod prometheus-kube-prometheus-stack-prometheus-0 -n kommander -o jsonpath='{.spec.containers[0].args}' | tr ' ' '\n' | grep retention

kubectl describe pod prometheus-kube-prometheus-stack-prometheus-0  -n kommander

# 如果是修改 managed cluster 設定， helm release 要手動新增 valueFrom

  valuesFrom:
  - kind: ConfigMap
    name: kube-prometheus-stack-69.1.2-d2iq-defaults
  - kind: ConfigMap
    name: kube-prometheus-stack-mgmt-overrides
    optional: true
  - kind: ConfigMap
    name: kube-prometheus-stack-overrides
```

![image-20250814103845372](https://kenkenny.synology.me:5543/images/2025/08/image-20250814103845372.png)

managed cluster 修改 helm release
kubectl --kubeconfig=dev-cluster.conf edit hr kube-prometheus-stack -n dev

![image-20250814144055684](https://kenkenny.synology.me:5543/images/2025/08/image-20250814144055684.png)

![image-20250814144033583](https://kenkenny.synology.me:5543/images/2025/08/image-20250814144033583.png)

![image-20250814120601721](https://kenkenny.synology.me:5543/images/2025/08/image-20250814120601721.png)

在 Grafana 如果要拉7天的以上資料，thanos-query 這個 pod 會頻繁的 OOMKilled ，造成畫面出不來，所以要調整記憶體跟CPU大小

```
預設值
    Limits:
      cpu:                150m
      ephemeral-storage:  2Gi
      memory:             192Mi
    Requests:
      cpu:                100m
      ephemeral-storage:  50Mi
      memory:             128Mi
      
kubectl get pods -n kommander | grep thanos-query
kubectl top pod thanos-query-7cfdcf5649-g6phw -n kommander

# 調整成 1Gi ， limit 到 4Gi
kubectl -n kommander patch deployment thanos-query \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "4Gi"}, {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "1Gi"}, {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/cpu", "value": "2000m"}, {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value": "500m"}]'

```

拉30天時，峰值 cpu 大概有到 1200m ， memory 1200m  

![image-20250814105446987](https://kenkenny.synology.me:5543/images/2025/08/image-20250814105446987.png)

![Screenshot 2025-08-14 14.48.37](https://kenkenny.synology.me:5543/images/2025/08/Screenshot 2025-08-14 14.48.37.png)

![Screenshot 2025-08-14 14.48.23](https://kenkenny.synology.me:5543/images/2025/08/Screenshot 2025-08-14 14.48.23.png)



### NTP

NKP 預設 NTP Server 是對外的，如果要修改成內部 NTP 有兩種方式 Daemonset 跟 KubeadmConfigTemplate

原廠比較推薦用 DaemonSet 方式調整（會修改節點的chrony.conf），因為 KubeadmConfigTemplate 有可能因為升級而被覆蓋掉

1. Create ServiceAccount

   vi ntp-sa.yaml

   kubectl -f ntp-sa.yaml

   ```
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: ntp-updater-sa
     namespace: kube-system
   ```

2. Create NTP configmaps

   vi ntp-configmaps.yaml

   kubectl -f ntp-configmaps.yaml

   ```
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: ntp-servers-config
     namespace: kube-system
   data:
     ntp-servers: |
       time.stdtime.gov.tw #請刪除此行備註，修改成現有環境的ntp server 
       tock.stdtime.gov.tw #請刪除此行備註，修改成現有環境的ntp server 
       watch.stdtime.gov.tw #請刪除此行備註，修改成現有環境的ntp server 
   ```

3. Create Daemonsets 

   vi ntp-daemonset.yaml

   kubectl -f ntp-daemonset.yaml

   ```
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: ntp-updater
     namespace: kube-system
     annotations:
       configmap.reloader.stakater.com/reload: "ntp-servers-config"
   spec:
     selector:
       matchLabels:
         app: ntp-updater
     template:
       metadata:
         labels:
           app: ntp-updater
       spec:
         serviceAccountName: ntp-updater-sa
         hostPID: true
         tolerations:
         - key: "node-role.kubernetes.io/master"
           operator: "Exists"
           effect: "NoSchedule"
         - key: "node-role.kubernetes.io/control-plane"
           operator: "Exists"
           effect: "NoSchedule"
         containers:
         - name: ntp-updater
           image: docker.io/library/busybox:1
           securityContext:
             privileged: true
           volumeMounts:
           - name: host-etc
             mountPath: /host/etc
           - name: ntp-config
             mountPath: /config
           command: ["/bin/sh"]
           args:
             - "-c"
             - |
               #!/bin/sh
               set -e
   
               NTP_CONFIG="/config/ntp-servers"
               HOST_CHRONY_CONF="/host/etc/chrony.conf"
               # For Rocky linux , Ubuntu path is /host/etch/chrony/chrony.conf
   
               if [ ! -f "$HOST_CHRONY_CONF" ]; then
                 echo "Chrony configuration not found on host."
                 exit 1
               fi
   
               # Read existing server entries from chrony.conf
               EXISTING_SERVERS=$(grep "^server " "$HOST_CHRONY_CONF" | awk '{print $2}')
   
               # Process each server from the ConfigMap
               while IFS= read -r server_name; do
                 if [ -n "$server_name" ]; then
                   # Construct the server directive
                   SERVER_DIRECTIVE="server $server_name iburst"
   
                   # Check if the server is already configured
                   if echo "$EXISTING_SERVERS" | grep -Fxq "$server_name"; then
                     echo "NTP server already exists: $server_name"
                   else
                     echo "$SERVER_DIRECTIVE" >> "$HOST_CHRONY_CONF"
                     echo "Added NTP server: $server_name"
                     NEED_RESTART=true
                   fi
                 fi
               done < "$NTP_CONFIG"
   
               if [ "$NEED_RESTART" = true ]; then
                 nsenter --target 1 --mount --pid --ipc -- systemctl restart chronyd || echo "Failed to restart chronyd"
               fi
   
               # Keep the container running
               tail -f /dev/null
         volumes:
         - name: host-etc
           hostPath:
             path: /etc
         - name: ntp-config
           configMap:
             name: ntp-servers-config
   ```

4. 驗證 daemonset 與 node 的 /etc/chrony.conf

   ![image-20250718142517481](https://kenkenny.synology.me:5543/images/2025/07/image-20250718142517481.png)

   進入節點

   ![image-20250718143955061](https://kenkenny.synology.me:5543/images/2025/07/image-20250718143955061.png)

   ![image-20250718143204441](https://kenkenny.synology.me:5543/images/2025/07/image-20250718143204441.png)

   ```
   如果有寫錯，要進入節點修改 chrony.conf
   $ sudo vi /etc/chrony/chrony.conf
   $ sudo systemctl restart chronyd
   $ chronyc sources
   $ chronyc tracking
   ```




### Harbor

從 NKP UI 建立好 Harbor 後建立連線方式

#### URL&Password&CA

取得 url 及密碼後在 UI 建立一個 user or robot 帳號讓其他使用者或叢集可以連線

```
# password,預設帳號admin
kubectl get secrets -n ncr-system harbor-admin-password -o jsonpath='{.data.HARBOR_ADMIN_PASSWORD}' | base64 -d

# url
echo "https://$(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.address}'):5000"

# CA
kubectl get kommandercluster -n kommander host-cluster -o jsonpath='{.status.ingress.caBundle}' | base64 -d > ca.crt
```

#### For management cluster

```
# secret harbor-registry-credentials

export REGISTRY_USERNAME="username"
export REGISTRY_PASSWORD="password"

kubectl create secret generic harbor-registry-credentials -n kommander \
--from-literal username=$REGISTRY_USERNAME \
--from-literal password=$REGISTRY_PASSWORD \
--from-file=ca.crt=<(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.caBundle}' | base64 -d)

kubectl create secret generic harbor-registry-credentials \
--from-literal username=$REGISTRY_USERNAME \
--from-literal password=$REGISTRY_PASSWORD \
--from-file=ca.crt=<(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.caBundle}' | base64 -d)
```

```
# 修改 default registry，改完後會做 rolling update 加退節點

kubectl edit cluster <CLUSTER_NAME>

# 編輯改為 NKP 建立的 Harbor URL
imageRegistries:
- url: https://<HARBOR_ADDRESS>:5000
  credentials:
    secretRef:
      name: harbor-registry-credentials
```

![image-20250721131009179](https://kenkenny.synology.me:5543/images/2025/07/image-20250721131009179.png)

![image-20250721133439115](https://kenkenny.synology.me:5543/images/2025/07/image-20250721133439115.png)

#### For Managed Cluster

建立叢集時要輸入 Image Registry URL 及帳號密碼、憑證等資訊

要注意如果後續要修改 image registry 就不能直接用 edit cluster 方式，會一直卡在 Provisioning

要用建立 secret 加上 daemonset 來信任憑證的方式，但如果直接 trust node 裡面的 ca，後續 node 新增或刪除又要再做一次

```
nkp get clusters -A
nkp edit cluster  dev-cluster -n dev

# 會一直卡在 provisioning 要再把 Image Registries 的地方再改回來才會正常
```

![image-20250721140004226](https://kenkenny.synology.me:5543/images/2025/07/image-20250721140004226.png)

![image-20250721141803927](https://kenkenny.synology.me:5543/images/2025/07/image-20250721141803927.png)

##### When Create Cluster

輸入 URL、帳號密碼、CA 憑證 (必要)

![image-20250721164116207](https://kenkenny.synology.me:5543/images/2025/07/image-20250721164116207.png)

![image-20250721164134149](https://kenkenny.synology.me:5543/images/2025/07/image-20250721164134149.png)

##### Secret + Daemonsets

缺點就是每個 namespace 都要產生這組 secret ，
然後每個 deployment 都要加上 imagePullSecrets
或是每個 namespace 的 default ServiceAccount 加上 imagePullSecrets

```
kubectl patch serviceaccount default -n default \
  -p '{"imagePullSecrets": [{"name": "registry-secret"}]}'
```

1. registry-secret.yaml

   ```
   username = nkpuser
   password = Nutanix/4u
   echo -n "nkpuser:Nutanix/4u" | base64
   bmtwdXNlcjpOdXRhbml4LzR1
   
   $ kubectl create secret docker-registry registry-secret \
     --docker-server=192.168.102.92:5000 \
     --docker-username=nkpuser \
     --docker-password='Nutanix/4u' \
     --docker-email=nkpuser@nutanixlab.local \
     --kubeconfig=dev-cluster.conf
     
   ```

2. registry-ca-cert.yaml

   ```
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: registry-ca-cert
     namespace: kube-system
   data:
     ca.crt: |
       -----BEGIN CERTIFICATE-----
       MIIBcDCCARWgAwIBAgIRAIm422gclLcS7Ao1LxMTZQUwCgYIKoZIzj0EAwIwFzEV
       MBMGA1UEAxMMa29tbWFuZGVyLWNhMB4XDTI1MDQxMDA4NTUxOFoXDTM1MDQwODA4
       NTUxOFowFzEVMBMGA1UEAxMMa29tbWFuZGVyLWNhMFkwEwYHKoZIzj0CAQYIKoZI
       zj0DAQcDQgAE7uxYX5Kcy0WRuqaotNJBOrlO3RgARpm+cnohsXSHWiBiFYBgnTo7
       mOyQG16BtgYVVu2x/A1hHdCazwIR6xMfS6NCMEAwDgYDVR0PAQH/BAQDAgKkMA8G
       A1UdEwEB/wQFMAMBAf8wHQYDVR0OBBYEFKRY7oiU5IIkiDEUkWKSbOpuIsUfMAoG
       CCqGSM49BAMCA0kAMEYCIQDvf3ewJjwN4P1Iz9Wo04rPxyddDkm2QC8RXn7wNfwS
       LQIhAOzjrYuRt3Gek5t1qzveRjoNpKmzluCsBqWQLWLQJBvj
       -----END CERTIFICATE-----
   ```

3. registry-ca-installer-daemonset.yaml

   ```
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: registry-ca-installer
     namespace: kube-system
   spec:
     selector:
       matchLabels:
         name: registry-ca-installer
     template:
       metadata:
         labels:
           name: registry-ca-installer
       spec:
         hostNetwork: true
         hostPID: true
         tolerations:
           - operator: "Exists"
         containers:
           - name: ca-installer
             image: busybox
             securityContext:
               privileged: true
             command: ["/bin/sh", "-c"]
             args:
               - |
                 mkdir -p /host/etc/containerd/certs.d/192.168.102.92:5000 && \
                 cp /certs/ca.crt /host/etc/containerd/certs.d/192.168.102.92:5000/ca.crt && \
                 echo "[+] CA Installed. Sleeping..." && sleep 3600
             volumeMounts:
               - name: certs
                 mountPath: /certs
               - name: containerd-certs
                 mountPath: /host/etc/containerd/certs.d
                 mountPropagation: HostToContainer
         volumes:
           - name: certs
             configMap:
               name: registry-ca-cert
           - name: containerd-certs
             hostPath:
               path: /etc/containerd/certs.d
               type: DirectoryOrCreate
   
   ```

   

4. kubectl apply -f 上述2個檔案

   ```
   kubectl apply -f registry-ca.yaml --kubeconfig=dev-cluster.conf
   kubectl apply -f registry-daemonset.yaml --kubeconfig=dev-cluster.conf
   
   kubectl get pods -n kube-system --kubeconfig=dev-cluster.conf |grep registry
   registry-ca-installer-4hft5                         1/1     Running   0               39s
   registry-ca-installer-94nxg                         1/1     Running   0               39s
   registry-ca-installer-9k56f                         1/1     Running   0               39s
   registry-ca-installer-kxcz2                         1/1     Running   0               39s
   registry-ca-installer-xqjq2                         1/1     Running   0               39s
   ```

   

5. deployment 要加上 imagePullsecret

   ```
   vi test-registry.yaml 
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: test-registry-nginx
     namespace: default
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: test-registry-nginx
           image: 192.168.102.92:5000/app/nginx:alpine
         imagePullSecrets:
         - name: registry-secret
   ---      
   kubectl apply -f test-registry.yaml --kubeconfig=dev-cluster.conf
   
   kubectl get pods --kubeconfig=dev-cluster.conf
   NAME                                   READY   STATUS    RESTARTS   AGE
   test-registry-nginx-54f5f6478d-txvx2   1/1     Running   0          9s
   ```

   確認有從 registry url 抓取 image

   ![image-20250721161722831](https://kenkenny.synology.me:5543/images/2025/07/image-20250721161722831.png)



#### docker&podman login

確認 Harbor URL

```
echo "https://$(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.address}'):5000"

https://192.168.102.92:5000
```

信任憑證

```
echo "$(kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.address}')"

export HARBOR_ADDRESS="192.168.102.92"

sudo mkdir -p /etc/docker/certs.d/$HARBOR_ADDRESS:5000/

kubectl -n kommander get kommandercluster host-cluster -o jsonpath='{.status.ingress.caBundle}' | base64 -d | sudo tee /etc/docker/certs.d/$HARBOR_ADDRESS:5000/ca.crt > /dev/null
```

Docker login

```
docker login https://192.168.102.92:5000 -unkpuser

i Info → A Personal Access Token (PAT) can be used instead.
         To create a PAT, visit https://app.docker.com/settings
         
         
Password: 
Login Succeeded
```

Tag image 並上傳測試

推送 Image 命令範例

![image-20250721170147149](https://kenkenny.synology.me:5543/images/2025/07/image-20250721170147149.png)

```
# 範例
docker tag SOURCE_IMAGE[:TAG] 192.168.102.92:5000/nkp/REPOSITORY[:TAG]
docker push 192.168.102.92:5000/nkp/REPOSITORY[:TAG]


docker image load -i k8s-agent-1.1.688.tar 
docker image tag k8s-agent:1.1.688 192.168.102.92:5000/nkp/k8s-agent:688
docker push 192.168.102.92:5000/nkp/k8s-agent:688
```

![image-20250721170343903](https://kenkenny.synology.me:5543/images/2025/07/image-20250721170343903.png)

![image-20250721170414606](https://kenkenny.synology.me:5543/images/2025/07/image-20250721170414606.png)

![image-20250721170425943](https://kenkenny.synology.me:5543/images/2025/07/image-20250721170425943.png)



### MetalLB

```
metallb-conf.yaml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb
  namespace: metallb-system
spec:
  addresses:
  - 172.16.90.209-172.16.90.211
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: metallb
  namespace: metallb-system
spec:
  ipAddressPools:
  - metallb
```



### NDK

Nutanix Data Service for Kubernetes，NKP 叢集安裝好後安裝 NDK 來實現開發人員自助還原應用程式的功能

對應 Velero 是做備份到外部 Storage，NDK 是將應用程式快照在 Nutanix Cluster 當中 

[NDK Documents](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Data-Services-for-Kubernetes-v1_3:Nutanix-Data-Services-for-Kubernetes-v1_3)

#### Limitations

NDK 1.3.0 目前不支援任何 NKP 的版本，所以 NKP 目前只能安裝 NDK 1.2.0

1. 單座 Prism Element 支援 Kubernetes one-to-one replication

   Nutanix cluster disaster recovery constructs 不支援在同一座 Prism Element cluster

2. 兩座以上 Prism Element 支援 Kubernetes one-to-many replications

   但是AOS 版本必須相同

3. 支援同一或多個 Prism Central 管理兩座 Prism Element 架構，必需定義好 Availability Zones paired 

   並且顯示 **Reachable**

4. NDK supports only Nutanix Volumes.

5. 主要 k8s 叢集跟 DR 叢集 Production 環境下不應該在同一個 Prism Element

6. ApplicationSnapshot 必需小於或等於 8 MB

```
Snapshot space consists of two parts:
1. Configuration data space or metadata space that contains all the specifications that define the application (YAML file) that are stored in etcd as Secret.
2. Volume snapshot space that contains the volume data snapshots (persistent volumes).
```

#### Requirements

1. DR及主要的叢集中 Prism Central 與 Prism Element 的 UUID 以及 Data Service IP (StorageCluster需要)

2. Prism Central 啟用 Kubernetes Management 並更新到最新，兩地的 Prism Central 要做Availability Zones串連

3. 兩座 Kubernetes 必須同一個供應商 ex. NKP 對 NKP or OCP 對 OCP

4. Kubenetes Cluster  安裝 Nutanix CSI 3.0 以上、 Cert-manager、Helm ( NKP預設皆有安裝在 ntnx-system namespace ) 

5. Onboard Kubenetes Cluster 到 Prism Central

6. 保留一個 Loadbalancer IP 給 NDK 使用

7. 需要有 Image Registry 上傳 k8s-agent 以及 ndk 相關的 image
   ( 此測試用 NKP 建立出來的 Harbor ( 192.168.102.92:5000/nkp ) )

8. 開發者權限補充：各 crd 列表在 ndk-1.2.0/chart/crds

   ![image-20250722180942990](https://kenkenny.synology.me:5543/images/2025/07/image-20250722180942990.png)

#### Installation Step

1. 下載 k8s agent 、 ndk 等 Image
2. 推送到 Image Registry
3. Helm 安裝 k8s agent ( Onboard K8s cluster 到 PC )
4. Helm 安裝 NDK
5. NDK 設定 Storage Cluster 以及 Replication Target (單座不用)
6. 建立應用程式 CR 測試本地快照還原
7. 設定排程快照測試



##### 下載 Image

Support Portal 下載 NDK 以及 k8s-agent

![image-20250721174232894](https://kenkenny.synology.me:5543/images/2025/07/image-20250721174232894.png)

![image-20250721174259984](https://kenkenny.synology.me:5543/images/2025/07/image-20250721174259984.png)

```
curl -o ndk-1.3.0.tar "https://download.nutanix.com/downloads/ndk/1.2.0/ndk-1.2.0.tar?Expires=1745237015&Key-Pair-Id=APKAJTTNCWPEI42q5efZ3aJGjekGgoW5w__"

curl -o k8s-agent-1.1.688.tar "https://download.nutanix.com/downloads/ndk/1.2.0/k8s-agent-1.1.688.tar?Expires=1745237080&Key-Pair-Id=APKAJTTNCWPEI42QKMSA&Signature=PB74DhIdBozRvpm-wEgkzVZQU5N2YksxO3T6ew1WAwt7itJwx-swYqrDuw__"
```

##### Push Image

k8s agent

```
docker image load -i k8s-agent-1.1.688.tar 
docker image tag k8s-agent:1.1.688 192.168.102.92:5000/nkp/k8s-agent:688
docker push 192.168.102.92:5000/nkp/k8s-agent:688
```

![image-20250721170414606](https://kenkenny.synology.me:5543/images/2025/07/image-20250721170414606.png)

ndk images

這裡我 tag 出來的 image 名稱不一樣，因為 Harbor push 上去的前面是帶專案名稱，而且會有不合規的名稱情形，所以我改加上 ndk- 前綴

1.3.0

```
docker image load -i ndk-1.3.0/ndk-1.3.0.tar

ndk/manager                         1.3.0     5e39e767485d   5 weeks ago     74.3MB
ndk/infra-manager                   1.3.0     a5fcd3e7279b   5 weeks ago     31.9MB
ndk/job-scheduler                   1.3.0     b0e766afd25e   3 months ago    58.8MB
ndk/kubectl                         1.32.3    f3079d8d0555   3 months ago    299MB
ndk/kube-rbac-proxy                 v0.19.0   da047c323be4   5 months ago    73.9MB

docker image tag ndk/manager:1.3.0 192.168.102.92:5000/nkp/ndk-manager:1.3.0
docker image tag ndk/infra-manager:1.3.0 192.168.102.92:5000/nkp/ndk-infra-manager:1.3.0
docker image tag ndk/job-scheduler:1.3.0 192.168.102.92:5000/nkp/ndk-job-scheduler:1.3.0
docker image tag ndk/kubectl:1.32.3 192.168.102.92:5000/nkp/ndk-kubectl:1.32.3
docker image tag ndk/kube-rbac-proxy:v0.19.0 192.168.102.92:5000/nkp/ndk-kube-rbac-proxy:v0.19.0

docker push 192.168.102.92:5000/nkp/ndk-manager:1.3.0
docker push 192.168.102.92:5000/nkp/ndk-infra-manager:1.3.0
docker push 192.168.102.92:5000/nkp/ndk-job-scheduler:1.3.0
docker push 192.168.102.92:5000/nkp/ndk-kubectl:1.32.3
docker push 192.168.102.92:5000/nkp/ndk-kube-rbac-proxy:v0.19.0
```

1.2.0

```
docker image tag ndk/manager:1.2.0 192.168.102.92:5000/nkp/ndk-manager:1.2.0
docker image tag ndk/infra-manager:1.2.0 192.168.102.92:5000/nkp/ndk-infra-manager:1.2.0
docker image tag ndk/job-scheduler:1.2.0 192.168.102.92:5000/nkp/ndk-job-scheduler:1.2.0
docker image tag ndk/bitnami-kubectl:1.30.3 192.168.102.92:5000/nkp/ndk-bitnami-kubectl:1.30.3
docker image tag ndk/kube-rbac-proxy:v0.17.0 192.168.102.92:5000/nkp/ndk-kube-rbac-proxy:v0.17.0

docker push 192.168.102.92:5000/nkp/ndk-manager:1.2.0
docker push 192.168.102.92:5000/nkp/ndk-infra-manager:1.2.0
docker push 192.168.102.92:5000/nkp/ndk-job-scheduler:1.2.0
docker push 192.168.102.92:5000/nkp/ndk-bitnami-kubectl:1.30.3
docker push 192.168.102.92:5000/nkp/ndk-kube-rbac-proxy:v0.17.0
```

![image-20250721180445811](https://kenkenny.synology.me:5543/images/2025/07/image-20250721180445811.png)

![image-20250721180623296](https://kenkenny.synology.me:5543/images/2025/07/image-20250721180623296.png)

![image-20250722125736075](https://kenkenny.synology.me:5543/images/2025/07/image-20250722125736075.png)

##### install k8s-agent

1. 取得並確認 Image Registry 的帳號密碼

   ```
   $ echo -n "nkpuser:Nutanix/4u" | base64
   bmtwdXNlcjpOdXRhbml4LzR1
   
   
   $ vi dockerconfig.json
   
   #---分隔線---
   {
     "auths": {
       "https://192.168.102.92:5000": {
         "username": "nkpuser",
         "password": "Nutanix/4u",
         "auth": "bmtwdXNlcjpOdXRhbml4LzR1"
       }
     }
   }
   #---分隔線---
   
   $ cat dockerconfig.json | base64 -w 0
   ewogICJhdXRocyI6IHsKICAgICJodHRwczovLzE5Mi4xNjguMTAyLjkyOjUwMDAiOiB7CiAgICAgICJ1c2VybmFtZSI6ICJua3B1c2VyIiwKICAgICAgInBhc3N3b3JkIjogIk51dGFuaXgvNHUiLAogICAgICAiYXV0aCI6ICJibXR3ZFhObGNqcE9kWFJoYm1sNEx6UjEiCiAgICB9CiAgfQp9Cg==
   ```

   

2. 確認 cluster uid

   ```
   $ kubectl --kubeconfig <kubeconfig> get ns kube-system -o json | grep uid
    
   $ kubectl --kubeconfig=dev-cluster.conf get ns kube-system -o json | grep uid
           "uid": "3e0ce419-72ff-4f9e-819d-b2e516f0fb8a"
   ```

3. 設定環境變數 cluster name 、 uid (Optional)

   ```
   $ export CLUSTER_NAME=<cluster-name>
   $ export CLUSTER_UUID=<uid-of-kube-system-namespace>
   
   $ kubectl  get cluster -A
   NAMESPACE   NAME          CLUSTERCLASS          PHASE         AGE     VERSION
   default     netfos-nkp    nkp-nutanix-v2.14.0   Provisioned   102d    v1.31.4
   dev         dev-cluster   nkp-nutanix-v2.14.0   Provisioned   7d22h   v1.31.4
   
   $ export CLUSTER_NAME=dev-cluster
   $ export CLUSTER_UUID=3e0ce419-72ff-4f9e-819d-b2e516f0fb8a
   ```

4. 修改 values.yaml，其中 categoryMappings 部分 KubernetesClusterUUID=  前面要加 **k8s-**

   ```
   $ cd /root/ndk/k8sagent/k8s-agent-1-1-688/chart
   $ ls -l 
   total 448
   -rw-r--r-- 1 502 games    212 Oct 16  2024  Chart.yaml
   -rw-r--r-- 1 502 games 433682 Oct 16  2024 'Nutanix-core_k8s-agent Notice.txt'
   -rw-r--r-- 1 502 games    228 Oct 16  2024  README.md
   drwxr-xr-x 2 502 games    184 Jul 21 16:58  templates
   -rw-r--r-- 1 502 games   8346 Oct 16  2024  values.schema.json
   -rw-r--r-- 1 502 games    770 Oct 16  2024  values.yaml
   
   $ vi values.yaml
   
   
   agent:
     namespaceOverride: ntnx-system
     name: nutanix-agent
     port: 8080
     image:
       repository: 192.168.102.92:5000/nkp
       name: k8s-agent
       pullPolicy: IfNotPresent
       tag: 688
       privateRegistry: true
       imageCredentials:
         dockerconfig: "ewogICJhdXRocyI6IHsKICAgICJodHRwczovLzE5Mi4xNjguMTAyLjkyOjUwMDAiOiB7CiAgICAgICJ1c2VybmFtZSI6ICJua3B1c2VyIiwKICAgICAgInBhc3N3b3JkIjogIk51dGFuaXgvNHUiLAogICAgICAiYXV0aCI6ICJibXR3ZFhObGNqcE9kWFJoYm1sNEx6UjEiCiAgICB9CiAgfQp9Cg=="
     updateConfigInMin: 10
     updateMetricsInMin: 360
   
   pc:
     port: 9440
     insecure: true #set this to true if PC does not have https enabled
     endpoint: "172.16.90.75" # eg: ip or fqdn
     username: "admin" # eg: admin or any other user with Kubernetes Infrastructure provision role
     password: "********"
   k8sClusterName: "dev-cluster"
   k8sClusterUUID: "3e0ce419-72ff-4f9e-819d-b2e516f0fb8a"
   k8sDistribution: "NKP" # eg: CAPX or NKE or OCP or EKSA or NKP
   categoryMappings: "KubernetesClusterName=dev-cluster,KubernetesClusterUUID=k8s-3e0ce419-72ff-4f9e-819d-b2e516f0fb8a" # "one or more comma separated key=value" eg: "key1=value1" or "key1=value1\,key2=value2"
   ```

   secret 部分理論上應該可以改成用現有的 secret，其他部分留白，private registry 記得要先處理憑證

   ```
   imageCredentials:
     imagePullSecretName: <secret name>
     credentials:
       username: ""
       password: ""
       email: ""
   ```

   

5. 安裝 k8s-agent

   ```
   $ pwd
   /root/ndk/k8sagent
   $ ls -l
   total 20928
   drwxr-xr-x 3  502 games       48 Oct 16  2024 k8s-agent-1-1-688
   -rw-r--r-- 1 root root  21428736 Jul 21 16:58 k8s-agent-1.1.688.tar
   
   $ helm install nutanix-k8s-agent-1.1.688 k8s-agent-1-1-688/chart  -n ntnx-system --kubeconfig=/root/dev-cluster/dev-cluster.conf 
   ```

   ![image-20250722103616628](https://kenkenny.synology.me:5543/images/2025/07/image-20250722103616628.png)

   ![image-20250722103754935](https://kenkenny.synology.me:5543/images/2025/07/image-20250722103754935.png)

6. Prism Central 確認已經 Onbaord

   ![image-20250722103558377](https://kenkenny.synology.me:5543/images/2025/07/image-20250722103558377.png)

   ![image-20250722103832394](https://kenkenny.synology.me:5543/images/2025/07/image-20250722103832394.png)

   ![image-20250722103854923](https://kenkenny.synology.me:5543/images/2025/07/image-20250722103854923.png)

   ![image-20250722103910875](https://kenkenny.synology.me:5543/images/2025/07/image-20250722103910875.png)

7. 移除方式

   ```
   helm uninstall nutanix-k8s-agent-1.1.688 -n ntnx-system
   helm list -n ntnx-system
   kubectl get secret -n ntnx-system | grep nutanix-k8s-agent-1.1.688
   kubectl get configmap -n ntnx-system | grep nutanix-k8s-agent-1.1.688
   kubectl get deployment -n ntnx-system | grep nutanix-k8s-agent-1.1.688
   kubectl delete secret sh.helm.release.v1.nutanix-k8s-agent-1.1.688.v1 -n ntnx-system
   ```
   
   ![image-20250904105720496](https://kenkenny.synology.me:5543/images/2025/09/image-20250904105720496.png)

##### install ndk

官方建議使用 Loadbalancer 的 Service Type ，避免其他錯誤

1. 建立 Prism Central secret 與確認 Registry 的 ImagePullSecret

   ```
   $ vi ntnx-pc-secret.yaml
   
   apiVersion: v1
   kind: Secret
   metadata:
     name: ntnx-pc-secret
     namespace: ntnx-system
   stringData:
     # prism-pc-ip:prism-port:admin:password
     key: 172.16.90.75:9440:admin:password
     
   $ kubectl --kubeconfig=dev-cluster.conf apply -f ntnx-pc-secret.yaml 
   $ kubectl --kubeconfig=dev-cluster.conf get secret -n ntnx-system
   
   NAME                                              TYPE                             DATA   AGE
   ntnx-pc-secret                                    Opaque                           1      75m
   nutanix-agent                                     kubernetes.io/basic-auth         2      91m
   nutanix-csi-credentials                           Opaque                           1      8d
   nutanix-k8s-agent-pull-secret                     kubernetes.io/dockerconfigjson   1      92m
   registry-secret                                   kubernetes.io/dockerconfigjson   1      156m
   sh.helm.release.v1.nutanix-csi.v1                 helm.sh/release.v1               1      8d
   sh.helm.release.v1.nutanix-k8s-agent-1.1.688.v1   helm.sh/release.v1               1      92m
   
   
   $ kubectl patch serviceaccount ndk-controller-manager -n ntnx-system \
     -p '{"imagePullSecrets":[{"name":"nutanix-k8s-agent-pull-secret"}]}' --kubeconfig=dev-cluster.conf
   serviceaccount/ndk-controller-manager patched
   
   ```

   

2. 使用 helm 安裝 原廠範例設定 image 跟 tag

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

   

3. 或是選擇修改 value.yaml 在 Chart 目錄底下，如果要改連線透過 Node port 的話要把 Loadbalancer 換掉
   這裡是把 LoadBalancer 換掉，如果沿用就不用修改，另外修改 ImagePullSecret 

   ```
   $ cat values.yaml
   # Default values for nutanix-dataservices-operator
   # This is a YAML-formatted file.
   # Declare variables to be passed into your templates.
   
   manager:
     # -- Image Repository
     repository: 192.168.102.92:5000/nkp/ndk-manager
     # -- Image tag
     # @default -- .Chart.AppVersion
     tag: "1.2.0"
     # -- Image digest
     digest: 
     # -- Image pull policy
     pullPolicy: Always
   
   infraManager:
     # -- Image Repository
     repository: 192.168.102.92:5000/nkp/ndk-infra-manager 
     # -- Image tag
     # @default -- .Chart.AppVersion
     tag: "1.2.0"
     # -- Image digest
     digest: 
     # -- Image pull policy
     pullPolicy: Always
   
   kubeRbacProxy:
     # -- Image Repository
     repository: 192.168.102.92:5000/nkp/ndk-kube-rbac-proxy 
     # -- Image tag
     tag: "v0.17.0"
     # -- Image digest
     digest: 
   
   kubectl:
     # -- Image Repository
     repository: 192.168.102.92:5000/nkp/ndk-bitnami-kubectl
     # -- Image tag
     tag: "1.30.3"
     # -- Image digest
     digest: 
     # -- Image pull policy
   
   # Set Values for NDK Metrics Monitoring using Prometheus
   prometheus:
     # Requires Prometheus Operator CRDs
     enable: false
     # Set values for Prometheus serviceMonitor
     serviceMonitor:
       # Add Custom labels for serviceMonitor here
       customLabels: {}
   
   # Set Values for Job Scheduler container
   jobScheduler:
     # -- Image Repository
     repository: 192.168.102.92:5000/nkp/ndk-job-scheduler
     # -- Image tag
     # @default -- .Chart.AppVersion
     tag: "1.2.0"
     # -- Image digest
     digest: 
     # -- Image pull policy
     pullPolicy: Always
   
   # Credentials to pull image from container registry.
   imageCredentials:
     # Name of the secret containing the credentials to pull image from the container registry.
     imagePullSecretName: nutanix-k8s-agent-pull-secret
     # Image registry credentials
     # The credentials are used to create the image pull secret during helm install.
     # If the username is specified, the password and email must also be specified.
     # If the username is not specified, it is assumed the image pull secret already
     # exists in the cluster.
     credentials:
       registry: ""
       username: ""
       password: ""
       email: ""
   # Configuration to expose controller metrics service
   metricsService:
     ports:
     - name: https
       port: 8443
       protocol: TCP
       targetPort: https
     type: ClusterIP
   # Configuration to expose NDK's webhook service
   webhookService:
     ports:
     - port: 443
       protocol: TCP
       targetPort: 9443
     type: ClusterIP
   # Configuration to expose the grpc service.
   intercomService:
     ports:
     - port: 2021
       protocol: TCP
       targetPort: 2021
       nodePort: 32021
     type: NodePort
     # kube-vip  v0.2.1+ must be installed for this feature to work.
     # When set to true, kube-vip uses the local network DHCP to assign
     # LoadBalancer IP address to the intercom service. For dev/qa purposes only.
     useKubevipDhcp: false
     loadBalancerClass:
     loadBalancerIP:
   # Configuration to expose job scheduler's webhook service
   schedulerWebhookService:
     port: 9444
     protocol: TCP
     targetPort: 9444
     type: ClusterIP
     
   # Configurable parameters for dataservices controllers 
   config:
     # To provide secret to be used by controllers for storage backend authentication
     # @Required-- secret should be in the helm release namespace
     secret:
       # pointer to the secret to be consumed by controllers
       name: ntnx-pc-secret
       # path to mount the secret as a volume
       dir:  "/var/run/ntnx-secret-dir"
   
   ```

   

4. helm 安裝 (無 tls 安裝)
   用 private registry 有發生 apply-crd 這個 job 沒有使用 nutanix-k8s-agent-pull-secret 這個 secret 去拉取 Image
   最後是 pathch ServiceAccount ndk-controller-manager 讓他預設使用此 Secret ，然後刪除 apply-crd 的 pod 後就正常了

   ```
   $ ll ndk-1.2.0/chart/
   total 916
   -rwxr-xr-x 1 3434 3434   1542 Oct 22  2024 Chart.yaml
   drwxr-xr-x 2 3434 3434   4096 Oct 22  2024 crds
   -rw-r--r-- 1 3434 3434 177008 Oct 22  2024 Nutanix-core-k8s-job-scheduler-Disclosure.txt
   -rw-r--r-- 1 3434 3434 223591 Oct 22  2024 Nutanix-core-k8s-juno-aos-pc-client-Disclosure.txt
   -rw-r--r-- 1 3434 3434 223234 Oct 22  2024 Nutanix-core-k8s-juno-Disclosure.txt
   -rw-r--r-- 1 3434 3434 269411 Oct 22  2024 Nutanix-kube-rbac-proxy-Disclosure.txt
   drwxr-xr-x 2 3434 3434   4096 Oct 22  2024 templates
   -rw-r--r-- 1 3434 3434   9066 Jul 22 13:01 values.yaml
   -rw-r--r-- 1 root root   8791 Jul 22 12:57 values.yaml.bk
   
   
   $ helm install ndk -n ntnx-system ndk-1.2.0/chart/ --set tls.server.enable=false --kubeconfig=/root/dev-cluster/dev-cluster.conf
   
   $ kubectl patch serviceaccount ndk-controller-manager -n ntnx-system \
     -p '{"imagePullSecrets":[{"name":"nutanix-k8s-agent-pull-secret"}]}' --kubeconfig=dev-cluster.conf
   serviceaccount/ndk-controller-manager patched
   ```

   

5. 確認安裝完成

   ```
   $ kubectl --kubeconfig=dev-cluster.conf get pods -n ntnx-system
   
   $ kubectl --kubeconfig=dev-cluster.conf get svc -n ntnx-system
   
   NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
   ndk-controller-manager-metrics-service   ClusterIP   10.108.171.243   <none>        8443/TCP                              78s
   ndk-intercom-service                     NodePort    10.103.128.57    <none>        2021:32021/TCP                        78s
   ndk-scheduler-webhook-service            ClusterIP   10.105.65.97     <none>        9444/TCP                              78s
   ndk-webhook-service                      ClusterIP   10.111.167.91    <none>        443/TCP                               78s
   nutanix-csi-metrics                      ClusterIP   10.104.85.189    <none>        9809/TCP,9810/TCP,9811/TCP,9812/TCP   8d
   
   ```

   ![image-20250722132617908](https://kenkenny.synology.me:5543/images/2025/07/image-20250722132617908.png)

   ![image-20250722134057031](https://kenkenny.synology.me:5543/images/2025/07/image-20250722134057031.png)

6. 用 grpcurl 來連 node or Loadbalancer ip 確認服務正常

   ```
   $ kubectl --kubeconfig=dev-cluster.conf get nodes -o wide
   
   NAME                                 STATUS                     ROLES           AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
   dev-cluster-md-0-424sp-bzw7l-db2s5   Ready                      <none>          7d21h   v1.31.4   192.168.102.164   <none>        Ubuntu 22.04.5 LTS   5.15.0-135-generic   containerd://1.7.24-d2iq.1
   dev-cluster-md-0-424sp-bzw7l-z6qsp   Ready                      <none>          7d21h   v1.31.4   192.168.102.120   <none>        Ubuntu 22.04.5 LTS   5.15.0-135-generic   containerd://1.7.24-d2iq.1
   dev-cluster-v9lxs-9c2bk              Ready,SchedulingDisabled   control-plane   23h     v1.31.4   192.168.102.113   <none>        Ubuntu 22.04.5 LTS   5.15.0-135-generic   containerd://1.7.24-d2iq.1
   dev-cluster-v9lxs-thx8s              Ready                      control-plane   23h     v1.31.4   192.168.102.112   <none>        Ubuntu 22.04.5 LTS   5.15.0-135-generic   containerd://1.7.24-d2iq.1
   
   $ docker run fullstorydev/grpcurl -plaintext 192.168.102.164:32021 list
   grpc.reflection.v1.ServerReflection
   grpc.reflection.v1alpha.ServerReflection
   juno_interface.Juno
   
   $ docker run fullstorydev/grpcurl -plaintext -d '{}' 192.168.102.164:32021 juno_interface.Juno.GetVersion
   {
     "name": "\"1.2.0\""
   }
   ```

   ![image-20250722135407582](https://kenkenny.synology.me:5543/images/2025/07/image-20250722135407582.png)

##### NDK Setting

###### Storage Cluster

安裝好後必須設定 Storage Cluster，如果是有DR環境，兩邊也都要設定好 Storage Cluster

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
```

StorageCluster-NX1365G6PE.yaml 

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: StorageCluster
metadata:
 name: nx1365g6pe
spec:
 storageServerUuid: 00061972-aeba-fee2-0000-000000028e95
 managementServerUuid: acb94940-7f03-416e-9dcd-6b442d3fe081
```

apply 並驗證

```
$ kubectl apply -f StorageCluster-NX1365G6PE.yaml --kubeconfig=/root/dev-cluster/dev-cluster.conf
$ kubectl get storagecluster --kubeconfig=/root/dev-cluster/dev-cluster.conf
$ kubectl describe storagecluster nx1365g6pe --kubeconfig=/root/dev-cluster/dev-cluster.conf
```

![image-20250722140208396](https://kenkenny.synology.me:5543/images/2025/07/image-20250722140208396.png)



###### Remote Cluster

先確認 Remote 的 Cluster 有安裝好 NDK

Remote CR without TLS

RemoteCR.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: Remote
metadata:
  name: netfos-nkp
spec:
  clusterName: netfos-nkp
  ndkServiceIp: 192.168.102.93
  ndkServicePort: 2021
  
$ kubectl apply -f RemoteCR.yaml
$ kubectl get remote
$ kubectl describe remote netfos-nkp-dr
```

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

##### Testing NDK

###### 本地手動快照還原

部署測試應用程式，有加上 label 是 nginx-ndk

nginx-demo-ndk.yaml

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-ndk-pvc
  namespace: demo-ndk
  labels:
    app: nginx-ndk
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ndk
  namespace: demo-ndk
  labels:
    app: nginx-ndk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ndk
  template:
    metadata:
      labels:
        app: nginx-ndk
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-storage
        persistentVolumeClaim:
          claimName: nginx-ndk-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: demo-ndk
  labels:
    app: nginx-ndk
spec:
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
  selector:
    app: nginx-ndk

```

```
$ kubectl get all -n demo-ndk
NAME                            READY   STATUS    RESTARTS   AGE
pod/nginx-ndk-575785bbd-rnf5x   1/1     Running   0          85s

NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/nginx-svc   NodePort   10.109.204.85   <none>        80:30080/TCP   85s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ndk   1/1     1            1           85s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ndk-575785bbd   1         1         1       85s

$ kubectl  -n demo-ndk exec nginx-ndk-575785bbd-rnf5x -it -- /bin/bash
root@nginx-ndk-575785bbd-rnf5x:/# echo "Hello Testing NDK" > /usr/share/nginx/html/index.html
root@nginx-ndk-575785bbd-rnf5x:/# exit
```

![image-20250722153548329](https://kenkenny.synology.me:5543/images/2025/07/image-20250722153548329.png)

建立Application CR

此範例是 namespace demo-ndk 內 label 為 nginx-ndk 的應用程式，也可以設定 includeResources: 來指定要快照的項目

demo-ndk-cr.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: Application
metadata:
  name: demo-nginx-cr
  namespace: demo-ndk
spec:
  applicationSelector:
    resourceLabelSelectors:
      - labelSelector:
          matchLabels:
            app: nginx-ndk

$ kubectl apply -f demo-ndk-cr.yaml

$ kubectl get application -n demo-ndk
NAME            AGE   LAST-STATUS-UPDATE
demo-nginx-cr   20s   20s

$ kubectl describe applications -n demo-ndk
kubectl describe applications -n demo-ndk
Name:         demo-nginx-cr
Namespace:    demo-ndk
Labels:       <none>
Annotations:  <none>
API Version:  dataservices.nutanix.com/v1alpha1
Kind:         Application
Metadata:
  Creation Timestamp:  2025-07-22T08:17:54Z
  Finalizers:
    dataservices.nutanix.com/app
  Generation:        1
  Resource Version:  8011050
  UID:               c9e42bfe-be5f-4715-b263-9a587d7211c2
Spec:
  Application Selector:
    Resource Label Selectors:
      Label Selector:
        Match Labels:
          App:  nginx-ndk
Status:
  Last Updated Time:  2025-07-22T08:17:54Z
  Summary:
    Resources:
      apps/v1/Deployment:
        Name:  nginx-ndk
      cilium.io/v2/CiliumEndpoint:
        Name:  nginx-ndk-575785bbd-rnf5x
      v1/PersistentVolumeClaim:
        Name:  nginx-ndk-pvc
      v1/Service:
        Name:  nginx-svc
Events:        <none>

```

建立 ApplicationSnapshot CR

demo-ndk-snap-cr.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ApplicationSnapshot
metadata:
  name: demo-ndk-snap-1
  namespace: demo-ndk
spec:
  source:
    applicationRef:
     name: demo-nginx-cr
  expiresAfter: 7200m
 
$ kubectl apply -f demo-ndk-snap-cr.yaml
$ kubectl get applicationsnapshot -n demo-ndk
kubectl get applicationsnapshot -n demo-ndk
NAME              AGE   READY-TO-USE   BOUND-SNAPSHOTCONTENT                      SNAPSHOT-AGE
demo-ndk-snap-1   39s   true           asc-e665dd62-23f5-42c7-8c1e-5bc128db1cb8   13s

$ kubectl get applicationsnapshot -n demo-ndk demo-ndk-snap-1 -o yaml
```

![image-20250722162030069](https://kenkenny.synology.me:5543/images/2025/07/image-20250722162030069.png)

管理者在 Prism Central 也可以看到有快照產生

![image-20250722162051463](https://kenkenny.synology.me:5543/images/2025/07/image-20250722162051463.png)

![image-20250722162115480](https://kenkenny.synology.me:5543/images/2025/07/image-20250722162115480.png)

![image-20250722162128121](https://kenkenny.synology.me:5543/images/2025/07/image-20250722162128121.png)



刪除 nginx 服務及容器，（Nutanix 建議要復原時要把原來的應用程式刪除）

```
$ kubectl delete -f nginx-demo-ndk.yaml

persistentvolumeclaim "nginx-ndk-pvc" deleted
deployment.apps "nginx-ndk" deleted
service "nginx-svc" deleted

$ kubectl get all -n demo-ndk
No resources found in demo-ndk namespace.
```

![image-20250722162153005](https://kenkenny.synology.me:5543/images/2025/07/image-20250722162153005.png)

建立 ApplicationSnapshotRestore CR

demo-ndk-restore-cr.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ApplicationSnapshotRestore
metadata:
  name: restore-demo-ndk-snap-1
  namespace: demo-ndk
spec:
  applicationSnapshotName: demo-ndk-snap-1
  
kubectl apply -f demo-ndk-restore-cr.yaml
kubectl get ApplicationSnapshotRestore -n demo-ndk
kubectl describe ApplicationSnapshotRestore -n demo-ndk
```

![image-20250722162255912](https://kenkenny.synology.me:5543/images/2025/07/image-20250722162255912.png)

復原完成

```
kubectl get ApplicationSnapshotRestore -n demo-ndk
kubectl get all -n demo-ndk
```

![image-20250722163006189](https://kenkenny.synology.me:5543/images/2025/07/image-20250722163006189.png)

![image-20250722163039205](https://kenkenny.synology.me:5543/images/2025/07/image-20250722163039205.png)

###### 排程快照

**排程快照需要建立** 

JobSchedulerCR.yaml、ProtectionPlanCR.yaml、AppProtectionPlanCR.yaml

demo-ndk-jobschedule.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ProtectionPlan
metadata:
 name: protect-plan-6h
 namespace: demo-ndk
spec:
 scheduleName: interval-6h
 retentionPolicy:
     retentionCount: 5
```

demo-ndk-protectplan.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: ProtectionPlan
metadata:
 name: protect-plan-6h
 namespace: demo-ndk
spec:
 scheduleName: interval-6h
 retentionPolicy:
     retentionCount: 5
```

demo-ndk-appprotecplan.yaml

```
apiVersion: dataservices.nutanix.com/v1alpha1
kind: AppProtectionPlan
metadata:
  name: nginx-ndk-plan
  namespace: demo-ndk
spec:
  applicationName: demo-nginx-cr
  labels:
    appName: demo-nginx-cr
  protectionPlanNames:
  - protect-plan-6h
```

```
kubectl get jobschedulers -n demo-ndk
kubectl get protectionplan -n demo-ndk
kubectl get appprotectionplan  -n demo-ndk
kubectl describe appprotectionplan nginx-ndk-plan -n demo-ndk
kubectl get applicationsnapshots -n demo-ndk
kubectl describe applicationsnapshots  -n demo-ndk
```

![image-20250723132949638](https://kenkenny.synology.me:5543/images/2025/07/image-20250723132949638.png)

![image-20250723132826432](https://kenkenny.synology.me:5543/images/2025/07/image-20250723132826432.png)

![image-20250723133121998](https://kenkenny.synology.me:5543/images/2025/07/image-20250723133121998.png)

![image-20250723133134403](https://kenkenny.synology.me:5543/images/2025/07/image-20250723133134403.png)



## Command



### Node CPU&MEM

```
kubectl describe node | egrep "Name:|cpu" 

kubectl describe node | egrep "Name:|memory"

kubectl top nodes
```

![image-20250715112719773](https://kenkenny.synology.me:5543/images/2025/07/image-20250715112719773.png)

### Pod Usage

```
kubectl top pod mysql-0 -n demo-dev
```

### NS Usage&Quotas

```
kubectl get resourcequota -n namespace
kubectl describe resourcequota -n namespace
```



### Containerd Log

```
sudo journalctl -u containerd -o cat --since "10 minutes ago" | grep -i "PullImage" -n
sudo journalctl -u kubelet -o cat -n 200 | sed -n '1,200p'
sudo journalctl -u kubelet

# 列出 containerd 的 image（k8s namespace）
sudo ctr -n k8s.io images list | grep -i kube-vip -n

# 或使用 crictl（若有安裝）
sudo crictl images | grep -i kube-vip -n


# pull mirror image（用 ctr）
sudo ctr -n k8s.io images pull 192.168.102.92:5000/nkp-2.14/kube-vip/kube-vip@sha256:2046373e4e8856f35dfbea635faedfb1269bb19cd9a1b2d62dcfca6b32e7d170

# 然後 tag 成 ghcr.io 的 reference（使用相同 digest）
sudo ctr -n k8s.io images tag \
  192.168.102.92:5000/nkp-2.14/kube-vip/kube-vip@sha256:2046373e4e8856f35dfbea635faedfb1269bb19cd9a1b2d62dcfca6b32e7d170 \
  ghcr.io/kube-vip/kube-vip@sha256:2046373e4e8856f35dfbea635faedfb1269bb19cd9a1b2d62dcfca6b32e7d170

```



### Create  Cluster

#### Management

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
```

```
nkp-rocky-9.5-release-1.31.4-20250609205019.qcow2
```

create cli flag

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

```
nkp create cluster nutanix \
--cluster-name=$MGMT_CLUSTER_NAME \
--control-plane-prism-element-cluster=$PE_CLUSTER_NAME \
--control-plane-subnets=$SUBNET \
--control-plane-endpoint-ip=$MGMT_CP_ENDPOINT \
--control-plane-vm-image=$VM_IMAGE \
--control-plane-vcpus=4 \
--control-plane-cores-per-vcpu=1 \
--control-plane-memory=12 \
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



#### Managed

env

```
export MANAGED_CLUSTER_NAME="netfos-nkp-dev"
export WORKSPACE_NAMESPACE="netfos-dev"
export MANAGED_CP_ENDPOINT="172.16.90.208"
export MANAGED_LB_IP_RANGE="172.16.90.209-172.16.90.211"
export PC_ENDPOINT="172.16.90.75"
export PE_CLUSTER_NAME="NX1365G6PE"
export SUBNET="Netfos_90_Access_IPAM"
export VM_IMAGE="nkp-ubuntu-22.04-1.31.4-20250409073323"
export STORAGE_CONTAINER="Kubernetes-Container"
export NUTANIX_USER="k8sadmin"
export NUTANIX_PASSWORD="P@55w.rd"
export CSI_HYPERVISOR_ATTACHED="true"
export CSI_FILESYSTEM="ext4"
export SSH_USER="nkp"
export SSH_PUBLIC_KEY="/home/nkp/.ssh/id_rsa.pub"
export REGISTRY_MIRROR_URL="https://192.168.102.92:5000/nkp-2.14"
export REGISTRY_MIRROR_USERNAME="nkpuser"
export REGISTRY_MIRROR_PASSWORD="Nutanix/4u"
export REGISTRY_MIRROR_CACERT="/root/harbor-ca.crt"
export REGISTRY_URL="https://192.168.102.92:5000"
export REGISTRY_USERNAME="nkpuser"
export REGISTRY_PASSWORD="Nutanix/4u"
export REGISTRY_CACERT="/root/harbor-ca.crt"
export KUBECONFIG_MGMT="/root/netfos-nkp.conf"
```

create cli flag

```
nkp create cluster nutanix \
--cluster-name=$MANAGED_CLUSTER_NAME \
--control-plane-prism-element-cluster=$PE_CLUSTER_NAME \
--control-plane-subnets=$SUBNET \
--control-plane-endpoint-ip=$MANAGED_CP_ENDPOINT \
--control-plane-vm-image=$VM_IMAGE \
--control-plane-replicas=1 \
--control-plane-vcpus=4 \
--control-plane-cores-per-vcpu=1 \
--control-plane-memory=8 \
--worker-prism-element-cluster=$PE_CLUSTER_NAME \
--worker-subnets=$SUBNET \
--worker-vm-image=$VM_IMAGE \
--worker-vcpus=8 \
--worker-cores-per-vcpu=1 \
--worker-memory=8 \
--worker-replicas=2 \
--csi-storage-container=$STORAGE_CONTAINER \
--kubernetes-service-load-balancer-ip-range=$MANAGED_LB_IP_RANGE \
--endpoint="https://$PC_ENDPOINT:9440" \
--insecure=true \
--registry-mirror-url=$REGISTRY_MIRROR_URL \
--registry-mirror-username=$REGISTRY_MIRROR_USERNAME \
--registry-mirror-password=$REGISTRY_MIRROR_PASSWORD \
--registry-mirror-cacert=$REGISTRY_MIRROR_CACERT \
--registry-url=$REGISTRY_URL \
--registry-username=$REGISTRY_USERNAME \
--registry-password=$REGISTRY_PASSWORD \
--registry-cacert=$REGISTRY_CACERT \
--ssh-username=$SSH_USER \
--ssh-public-key-file=$SSH_PUBLIC_KEY \
--kubeconfig=$KUBECONFIG_MGMT \
--namespace=$WORKSPACE_NAMESPACE \
--dry-run \
--airgapped \
--output=yaml > deploy-nkp-$MANAGED_CLUSTER_NAME.yaml
```

```
nkp create cluster nutanix \
--cluster-name=$MANAGED_CLUSTER_NAME \
--control-plane-prism-element-cluster=$PE_CLUSTER_NAME \
--control-plane-subnets=$SUBNET \
--control-plane-endpoint-ip=$MANAGED_CP_ENDPOINT \
--control-plane-vm-image=$VM_IMAGE \
--control-plane-replicas=1 \
--control-plane-vcpus=4 \
--control-plane-cores-per-vcpu=1 \
--control-plane-memory=8 \
--worker-prism-element-cluster=$PE_CLUSTER_NAME \
--worker-subnets=$SUBNET \
--worker-vm-image=$VM_IMAGE \
--worker-vcpus=8 \
--worker-cores-per-vcpu=1 \
--worker-memory=8 \
--worker-replicas=2 \
--csi-storage-container=$STORAGE_CONTAINER \
--kubernetes-service-load-balancer-ip-range=$MANAGED_LB_IP_RANGE \
--endpoint="https://$PC_ENDPOINT:9440" \
--insecure=true \
--registry-mirror-url=$REGISTRY_MIRROR_URL \
--registry-mirror-username=$REGISTRY_MIRROR_USERNAME \
--registry-mirror-password=$REGISTRY_MIRROR_PASSWORD \
--registry-url=$REGISTRY_URL \
--registry-username=$REGISTRY_USERNAME \
--registry-password=$REGISTRY_PASSWORD \
--ssh-username=$SSH_USER \
--ssh-public-key-file=$SSH_PUBLIC_KEY \
--dry-run \
--airgapped \
--output=yaml > deploy-nkp-$MANAGED_CLUSTER_NAME.yaml
```



command

```
 nkp create cluster nutanix --help
Create a Konvoy cluster in Nutanix

Usage:
  nkp create cluster nutanix [flags]

Flags:
      --acme-email string                                  Email address the ACME server can use to contact you.
      --acme-server string                                 Address of the ACME service issuing the certificates (default: Let's encrypt). (default "https://acme-v02.api.letsencrypt.org/directory")
      --additional-trust-bundle string                     Additional CA trust bundle to use to validate the Prism Central server certificate.
      --airgapped                                          Enable airgapped mode.
      --allow-missing-template-keys                        If true, ignore any errors in templates when a field or map key is missing in the template. Only applies to golang and jsonpath output formats. (default true)
      --aws-service-endpoints string                       Custom AWS service endpoints in a semi-colon separated format: ${SigningRegion1}:${ServiceID1}=${URL},${ServiceID2}=${URL};${SigningRegion2}...
      --certificate-renew-interval int                     The interval number of days Kubernetes managed PKI certificates are renewed. For example, an Interval value of 30 means the certificates will be refreshed every 30 days. A value of 0 disables the feature. (default 0)
      --cluster-hostname string                            Hostname that is used for accessing the cluster's ingresses.
  -c, --cluster-name name                                  Name used to prefix the cluster and all the created resources.
      --control-plane-cores-per-vcpu int32                 The number of cores per vCPU(equivalent to CPU cores) to use in a control plane machine (default 1)
      --control-plane-disk-size int32                      The size of the primary disk (in GiB) of a control plane machine (default 80)
      --control-plane-endpoint-ip ip                       The control plane endpoint address. To use an external load balancer, set to its IP or hostname. To use the built-in virtual IP, set to a static IPv4 address in the Layer 2 network of the control plane machines. [Not for production use: To use a single-machine control plane, set to the IP or hostname of the machine.]
      --control-plane-endpoint-port int32                  The control plane endpoint port. To use an external load balancer, set to its listening port. (default 6443)
      --control-plane-memory int32                         The size of memory (in GiB) of a control plane machine (default 16)
      --control-plane-pc-categories strings                Names of Prism Central categories to associate with control plane resources (VMs, VGs, etc). Example: key1=value1,key1=value2,key2=value2 (default [])
      --control-plane-pc-project string                    Name of Prism Central project to associate with control plane resources (VMs, VGs, etc).
      --control-plane-prism-element-cluster string         Name of the Prism Element cluster to use to create a control plane machine
      --control-plane-replicas int32                       Number of control plane nodes (default 3)
      --control-plane-subnets strings                      Names of Prism Central subnets to use for control plane machines. Example: subnet1,subnet2,subnet3 (default [])
      --control-plane-vcpus int32                          The number of vCPUs(equivalent to CPU sockets) to use in a control plane machine (default 4)
      --control-plane-vm-image string                      Name of OS image to use for control plane machines.
      --csi-file-system string                             File system to use for CSI volumes. Allowed values ["ext4" "xfs"]. (default "ext4")
      --csi-flash-mode                                     If true, will enable flash mode for CSI volumes.
      --csi-hypervisor-attached-volumes                    If true, will enable the hypervisor attached feature for CSI volumes which allows disks to attach directly to the host without using iSCSI. (default true)
      --csi-reclaim-policy string                          Reclaim policy for CSI volumes. Allowed values ["Delete" "Retain"]. (default "Delete")
      --csi-storage-container string                       Name of the Prism Central storage container to associate with the storage class created on the cluster.
      --dry-run                                            Only print the objects that would be created, without creating them.
      --endpoint url                                       Prism Central URL. Accepted formats: host, host:port, http[s]://host[:port]. Accepted host formats: IP, FQDN.
      --extra-sans strings                                 A comma separated list of additional Subject Alternative Names for the API Server signing cert (default [])
      --fips                                               Enable FIPS mode. Note: The OS images used by the cluster must be prepared with FIPS mode enabled.
  -h, --help                                               Help for nutanix
      --http-proxy string                                  HTTP proxy for all nodes in the cluster
      --https-proxy string                                 HTTPS proxy for all nodes in the cluster
      --ingress-ca file                                    Path to file containing the certificate's CA bundle.
      --ingress-certificate file                           Path to file containing certificates for configuring Ingress.
      --ingress-private-key file                           Path to file containing the certificate's private key (PEM).
      --insecure                                           If true, the Prism Central server certificate will not be validated.
      --kind-cluster-image string                          Kind node image for the bootstrap cluster (default "mesosphere/konvoy-bootstrap:v2.14.0")
      --kubeconfig string                                  Path to the kubeconfig for the management cluster. If unspecified, default discovery rules apply. This flag is ignored if used with the --self-managed flag.
      --kubernetes-pod-network-cidr cidr                   The Kubernetes Pod network CIDR to use in the cluster (default 192.168.0.0/16)
      --kubernetes-service-cidr cidr                       The Kubernetes Service CIDR to use in the cluster (default 10.96.0.0/12)
      --kubernetes-service-load-balancer-ip-range string   A hyphen separated IP range to configure the Kubernetes Service Load Balancer provider with. Example: 10.0.0.0-10.0.0.10
      --kubernetes-version string                          Kubernetes version (default "1.31.4")
  -n, --namespace string                                   If present, the namespace scope for this CLI request. (default "default")
      --no-proxy strings                                   No Proxy list for all nodes in the cluster (default [])
  -o, --output string                                      Output format. One of: (json, yaml, name, go-template, go-template-file, template, templatefile, jsonpath, jsonpath-as-json, jsonpath-file).
      --output-directory string                            Used with --output=json|yaml. The directory where to output resources to files. The directory must already exist.
      --registry-cacert file                               Path to file containing the CA certificate used to verify the registry server certificate
      --registry-mirror-cacert file                        Path to file containing the CA certificate used to verify the registry mirror server certificate
      --registry-mirror-password string                    Password used to authenticate with the registry mirror
      --registry-mirror-url url                            URL of a container registry used as a mirror
      --registry-mirror-username string                    Username used to authenticate with the registry mirror
      --registry-password string                           Password used to authenticate with the registry
      --registry-url url                                   URL of a container registry
      --registry-username string                           Username used to authenticate with the registry
      --self-managed                                       When set to true, the required prerequisites are created before creating the cluster and the resulting cluster has all necessary components deployed onto itself, so it can manage its own cluster lifecycle. When set to false, a management cluster is used. (default false)
      --show-managed-fields                                If true, keep the managedFields when printing objects in JSON or YAML format.
      --ssh-public-key-file string                         Path to the authorized SSH key for the user
      --ssh-username string                                Name of the user to create on the instance (default "konvoy")
      --template string                                    Template string or path to template file to use when -o=go-template, -o=go-template-file. The template format is golang templates [http://golang.org/pkg/text/template/#pkg-overview].
      --timeout duration                                   The length of time to wait before giving up. Zero means wait forever (e.g. 300s, 30m, 3h). (default 30m0s)
      --vm-image string                                    Name of OS image to use for all machines.
      --wait                                               If true, wait for operations to complete before returning. This flag is ignored and will always be 'true' if used with the --self-managed flag. (default true)
      --with-aws-bootstrap-credentials                     Set true to use AWS bootstrap credentials from your environment. When false, the instance profile of the EC2 instance where the CAPA controller is scheduled on will be used instead.
      --with-gcp-bootstrap-credentials                     Set true to use GCP bootstrap credentials from your environment. When false, the service account of the VM instance where the CAPG controller is scheduled on will be used instead.
      --worker-cores-per-vcpu int32                        The number of cores per vCPU(equivalent to CPU cores) to use in a worker machine (default 1)
      --worker-disk-size int32                             The size of the primary disk (in GiB) of a worker machine (default 80)
      --worker-memory int32                                The size of memory (in GiB) of a worker machine (default 32)
      --worker-pc-categories strings                       Names of Prism Central categories to associate with worker resources (VMs, VGs, etc). Example: key1=value1,key1=value2,key2=value2 (default [])
      --worker-pc-project string                           Name of Prism Central project to associate with worker resources (VMs, VGs, etc).
      --worker-prism-element-cluster string                Name of the Prism Element cluster to use to create a worker machine
      --worker-replicas int32                              Number of workers (default 4)
      --worker-subnets strings                             Names of Prism Central subnets to use for worker machines. Example: subnet1,subnet2,subnet3 (default [])
      --worker-vcpus int32                                 The number of vCPUs(equivalent to CPU sockets) to use in a worker machine (default 8)
      --worker-vm-image string                             Name of OS image to use for worker machines.

Global Flags:
  -v, --verbose int   Output verbosity
```



## Upgrade

### nkp 2.15 to nkp 2.16

nkp 2.16 原廠開始提供除了 Rocky Linux 還有 Ubuntu 的 Image ，可以不用另外透過 NIB去製作，而且有先做 CIS Hardened

另外 NKP Bundle 也提供一整包，含 NKP CLI , KIB  , Airgaped bundle，簡化下載流程

在 Nutanix 環境的升級流程為 Management Cluster --> Manged Cluster ，先升 Kommander --> Node Pools ( k8s version )

NKP 2.15 為 v1.32.3 ， NKP 2.16 為 v1.33.2

![image-20250917144337397](https://kenkenny.synology.me:5543/images/2025/09/image-20250917144337397.png)

![image-20250917144556827](https://kenkenny.synology.me:5543/images/2025/09/image-20250917144556827.png)

1. Download images

   ![image-20250917144500467](https://kenkenny.synology.me:5543/images/2025/09/image-20250917144500467.png)

2. Download NKP Bundle 

   ```
   $ cd /home/nkp/nkp-download
   $ curl -o "nkp-bundle_v2.16.0_linux_amd64.tar.gz" "https://download.nutanix.com/downloads/nkp/v2.16.0/nkp-bundle_v2.16.0_linux_amd64.tar.gz?Expires=1758125878&Key-Pair-Id=APKAJTTNCWPEI42QKMSA&Signature=Od2SbuxKJwjUorxB6LXqJaWvzKUwzMRqx6UbbIDieYkhenjElVK-ITjR43Z2PhfdHhMSBSZkJYwko~NQzf0scLNp41Uy-hqOVceH744lqSG7zEgVV9wfBjkgKGnrdovF7Jjkk15MmR8-AH9hoxkd--LMkvNBxaVeEU17~iEHBMyAhc0QoRX4TdYEa-OhWOGR5yZsYsuxaujx7lmMsO9ljB-lLtl1pHYy6LUq1wKnk922m0MZRtfITJ~EUbEsb23mFIuueUveHBlynG8hf8Zz-FnsoDBQnJ-OTcta9zrhqwFePtp5Sgp1i5yrakK7tviUP3lMCjwr~O6npeRNZoNWMQ__"
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
   100 19.9G  100 19.9G    0     0  19.2M      0  0:17:43  0:17:43 --:--:-- 19.2M
   
   [nkp@ken-rhel9 nkp-download]$ ll
   total 20917612
   -rw-r--r-- 1 nkp nkp 21419634634 Sep 17 14:35 nkp-bundle_v2.16.0_linux_amd64.tar.gz
   
   [nkp@ken-rhel9 nkp-download]$ tar xvf nkp-bundle_v2.16.0_linux_amd64.tar.gz 
   nkp-v2.16.0/container-images/
   nkp-v2.16.0/kib/
   nkp-v2.16.0/cli/
   nkp-v2.16.0/application-repositories/
   nkp-v2.16.0/NOTICES
   nkp-v2.16.0/nkp-image-builder-image-v2.16.0.tar
   nkp-v2.16.0/kubectl
   nkp-v2.16.0/konvoy-bootstrap-image-v2.16.0.tar
   nkp-v2.16.0/container-images/konvoy-image-bundle-v2.16.0.tar
   nkp-v2.16.0/container-images/kommander-image-bundle-v2.16.0.tar
   nkp-v2.16.0/kib/overrides/
   nkp-v2.16.0/kib/bundles/
   ...
   nkp-v2.16.0/cli/NOTICES
   nkp-v2.16.0/cli/nkp
   nkp-v2.16.0/application-repositories/kommander-applications-v2.16.0.tar.gz
   
   # 替換成新的 cli
   cp /home/nkp/nkp-download/nkp-v2.16.0/kib/konvoy-image /usr/local/bin/
   cp /home/nkp/nkp-download/nkp-v2.16.0/kubectl
   cp /home/nkp/nkp-download/nkp-v2.16.0/cli/nkp /usr/local/bin/
   
   nkp version
   catalog: v0.7.0
   diagnose: v0.12.0
   imagebuilder: v2.16.0
   kommander: v2.16.0
   konvoy: v2.16.0
   mindthegap: v1.22.1
   nkp: v2.16.0
   
   kubectl version
   Client Version: v1.32.3
   Kustomize Version: v5.5.0
   Server Version: v1.32.3
   [root@ken-rhel9 ~]# cp /home/nkp/nkp-download/nkp-v2.16.0/kubectl /usr/local/bin/
   cp: overwrite '/usr/local/bin/kubectl'? y
   [root@ken-rhel9 ~]# kubectl version
   Client Version: v1.33.2
   Kustomize Version: v5.6.0
   Server Version: v1.32.3
   
   ```

   

3. Create Repository 

   ![image-20250917144859361](https://kenkenny.synology.me:5543/images/2025/09/image-20250917144859361.png)

4. Push Bundle (Konvoy and Kommander)

   ```
   export REGISTRY_URL="http://172.16.90.206/nkp-2.16"
   export REGISTRY_USERNAME="admin"
   export REGISTRY_USERNAME="Harbor12345"
   
   # Kommander
   nkp push bundle --bundle /home/nkp/nkp-download/nkp-v2.16.0/container-images/kommander-image-bundle-v2.16.0.tar --to-registry=http://172.16.90.206/nkp-2.16 --to-registry-insecure-skip-tls-verify
   
   # Konvoy-image-bundle
   nkp push bundle --bundle /home/nkp/nkp-download/nkp-v2.16.0/container-images/konvoy-image-bundle-v2.16.0.tar --to-registry=http://172.16.90.206/nkp-2.16 --to-registry-insecure-skip-tls-verify
   
   
   原廠範例
   nkp push bundle --bundle ./container-images/kommander-image-bundle-nkp-version.tar --to-registry=${REGISTRY_URL} --to-registry-username=${REGISTRY_USERNAME} --to-registry-password=${REGISTRY_PASSWORD}
   
   nkp push bundle --bundle ./container-images/konvoy-image-bundle-nkp-version.tar 
   --to-registry=$REGISTRY_URL 
   --to-registry-username=$REGISTRY_USERNAME 
   --to-registry-password=$REGISTRY_PASSWORD
   ```

   ![image-20250917152514884](https://kenkenny.synology.me:5543/images/2025/09/image-20250917152514884.png)

   ![image-20250917152624666](https://kenkenny.synology.me:5543/images/2025/09/image-20250917152624666.png)

5. 原版本

   ![image-20250917152338534](https://kenkenny.synology.me:5543/images/2025/09/image-20250917152338534.png)

6. 修改 Registry Mirror

   ```
   這段官方沒寫要修改，因為 LAB 環境 registry mirror 有設定不一樣的專案
   
   $nkp edit cluster
   cluster.cluster.x-k8s.io/netfos-nkp edited
   
           globalImageRegistryMirror:
             credentials:
               secretRef:
                 name: netfos-nkp-image-registry-mirror-credentials
             url: http://172.16.90.206/nkp-2.16
   ```

   

7. 升級 Kommander (airgapped)
   有啟用 Nutanix insight 要先 uninstall

   ```
   $ cd /home/nkp/nkp-download/nkp-v2.16.0/
   $ ll
   total 4293696
   drwxr-xr-x 2 nkp nkp         51 Sep 17 14:42 application-repositories
   drwxr-xr-x 2 nkp nkp         32 Sep 17 14:42 cli
   drwxr-xr-x 2 nkp nkp         87 Sep 17 14:38 container-images
   drwxr-xr-x 8 nkp nkp        148 Sep 17 14:54 kib
   -rw-r--r-- 1 nkp nkp 2798137344 Sep 10 23:52 konvoy-bootstrap-image-v2.16.0.tar
   -rwxr-xr-x 1 nkp nkp   60129464 Sep 10 23:56 kubectl
   -rw-r--r-- 1 nkp nkp 1431897600 Sep 10 23:53 nkp-image-builder-image-v2.16.0.tar
   -rw-r--r-- 1 nkp nkp  106573188 Sep 10 23:57 NOTICES
   
   (nkp upgrade kommander -v 4 or 6 可以看詳細的過程)
   
   $ nkp upgrade kommander -v 6 --kommander-applications-repository ./application-repositories/kommander-applications-v2.16.0.tar.gz
   ```

   ![image-20250917153940571](https://kenkenny.synology.me:5543/images/2025/09/image-20250917153940571.png)

   ![image-20250917163318001](https://kenkenny.synology.me:5543/images/2025/09/image-20250917163318001.png)

   ![image-20250917165102028](https://kenkenny.synology.me:5543/images/2025/09/image-20250917165102028.png)

   總計時間大約 30 分鐘左右

8. Kommander 升級後 UI

   2.16 有 Pulse 功能，可以選擇啟用

   ![image-20250917165146805](https://kenkenny.synology.me:5543/images/2025/09/image-20250917165146805.png)

   ![image-20250917165250248](https://kenkenny.synology.me:5543/images/2025/09/image-20250917165250248.png)

   ![image-20250917165305197](https://kenkenny.synology.me:5543/images/2025/09/image-20250917165305197.png)

   ![image-20250917165318495](https://kenkenny.synology.me:5543/images/2025/09/image-20250917165318495.png)

   多了 NKP AI RAG  

   ![image-20250917170730560](https://kenkenny.synology.me:5543/images/2025/09/image-20250917170730560.png)

   vGPU  Token Operator ， 因為 2.16 之後可以支援 vGPU

   ![image-20250917170810417](https://kenkenny.synology.me:5543/images/2025/09/image-20250917170810417.png)

9. 升級 NKP Kubernetes Version
   以 Nutanix 環境為例，大部分的 Control Plane 跟 Worker Node 會是同一個 VM Image ，但是也可以各別指定不同的 Image
   使用 Nutanix 預先封裝好的 Ubuntu

   ![image-20250917171113498](https://kenkenny.synology.me:5543/images/2025/09/image-20250917171113498.png)

   ```
   export MANAGEMENT_CLUSTER_NAME=netfos-nkp
   export VM_IMAGE_NAME=nkp-ubuntu-22.04-release-cis-1.33.2-20250811224527.qcow2
   
   nkp upgrade cluster nutanix \
         --cluster-name ${MANAGEMENT_CLUSTER_NAME} \
         --vm-image ${VM_IMAGE_NAME}
   ```

   VM 先建後拆， Control Plane --> Worker Node

   ![image-20250917171534987](https://kenkenny.synology.me:5543/images/2025/09/image-20250917171534987.png)

   ![image-20250917172200666](https://kenkenny.synology.me:5543/images/2025/09/image-20250917172200666.png)

   ![image-20250917175501264](https://kenkenny.synology.me:5543/images/2025/09/image-20250917175501264.png)

10. 升級完成

    ![image-20250917175609342](https://kenkenny.synology.me:5543/images/2025/09/image-20250917175609342.png)

    
