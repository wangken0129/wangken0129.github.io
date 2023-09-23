---
title: Nvidia_on_Nutanix
description: Nvidia_on_Nutanix
slug: Nvidia_on_Nutanix
date: 2023-09-23T03:16:38+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - Nvidia
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Nivida On Nutanix AHV

主要測試NVIDIA vGPU是否可在Nutanix AHV上安裝及運作，

要使用 vGPU 則需要先安裝License Server，VM上的vGPU才可正常使用，

vGPU相關的License Server、Host Driver、GuestOS Driver都有相依性，安裝前要先去對應，

以下方法已在客戶端驗證可安裝使用。

## 參考資料

1. Nutanix vGPU 說明&troubleshooting
   https://portal.nutanix.com/page/documents/solutions/details?targetId=TN-2046-vGPU-on-Nutanix:TN-2046-vGPU-on-Nutanix
2. Nvidia License Server Requirement
   https://docs.nvidia.com/license-system/latest/nvidia-license-system-faq/index.html
3. Nvidia License Server  User Guide
   https://docs.nvidia.com/license-system/latest/nvidia-license-system-user-guide/index.html#registering-dls-administrator-user
4. AHV Install Nvidia Driver
   https://portal.nutanix.com/page/documents/details?targetId=NVIDIA-Grid-Host-Driver-For-AHV-Install-Guide:NVIDIA-Grid-Host-Driver-For-AHV-Install-Guide
5. AHV Create VM with vGPU
   https://portal.nutanix.com/page/documents/details?targetId=Web-Console-Guide-Prism:wc-vm-create-acropolis-wc-t.html

## 預先準備

1. 下載AHV Driver 要對應Guest OS Driver
   https://portal.nutanix.com/page/downloads?product=ahv&bit=NVIDIA
   https://ui.licensing.nvidia.com/software
2. 下載License Server，NLS License Server (DLS 新版就用qcow2或是ovf) 
   https://ui.licensing.nvidia.com/software
3. License Key、Nvida Account

**Compatibility Matrix**:

| AOS Version     | AHV Versiob    | Nvidia Version |                       |
| --------------- | -------------- | -------------- | --------------------- |
| AOS 6.5.3 LTS   | 20220304.420   | 15.2           | Download driver below |
| AOS 6.6.2.5     | 20220304.10057 | 15.0           | Download driver below |
| AOS 6.5.2.7 LTS | 20220304.392   | 15.0           | Download driver below |
| AOS 6.6.2       | 20220304.10055 | 15.0           | Download driver below |
| AOS 6.5.2.6 LTS | 20220304.385   | 15.0           | Download driver below |
| AOS 6.6.0.5     | 20220304.10019 | 13.3           | Download driver below |
| AOS 6.5.2 LTS   | 20220304.342   | 13.3           | Download driver below |
| AOS 6.6         | 20220304.10013 | 13.3           | Download driver below |
| AOS 6.5.1.8 LTS | 20220304.336   | 13.3           | Download driver below |
| AOS 5.20.5 LTS  | 20201105.2312  | 13.3           | Download driver below |



## License Server 說明

*A VM obtains a license over the network from an NVIDIA vGPU software license server.*

*The license is “checked out” or “borrowed” when the VM is booted, and returned when the*

*VM is shut down.*
![image-20230616170317746](https://kenkenny.synology.me:5543/images/2023/09/image-20230616170317746.png)

![image-20230616144254708](https://kenkenny.synology.me:5543/images/2023/09/image-20230616144254708.png)

  

```
Before proceeding, ensure that you have a platform suitable for  hosting a DLS virtual appliance or containerized DLS software image. 

- The hosting platform must be a physical host running a supported hypervisor or container orchestration platform. 

- The minimum resource requirements for the VM in which the DLS virtual appliance or DLS container will run is as follows:           

  - Number of vCPUs:  4 
  - RAM: 8 Gbytes 
  - Disk Size: 10 Gbytes 

  For additional guidelines, refer to [Sizing Guidelines for a DLS Appliance](https://docs.nvidia.com/license-system/latest/nvidia-license-system-user-guide/index.html#dls-virtual-appliance-sizing-guidelines). 

- The platform must have a fixed (unchanging) IP  address. The IP address may be assigned dynamically by DHCP or  statically configured, but must be constant. 

- The platform’s date and time must be set accurately. NTP is recommended. 
```



## 執行步驟

### 建立Nvidia 帳號 並匯入Key

1. 在License文件下方點選註冊或是登入
   ![image-20230718095043423](https://kenkenny.synology.me:5543/images/2023/09/image-20230718095043423.png)

2. 登入後即可看到License

   ![anydesk00015](https://kenkenny.synology.me:5543/images/2023/09/anydesk00015.png)

### 安裝License Server

1. 上傳License Server的 qcow2 image

2. 開啟vm 設定 4 vpu 8gb ram

3. 預設使用者 dls_system 登入 ( nls 3.1.0 預設帳號改為dls_admin / welcome )

4. 設定固定IP

   ```
   $ /etc/adminscripts/set-static-ip-cli.sh
   ```

   ![image-20230616153813449](https://kenkenny.synology.me:5543/images/2023/09/image-20230616153813449.png)

5. 註冊DLS Admin User (Cluster架構只需建立一次)
   連線https://dls-vm-ip
   ![image-20230616154122814](https://kenkenny.synology.me:5543/images/2023/09/image-20230616154122814.png)

6. 選擇First time setup，設定dls_admin密碼
   ![image-20230616154243779](https://kenkenny.synology.me:5543/images/2023/09/image-20230616154243779.png)

7. 複製secret，並用dls_admin登入

   ```
   79256b19-ffc6-4617-bec3-b62b0696064d
   密碼: Nutanix/4u
   ```

   ![image-20230616154313559](https://kenkenny.synology.me:5543/images/2023/09/image-20230616154313559.png)

8. 登入後即可註冊在Nvidia 建立的Service Instance，以及上傳License File
   ![image-20230616154446654](https://kenkenny.synology.me:5543/images/2023/09/image-20230616154446654.png)

9. 建立sudo user: rsu_admin
   用dls_admin / welcome  or dls_system 登入License Server 

```
$ /etc/adminscripts/enable_sudo.sh
輸入密碼: Nutanix/4u
```

![image-20230616171148494](https://kenkenny.synology.me:5543/images/2023/09/image-20230616171148494.png)

![image-20230616171359579](https://kenkenny.synology.me:5543/images/2023/09/image-20230616171359579.png)


10. 製作HA Cluster ( optional )，依照上述步驟到步驟4再建立一台License Server
    並在Web 上點選Configure high avaliability並輸入另一台的ip
    要先確認防火牆port有通 ，建議同網段

    ![截圖 2023-06-16 17.19.07](https://kenkenny.synology.me:5543/images/2023/09/nvidia-ha.png)

    ![image-20230616175313349](https://kenkenny.synology.me:5543/images/2023/09/image-20230616175313349.png)

11. Create Cluster 
    ![image-20230616175345183](https://kenkenny.synology.me:5543/images/2023/09/image-20230616175345183.png)

    ![image-20230616175455856](https://kenkenny.synology.me:5543/images/2023/09/image-20230616175455856.png)

12. 登入另一台License Server確認 dls_admin 可以登入，資訊同步
    ![image-20230616175603060](https://kenkenny.synology.me:5543/images/2023/09/image-20230616175603060.png)

### 註冊License Server

1. 下載DLS Instance Token 上傳到Nvidia License Portal 
   點選SERVICE INSTANCES > Register DLS Instance


   ![anydesk00025](https://kenkenny.synology.me:5543/images/2023/09/anydesk00025.png)

2. 上傳Token
   ![anydesk00026](https://kenkenny.synology.me:5543/images/2023/09/anydesk00026.png)

   ![anydesk00027](https://kenkenny.synology.me:5543/images/2023/09/anydesk00027.png)

3. 在Nvidia Portal 建立License Server > CREATE SERVER
   ![anydesk00018](https://kenkenny.synology.me:5543/images/2023/09/anydesk00018.png)

4. 設定名稱、描述
   ![anydesk00020](https://kenkenny.synology.me:5543/images/2023/09/anydesk00020.png)

5. 選擇License 
   ![anydesk00021](https://kenkenny.synology.me:5543/images/2023/09/anydesk00021.png)

6. 建立License Server
   ![anydesk00022](https://kenkenny.synology.me:5543/images/2023/09/anydesk00022.png)

7. 建立好後把SERVER INSTANCES 連結到License Server
   ![anydesk00043](https://kenkenny.synology.me:5543/images/2023/09/anydesk00043.png)

8. 連結好後即可下載License File的bin檔
   ![anydesk00044](https://kenkenny.synology.me:5543/images/2023/09/anydesk00044.png)

   ![image-20230718102025826](https://kenkenny.synology.me:5543/images/2023/09/image-20230718102025826.png)

9. 再將bin檔上傳到地端的License Server
   ![image-20230616154446654](https://kenkenny.synology.me:5543/images/2023/09/image-20230616154446654.png)

10. 即可看到License 已可在地端使用
    ![anydesk00038](https://kenkenny.synology.me:5543/images/2023/09/anydesk00038.png)

    ![anydesk00036](https://kenkenny.synology.me:5543/images/2023/09/anydesk00036.png)

11. 下載Client Config Token 以利日後GuestOS 使用

    ![anydesk00039](https://kenkenny.synology.me:5543/images/2023/09/anydesk00039.png)

    ![anydesk00040](https://kenkenny.synology.me:5543/images/2023/09/anydesk00040.png)

    ![anydesk00041](https://kenkenny.synology.me:5543/images/2023/09/anydesk00041.png)

### AHV 安裝Nvidia Driver 

1. 確認AHV有抓到顯卡

   ```
   root@ahv# lspci | grep -i nvidia
   ```

2. PE 上看到Hardware有無顯卡

3. 上傳lcm_nvidia_5.10.170-2.el7.nutanix.20220304.420-525.105.14.tar.gz 到CVM

4. CVM 執行安裝Driver

   ```
   解壓縮
   nutanix@cvm$ tar -xvzf lcm_nvidia_version.tar.gz
   builds/nvidia-builds/version/
   builds/nvidia-builds/version/metadata.json
   builds/nvidia-builds/version/metadata.sign
   builds/nvidia-builds/version/nvidia-vgpu-version.rpm
   
   執行安裝
   nutanix@cvm$ install_host_package -r driver_package_location                      
   ```

5. 確認VM可能會關機或是被移轉

   ```
   VMs using a GPU must be powered off if their parent host is affected by this
   install. If left running, these VMs will be automatically powered off when
   installation begins on their parent host, and powered back on after the driver
   install is completed.
   
   VMs using vGPU will be migrated if their parent host is affected by this
   install. However, some vGPU VMs might be automatically powered off due to lack
   of resource when installation begins on their parent host, and powered back on
   after the driver install is completed.
   
   Install now (yes/no) yes
   
   Note: If a host already has the same version vGPU driver installed, the install_host_package script skips putting that host in maintenance mode, and also skips the restart of that host.
   ```

6. 這邊選擇no，在沒有GPU的node上面不用安裝

   ```
   2021-05-27 06:51:52,082Z INFO install_host_package:70 Waiting for download...
   2021-05-27 06:51:55,269Z INFO install_host_package:192 No supported GPU hardware found for host 1.1.1.1
   Continue installing on this node anyway?
   Install now (yes/no) no
   ```

7. 確認安裝完畢

   ```
   nutanix@cvm$ hostssh "rpm -qa | grep -i nvidia"
                       
   nutanix@cvm$ ncc health_checks hypervisor_checks gpu_driver_installed_check
                           
   nutanix@cvm$ hostssh nvidia-smi
   ```

   

### 指派vGPU Profile給Guest VM

![image-20230616180448664](https://kenkenny.synology.me:5543/images/2023/09/image-20230616180448664.png)

### Guest VM 安裝Driver 

參考 grid-vgpu-user-guide.pdf

安裝檔案

![image-20230617151001641](https://kenkenny.synology.me:5543/images/2023/09/image-20230617151001641.png)

#### Windows

安裝Driver，裝完會重開機

![image-20230617150418550](https://kenkenny.synology.me:5543/images/2023/09/image-20230617150418550.png)

指定License Server路徑

<img src="https://kenkenny.synology.me:5543/images/2023/09/image-20230616181414038.png" alt="image-20230616181414038" style="zoom:50%;" />

#### Linux

A-Series的顯卡要裝15.1以上的版本

1. ubuntu 安裝方法,裝完重開機

   ```
   ＄apt-get install ./nvidia-linux-grid-525_525.105.17_amd64.deb
   ```

2. 裝完後設定/etc/nvidia/gridd.conf

   ```
   $vi /etc/nvidia/gridd.conf
   # 主要有三項要修改
   ServerAddress=
   ServerPort= #新版預設是443
   FeatureType= #如下方描述
   
   Description: Set Feature to be enabled
   # Data type: integer
   # Possible values:
   # 0 => for unlicensed state
   # 1 => for NVIDIA vGPU (Auto Select)
   # 2 => for NVIDIA RTX Virtual Workstation
   # 4 => for NVIDIA Virtual Compute Server
   ```

3. 把Client_Configuration_Token放到 /etc/nvidia/ClientConfigToken/
   ![image-20230718103324873](https://kenkenny.synology.me:5543/images/2023/09/image-20230718103324873.png)

4. 重啟nvidia-gridd 服務,確認有抓到License

   ```
   $ service nvidia-gridd restart
   $ service nvidia-gridd status
   ```

   

![image-20230617150712476](https://kenkenny.synology.me:5543/images/2023/09/image-20230617150712476.png)



## Troubleshoot

1. 產Log report給Nvidia Support (enterprisesupport@nvidia.com)

   ```
   SSH to your AHV Host 
   Run: nvidia-bug-report.sh
   This will generate the file named: nvidia-bug-report.log.gz
   It will be placed in folder from which you ran the command
   Here is the example:
   https://docs.nvidia.com/grid/11.0/grid-vgpu-user-guide/index.html#capture-config-data-for-bug-report
   
   ```

2. 安裝時有遇到無法取得License：

   建議先檢查Client跟License Server的時區、時間是否正確。

3. License Server設定網卡IP時會出現錯誤訊息：

   建議可用IPAM先配發DHCP，等服務起來再進去Web更改。



