---
title: Nutanix-Notes
description: Nutanix-Notes
slug: Nutanix-Notes
date: 2023-09-23T08:25:49+08:00
categories:
    - Knowledge Base Category
tags:
    - Nutanix
    - KB
    - 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# 筆記區

純粹筆記，隨時更新。


## 查詢interface status

1. 登入一台CVM

2. 下指令查詢 "allssh manage_ovs show_interfaces"

   ```
   nutanix@NTNX-18FM6J330088-B-CVM:172.16.90.66:~$ allssh manage_ovs show_interfaces
   ================== 172.16.90.65 =================
   name  mode link speed
   eth0 10000 True  1000
   eth1 10000 True  1000
   eth2 10000 True 10000
   eth3 10000 True 10000
   ================== 172.16.90.67 =================
   name  mode link speed
   eth0 10000 True  1000
   eth1 10000 True  1000
   eth2 10000 True 10000
   eth3 10000 True 10000
   ================== 172.16.90.66 =================
   name  mode link speed
   eth0 10000 True  1000
   eth1 10000 True  1000
   eth2 10000 True 10000
   eth3 10000 True 10000
   ```


## 確認Cluster Status

```
登入CVM
1. 查看node投票機制
$nodetool -h 0 ring
2. 查看叢集狀態
$cs | grep -v UP   >>>> 有down的會show出來
$cluster status
3. 確認PE上的ata Resiliency Status
or $ncli cluster get-domain-fault-tolerance-status type=node
```

## 關機流程 (換記憶體)

```
登入CVM
1. 查看叢集狀態
$cs |grep -v UP
$nodetool -h 0 ring
2. 關閉CVM
$cvm_shutdown -P now
```



## Move Windows 2003

### 步驟

```
https://portal.nutanix.com/page/documents/kbs/details?targetId=kA00e000000Cr6CCAS

1. MergeIDE.bat
2. MOVE vm data (uncheck preparation, check only data seed)
3. poweroff vm
4. acli change scsi to ide , delete scsi.0
5. boot to windows
6. install virtIO (NetKVM、Balloon、VIOSCSI)
7. acli change hdd bus to pci sata
8. boot again
```

1. 將MergeIDE.bat的資料夾上傳到Windows 2003,並執行MergeIDE.bat
   ![MergeIDE](https://kenkenny.synology.me:5543/images/2023/09/MergeIDE.png)

   

2. 開啟Move將vCenter上的VM移轉到Nutanix Cluster

   ![Move01](https://kenkenny.synology.me:5543/images/2023/09/Move01.png)


   ![Move02](https://kenkenny.synology.me:5543/images/2023/09/Move02.png)

   ![Move03](https://kenkenny.synology.me:5543/images/2023/09/Move03.png)

   ![Move_SelectVM](https://kenkenny.synology.me:5543/images/2023/09/Move_SelectVM.png)

   ![Move_Network](https://kenkenny.synology.me:5543/images/2023/09/Move_Network.png)


   **勾選Bypass Guest Operations**

   ![Move_Prep](https://kenkenny.synology.me:5543/images/2023/09/Move_Prep.png)

   ![Move_prep02](https://kenkenny.synology.me:5543/images/2023/09/Move_prep02.png)

   ![Move_VMsetting](https://kenkenny.synology.me:5543/images/2023/09/Move_VMsetting.png)

   ![Move_Save&Start](https://kenkenny.synology.me:5543/images/2023/09/Move_Save&Start.png)

   

3. 關閉移轉過後的VM

4. ssh 登入Cluster內其中一台CVM

   ```
   執行指令取得scsi.0的硬碟uuid
   $acli vm.get VM_NAME
   $acli vm.get VM_win2003
   ------------scsi to IDE ----------------
   ＄acli vm.disk_create VM_NAME clone_from_vmdisk=uuid bus=ide
   
   ------------delete scsi.0 ----------------
   $acli vm.disk_delete VM_NAME disk_addr=scsi.0
   
   ------------vm.get VM_NAME----------------
   ```

5. 開啟VM,執行Device Clenup

6. 掛載ISO檔,把檔案複製到C槽再安裝Driver

   ```
   ## 開啟裝置管理員新增以下路徑的Driver
   Ballon Controller、PCI Device
   1. Windows_2003_Tools/virtio-win-0.1-81/WLH/x86
   console滑鼠、SCSI Passthrough Controller
   2. Windows_2003_Tools/virtio-win-0.1-81/WLH/x86
   網卡
   3. Windows_2003_Tools/virtio-win-0.1.189/NetKVM/2k3/x86
   USB Device、PCI Device
   4. Windows_2003_Tools/virtio-win-0.1.189/Balloon/2k3/x86
   ```

7. 調整ide to pci or scsi (prefer)

   ```shell
   執行指令取得ide.0的硬碟uuid
   $acli vm.get VM_NAME
   $acli vm.get VM_win2003
   ------------ide to pci ----------------
   ＄acli vm.disk_create VM_NAME clone_from_vmdisk=uuid bus=pci
   
   ------------delete ide.0 ----------------
   $acli vm.disk_delete VM_NAME disk_addr=ide.0
   
   ------------vm.get VM_NAME----------------
   $acli vm.get VM_NAME
   ```

8. boot win2003

## 把Disk Clone到image service

https://next.nutanix.com/ncm-intelligent-operations-formerly-prism-pro-ultimate-26/ahv-vdisk-part-3-creating-an-image-of-or-from-an-existing-vdisk-33686

### EKS on Nutanix

https://rural-gazelle-46d.notion.site/EKS-Anywhere-On-Nutanix-e6e7764e3daa4267a12cefe6c7a4b7e9



## LAB

### Welcome Banner

```shell
<h1 style="text-align:center; color:black;">NetFos Nutanix Lab公告</h1>
<br>
<br>
<p style="text-align:justify; color:blue;">1. 請將您建的虛擬機加上前輟_VM名稱，如Arthur_Jump，方便管理員管理。</p>
<p style="text-align:justify; color:red;">2. 2023-02-06調整網路架構，以能配置VPC為目標。</p>
<p style="text-align:justify; color:black;"> Netfos Nutanix Team 敬上，更新於2023-02-07</p>
<img src="/console/nutanix_welcome.jpg" style="width:400px:height:200px">

---
/home/apache/www/console
nutanix@NTNX-13SM35330024-B-CVM:172.16.90.62:/home/apache/www/console$ ll
total 128
-rwxr-x---.  1 nutanix apache   4800 Mar  2 12:55 access_denied.html
drwxr-x---.  2 nutanix apache   4096 Mar  2 12:55 api
drwxr-x---.  4 nutanix apache   4096 Mar  2 12:55 api-explorer
drwxr-x---.  7 nutanix apache   4096 Mar  2 12:55 app
drwxr-x---.  2 nutanix apache   4096 Mar  2 12:55 dev-mode
drwxr-x---.  2 nutanix apache   4096 Mar  2 12:55 downloads
-rwxr-x---.  1 nutanix apache   2532 Mar  2 12:55 index.html
-rwxr-x---.  1 nutanix apache   2532 Mar  2 12:55 index_qa.html
drwxr-x---. 41 nutanix apache   4096 Mar  2 12:55 lib
drwxr-x---.  5 nutanix apache   4096 Mar  2 12:55 minerva
-rwxr-x---.  1 nutanix apache  18753 Mar  2 12:55 not_found.html
drwxr-x---.  2 nutanix apache   4096 Mar  2 12:55 nutanixapi
-rw-------.  1 nutanix nutanix 48351 Mar  2 16:45 nutanix_welcome.jpg
drwxr-x---.  3 nutanix apache   4096 Mar  2 12:55 prism
drwxr-x---.  2 nutanix apache   4096 Mar  2 12:55 v2api
drwxr-x---.  2 nutanix apache   4096 Mar  2 12:55 vnc

wangken@wangken-MAC ~ % scp -O nutanix@172.16.90.72:/home/apache/www/console/nutanix_welcome.jpg /Users/wangken/Desktop/Nutanix/Notes
Prism Central VM
nutanix@172.16.90.72's password: 
nutanix_welcome.jpg 
```



## 擴充Node Error

1. 新的Node檢查Disk型號

   ```shell
   nutanix@NTNX-23SG3G330009-A-CVM:192.168.208.43:~$ list_disks
   ERROR:root:Could not reach zookeeper
   ERROR:root:Unable to connect to zookeeper session
   Slot  Disk      Model             Serial          Size  
   ----  --------  ----------------  --------------  ------
   1     /dev/sda  MZILG7T6HBLA/A07  S70KNE0W500857  7.7 TB
   2     /dev/sdb  MZILG7T6HBLA/A07  S70KNE0W500855  7.7 TB
   
   nutanix@NTNX-23SG3G330009-A-CVM:192.168.208.43:~$ grep -A6 -B12 MZILG7T6HBLA/A07 /etc/nutanix/hcl.json
           {
               "approved_by": "Nutanix",
               "blacklisted_firmware": [],
               "boot": true,
               "capacity_gb": 7681.51,
               "data": true,
               "diagnostics_test": 1,
               "endurance": 1,
               "include_in_hcl": true,
               "interface": "SAS",
               "manufacturer": "Samsung",
               "metadata": true,
               "model": "MZILG7T6HBLA/A07",
               "model_string": "SAMSUNG PM1653 SAS 7.6TB RI",
               "nand_type": "TLC",
               "recommended_firmware": [],
               "trim_enabled": true,
               "type": "SSD",
               "uuid": "772da35d-4621-4f8c-80bb-2f1a26648be0"
   ```

   ![image-20230830153417504](https://kenkenny.synology.me:5543/images/2023/09/image-20230830153417504.png)

2. 原來的Cluster內檢查/etc/nutanix/hcl.json 是否有此型號

3. 如果沒有，下載最新的Foundation Platform，並解壓縮
   裡面有hcl.json，上傳至CVM並傳到每個Node CVM

4. 重啟genesis 後即可再次Expand Cluster





