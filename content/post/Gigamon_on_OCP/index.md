---
title: Gigamon_on_OCP
description: Gigamon_on_OCP
slug: Gigamon_on_OCP
date: 2023-09-23T03:50:05+08:00
categories:
    - Lab Category
tags:
    - Gigamon
    - Openshift
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Gigamon UCT on OCP

https://docs.gigamon.com/pdfs/Content/Resources/PDF%20Library/GV-6000-Doc/Universal-Container-Tap-Guide-v60.pdf#page18

安裝Gigamon UCT 在RedHat Openshift上，讓Gigamon可以監看Openshift的流量，

UCT會透過daemon set的方式安裝在每個Node上面，所以需要修改Project的權限。

## 安裝步驟

1. create new project , 並加ssc給此project

   ```
   oc new-project
   oc adm policy add-scc-to-user anyuid -z default
   oc adm policy add-scc-to-user privileged -n gigamon -z default
   ```

2. Gigamon UTC-Controller,修改FM IP、Port (default 443)、External DNS、API等等
   以及image版本,要注意第一個執行目錄會有所不同,可以的話用podman run起來看看

   ```
   [ocpadmin@Bastion gigamon]$ cat uct-controller.yaml
   # Copyright 2020 Gigamon Inc.
   ##################################################################################################
   # UCT service
   ##################################################################################################
   apiVersion: v1
   kind: Service
   metadata:
     name: gigamon-uct-cntlr-service
     labels:
       app: uct-cntlr
       service: gigamon-uct-cntlr-service
     # change the namespace tp match your namespace
     namespace: gigamon
   spec:
     ports:
     - port: 8443
       protocol: TCP
       name: uct-rest
       targetPort: 8443
     - port: 42042
       protocol: TCP
       name: uct-stats
       targetPort: 42042
     selector:
       app: uct-cntlr
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: uct-cntlr-v1
     # change the namespace tp match your namespace
     namespace: gigamon
     labels:
       app: uct-cntlr
       version: v1
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: uct-cntlr
         version: v1
     template:
       metadata:
         labels:
           app: uct-cntlr
           version: v1
       spec:
         # this below serviceAccountName should match with the ServiceAccount
         # name under ClusterRoleBinding section.It is recommended to use
         # a non default name as customers may use name of their choice.
         # non default serviceAccountName should be created using the kubectl.
         # kubectl create serviceaccount <non default name> -n <namespace>
         serviceAccountName: default
         containers:
         - name: uct-cntlr
           image: gigamon/uct-controller:6.2.00-374893
           # Usage:
           # /uct-cntlr
           # <FM IP>
           # <FM REST Svc Port>
           # <UCT-Cntlr REST SVC Port>
           # <mTLS Mode: 1(ON)|0(OFF))
           # <Cert Path>
           # <Cert file>
           # <Pvt Key>
           # <CA-Root>
           command:
             - /uct-controller
             - "192.168.85.197"
             - '443'
             - '8443'
             - '0'
             - "/etc/gcbcerts"
             - "gcb-cert.pem"
             - "gcb-pvt-key.pem"
             - "gcb-ca-root-cert.pem"
           #command: [/giga_setup_init]
           imagePullPolicy: Always
           ports:
           - containerPort: 8443
           - containerPort: 42042
           env:
           # Service name.Shoudl match name specified in metadata section above.
           - name: UCT_CNTLR_SERVICE_NAME
             value: "gigamon-uct-cntlr-service"
           # External LB balancer IP, for controller (FM) to connect to uct-cntlr
           - name: UCT_CNTLR_EXT_IP_DNS
             value: "gigamon-uct.ocp.dynasafelab.local"
           # K8S cluster end-point (typically, master nodes with default port of 6443)
           # example: kubectl cluster-info cmd should give this value
           - name: K8S_CLUSTER_ENDPOINT
             value: "https://api.ocp.dynasafelab.local:6443"
           # This value is upto user to specify. Will accept any value
           - name: FM_FQDN
             value: "fm.dynasafelab.local"
           # Namespace of pod - gets from metadata
           - name: UCT_CNTLR_POD_NAMESPACE
             valueFrom:
               fieldRef:
                 fieldPath: metadata.namespace
           # This value is to enable inventory collection through yaml file.
           # true means enable, false or any other value means disable
           - name: UCT_CNTLR_INVENTORY_COLLECTION_ENABLE
             value: "false"
           # We are specifying HOME dir as /pod-data to work with SCC restrictions
           - name: HOME
             value: "/pod-data"
           # For No mTLS Case, Volume Mount is Optional.
           volumeMounts:
           - mountPath: /etc/gcbcerts
             name: pki-path
           - mountPath: /pod-data
             name: shared-data
         volumes:
         - emptyDir: {}
           name: shared-data
         - emptyDir: {}
           name: pki-path
         imagePullSecrets:
           - name: gig-dock-regkey
   ---
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: pods-list
   rules:
   - apiGroups: [""]
     resources: ["pods","services", "nodes"]
     verbs: ["get","watch","list"]
   - apiGroups: ["apps"]
     resources: ["deployments"]
     verbs: ["list"]
   ---
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: pods-list
   subjects:
   - kind: ServiceAccount
     name: default
     #change the below namespace as per the requirement
     namespace: gigamon
   roleRef:
     kind: ClusterRole
     name: pods-list
     apiGroup: rbac.authorization.k8s.io
   ```

3. Gigamon UTC-Tap , 注意此為Daemonset 所以一定要加scc不然會過不了
   我把原文件的SecurityContext刪除,改用scc
   修改service的名稱 、image版本

   ```
   [ocpadmin@Bastion gigamon]$ cat uct-tap.yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: gigamon-uct
     labels:
       app: gigamon-uct
       version: v1
   spec:
     selector:
       matchLabels:
         app: gigamon-uct
         version: v1
     template:
       metadata:
         labels:
           app: gigamon-uct
           version: v1
       spec:
         restartPolicy: Always
         hostPID: true
         hostNetwork: true
         terminationGracePeriodSeconds: 30
         securityContext: {}
         containers:
           - resources:
               limits:
                 cpu: '1'
                 memory: 512Mi
               requests:
                 cpu: '1'
                 memory: 512Mi
             name: gigamon-uct
             command:
               - /uctapp/uct
               - '1'
               - '1'
               - '3'
               - '0'
               - '55'
             env:
               - name: LD_LIBRARY_PATH
                 value: /usr/lib64
               - name: UCT_DEBUG_MODE
                 value: '0x0A000004'
               - name: UCT_SERVICE_NAME
                 value: UCT_HTTP_SERVICE
               - name: UCT_CNTLR_SVC_DNS
                 value: gigamon-uct-cntlr-service
               - name: UCT_CNTLR_REST_SVC_PORT
                 value: '8443'
               - name: UCT_WORKERNODE_NAME
                 valueFrom:
                   fieldRef:
                     apiVersion: v1
                     fieldPath: spec.nodeName
               - name: UCT_POD_NAMESPACE
                 valueFrom:
                   fieldRef:
                     apiVersion: v1
                     fieldPath: metadata.namespace
               - name: UCT_POD_IP
                 valueFrom:
                   fieldRef:
                     apiVersion: v1
                     fieldPath: status.podIP
               - name: UCT_NOKIA_SCP_VTAP
                 value: '0'
             ports:
               - hostPort: 9443
                 containerPort: 9443
                 protocol: TCP
             imagePullPolicy: IfNotPresent
             volumeMounts:
               - name: socket
                 mountPath: /var/run/containerd/containerd.sock
               - name: shared-data
                 mountPath: /pod-data
                 mountPropagation: None
               - name: map
                 mountPath: /sys/fs/bpf
             terminationMessagePolicy: File
             image: 'gigamon/uct-tap:6.2.00-374893'
         volumes:
           - name: socket
             hostPath:
               path: /var/run/containerd/containerd.sock
               type: ''
           - name: map
             hostPath:
               path: /sys/fs/bpf
               type: ''
           - name: shared-data
             emptyDir: {}
         dnsPolicy: ClusterFirstWithHostNet
   ```

4. UTC-Controller Routes , 此處因爲service endpoint走的是 8443, 所以加上edge的tls方式

   ```
   [ocpadmin@Bastion gigamon]$ cat cntrl-route.yml
   apiVersion: route.openshift.io/v1
   kind: Route
   metadata:
     name: gigamon-uct-cntlr-service
     namespace: gigamon
   spec:
     host: gigamon-uct.apps.ocp.dynasafelab.local
     port:
       targetPort: 8443
     to:
       kind: Service
       name: gigamon-uct-cntlr-service
     tls:
       termination: edge
   ```

5. 最後oc apply -f 上述的yaml檔案

   ```
   [ocpadmin@Bastion gigamon]$ oc apply -f uct-controller.yaml
   [ocpadmin@Bastion gigamon]$ oc apply -f cntrl-route.yml
   [ocpadmin@Bastion gigamon]$ oc apply -f uct-tap.yaml
   ```