---
title: vSphere_on_Nutanix
description: vSphere_on_Nutanix
slug: vSphere_on_Nutanix
date: 2025-03-03T07:39:16+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - vSphere
    - 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# vSphere on Nutanix

透過Nutanix Foundation安裝vSphere ESXi，並執行後續的設定，

Foundation步驟因為很簡單此處就不多加贅述，

原則上會建議先把Nutanix Node加入vCenter，再將Nutanix Cluster建立好，

因為加入vCenter時Node會需要進入維護模式，如有Nutanix Cluster運作時，

變成要一台一台加入vCenter，執行速度上會比較慢。

## 參考文件

https://portal.nutanix.com/page/documents/details?targetId=vSphere-Admin6-AOS-v6_5:vsp-cluster-introduction-vsphere-c.html

## Foundation

1. 準備vSphere ESXi、vCenter的iso檔
2. 利用mac的foundation安裝nutanix
3. 安裝時改用vSphere的iso檔
4. 如不是使用官方Support的版本,要提供md5 check sum
5. 目前支援到vSphere ESXi 7.0u3c , 實測7.0u3g也可以安裝
6. 安裝時間約50~60min

## 安裝後的設定

1. 設定DNS Server

2. 設定NTP Server

3. 刪除Default Container,再新增一個 (因名稱太長)

4. 設定Virtual IP、Data Service IP

5. Deploy vCenter or Registry to vCenter
   ![image-20230117115309394](https://kenkenny.synology.me:5543/images/2025/03/image-20230117115309394.png)

   ![image-20230117115414091](https://kenkenny.synology.me:5543/images/2025/03/image-20230117115414091.png)

   ![image-20230117115559389](https://kenkenny.synology.me:5543/images/2025/03/image-20230117115559389.png)

   ![image-20230117120202663](https://kenkenny.synology.me:5543/images/2025/03/image-20230117120202663.png)

   ![image-20230117120702169](https://kenkenny.synology.me:5543/images/2025/03/image-20230117120702169.png)

   ![image-20230117121452623](https://kenkenny.synology.me:5543/images/2025/03/image-20230117121452623.png)

   ![image-20230117121557920](https://kenkenny.synology.me:5543/images/2025/03/image-20230117121557920.png)
   
6. vCenter Setting

   - 先在vCenter Create Datacenter並新增Nutanix Cluster
     Cluster **開啟DRS、HA**

   ![image-20230117122218272](https://kenkenny.synology.me:5543/images/2025/03/image-20230117122218272.png)

   ![image-20230117131129201](https://kenkenny.synology.me:5543/images/2025/03/image-20230117131129201.png)

7. **確認Checklist都有設定完畢才能加入Nutanix Node**
   HA

   - Enable Host Monitoring

   - Set the Host Isolation Response of the cluster to **Power Off & Restart VMs**.
     ![image-20230117143553864](https://kenkenny.synology.me:5543/images/2025/03/image-20230117143553864.png)

   - Enable admission control (%)

     ![image-20230117142443552](https://kenkenny.synology.me:5543/images/2025/03/image-20230117142443552.png)

   - Set the VM Restart Priority of all CVMs to **Disabled**. -- After register
     ![image-20230117142551006](https://kenkenny.synology.me:5543/images/2025/03/image-20230117142551006.png)

     ![image-20230117142719050](https://kenkenny.synology.me:5543/images/2025/03/image-20230117142719050.png)

   - Set the VM Monitoring for all CVMs to **Disabled**. -- After register

     ![image-20230117142810049](https://kenkenny.synology.me:5543/images/2025/03/image-20230117142810049.png)

   - Datastore heartbeats -- After register 

     (只有一個Datastore時,增加此設定)
     ![image-20230117134844420](https://kenkenny.synology.me:5543/images/2025/03/image-20230117134844420.png)

     ![image-20230117142910096](https://kenkenny.synology.me:5543/images/2025/03/image-20230117142910096.png)

   DRS

   - Set the Automation Level on all CVMs to **Disabled**. --After register
     ![image-20230117142728280](https://kenkenny.synology.me:5543/images/2025/03/image-20230117142728280.png)

   - Select Automation Level to accept level 3 recommendations.

     ![image-20230117135654108](https://kenkenny.synology.me:5543/images/2025/03/image-20230117135654108.png)

   - Leave power management **disabled**.
     ![image-20230117135332448](https://kenkenny.synology.me:5543/images/2025/03/image-20230117135332448.png)

   Others

   - Configure advertised capacity for the Nutanix storage container (RF2)
     ![image-20230117140918527](https://kenkenny.synology.me:5543/images/2025/03/image-20230117140918527.png) 

   - Store VM swapfiles in the same directory as the VM.

     ![image-20230117143410180](https://kenkenny.synology.me:5543/images/2025/03/image-20230117143410180.png)
     https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.resmgmt.doc/GUID-12B8E0FB-CD43-4972-AC2C-4B4E2955A5DA.html

   - **Enable** enhanced vMotion compatibility (EVC) in the cluster.
     ![image-20230117140459645](https://kenkenny.synology.me:5543/images/2025/03/image-20230117140459645.png)

   - Configure Nutanix CVMs with the appropriate VM overrides. --After register
     (HA、DRS、Monitoring **Disabled**)

8. vCenter Adding Nutanix nodes

   ![image-20230117141404460](https://kenkenny.synology.me:5543/images/2025/03/image-20230117141404460.png)

   ![image-20230117141430544](https://kenkenny.synology.me:5543/images/2025/03/image-20230117141430544.png)

   ![image-20230117141503268](https://kenkenny.synology.me:5543/images/2025/03/image-20230117141503268.png)

   ![image-20230117141525157](https://kenkenny.synology.me:5543/images/2025/03/image-20230117141525157.png)

   實測第一台Node會被順利加到Cluster內,其他兩台需手動移入
   ![image-20230117142411214](https://kenkenny.synology.me:5543/images/2025/03/image-20230117142411214.png)

9. 補齊Checklist的設定

10. Register Nutanix Cluster to vCenter
    ![image-20230117143835885](https://kenkenny.synology.me:5543/images/2025/03/image-20230117143835885.png)

    ![image-20230117143908067](https://kenkenny.synology.me:5543/images/2025/03/image-20230117143908067.png)

    ![image-20230117144024141](https://kenkenny.synology.me:5543/images/2025/03/image-20230117144024141.png)



## 安裝後的設定-PC

https://portal.nutanix.com/page/documents/details?targetId=Acropolis-Upgrade-Guide:upg-vm-install-wc-r.html

1. create PC (可用Online下載或是部署時上傳PC的檔案)
   <img src="https://kenkenny.synology.me:5543/images/2025/03/image-20230117144856189.png" alt="image-20230117144856189" style="zoom:50%;" />

   <img src="https://kenkenny.synology.me:5543/images/2025/03/image-20230117144919733.png" alt="image-20230117144919733" style="zoom:50%;" />

   <img src="https://kenkenny.synology.me:5543/images/2025/03/image-20230117145119576.png" alt="image-20230117145119576" style="zoom:50%;" />
   

   ![image-20230117160527103](https://kenkenny.synology.me:5543/images/2025/03/image-20230117160527103.png)

2. PC建立好後即可用web連線
   預設帳密 admin , Nutanix/4u
   若無法登入web,可以在PE開PCVM的console,用nutanix / nutanix/4u的帳密登入
   登入後切換身份為root 執行以下指令來重設admin密碼

   ```
   ncli user reset-password user-name=admin password=password
   ```

   ![image-20230117162037651](https://kenkenny.synology.me:5543/images/2025/03/image-20230117162037651.png)
   
3. 註冊Nutanix Cluster至Prism Central (PC)
   ![image-20230117162121479](https://kenkenny.synology.me:5543/images/2025/03/image-20230117162121479.png)
   ![image-20230117162141769](https://kenkenny.synology.me:5543/images/2025/03/image-20230117162141769.png)

   ![image-20230117162216948](https://kenkenny.synology.me:5543/images/2025/03/image-20230117162216948.png)
   
4. 註冊完成即可在PC上看到Nutanix Cluster
   ![image-20230117162314773](https://kenkenny.synology.me:5543/images/2025/03/image-20230117162314773.png)

5. Prism Central 設定NTP、DNS Server

6. Enable Microserivces Infrastructure (可使用更多功能)
   https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Guide-vpc_2022_6:mul-cmsp-overview-pc-c.html


   ![image-20230117162822002](https://kenkenny.synology.me:5543/images/2025/03/image-20230117162822002.png)



# vSphere Settings Checklist

https://portal.nutanix.com/page/documents/details?targetId=vSphere-Admin6-AOS-v6_5:vsp-vcenter-settings-r.html

Review the following checklist of the settings that you have to configure to successfully deploy vSphere virtual environment running Nutanix Enterprise cloud.

## vSphere Availability Settings

- **Enable** host monitoring.

- **Enable** admission control and use the percentage-based policy with a value based on the number of nodes in the cluster.

  For more information about settings of percentage of cluster resources reserved as failover spare capacity, [vSphere HA Admission Control Settings for Nutanix Environment](https://portal.nutanix.com/page/documents/details?targetId=vSphere-Admin6-AOS-v6_5:vsp-cluster-settings-admissioncontrol-vcenter-vsphere-r.html).

- Set the VM Restart Priority of all CVMs to **Disabled**.

- Set the Host Isolation Response of the cluster to **Power Off & Restart VMs**.

- Set the VM Monitoring for all CVMs to **Disabled**.

- Enable datastore heartbeats by clicking **Use datastores only from the specified list** and choosing the Nutanix NFS datastore.

  If the cluster has only one datastore, click **Advanced Options** tab and add das.ignoreInsufficientHbDatastore with **Value** of true.

## vSphere DRS Settings

- Set the Automation Level on all CVMs to **Disabled**.
- Select Automation Level to accept level 3 recommendations.
- Leave power management **disabled**.

## Other Cluster Settings

- Configure advertised capacity for the Nutanix storage container 
  (total usable capacity minus the capacity of one node for replication factor 2 or two nodes for replication factor 3).
- Store VM swapfiles in the same directory as the VM.
- **Enable** enhanced vMotion compatibility (EVC) in the cluster. For more information, see [vSphere EVC Settings](https://portal.nutanix.com/page/documents/details?targetId=vSphere-Admin6-AOS-v6_5:vsp-cluster-settings-evc-vcenter-vsphere-t.html).
- Configure Nutanix CVMs with the appropriate VM overrides. For more information, see [VM Override Settings](https://portal.nutanix.com/page/documents/details?targetId=vSphere-Admin6-AOS-v6_5:vsp-cluster-settings-override-vcenter-vsphere-t.html).
- Check [Nonconfigurable ESXi Components](https://portal.nutanix.com/page/documents/details?targetId=vSphere-Admin6-AOS-v6_5:vsp-node-components-unconfigurable-vsphere-r.html). Modifying the nonconfigurable components may inadvertently constrain performance of your Nutanix cluster or render the Nutanix cluster inoperable.

