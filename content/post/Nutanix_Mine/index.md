---
title: Nutanix_Mine
description: Nutanix_Mine
slug: Nutanix_Mine
date: 2023-09-23T03:11:15+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - Veeam
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Mine™ with Veeam Guide 3.0

Nutanix Mine 是Nutanix整合備份軟體讓Nutanix Cluster成為備份專用的Cluster，

以下會測試安裝Nutanix Mine with Veeam，測試備份、還原、升級等。

## Refrence

1. Document
   https://portal.nutanix.com/page/documents/details?targetId=Mine_veeam:Mine_veeam
2. Video (Mine 2.0)
   https://www.youtube.com/watch?v=e13I-VXVaoo
3. Download
   https://portal.nutanix.com/page/downloads?product=mine
4. Veeam
   https://helpcenter.veeam.com/docs/backup/mine/architecture_overview.html?ver=120



## Notice

1. 安裝Mine Foundation VM時,時區要先調成UTC,安裝完後再調回需要的時區。

2. Mine Foundation需要一直保持開機狀態,才可以與AHV Cluster溝通。

3. Mine Cluster只支援AOS LTS版本,STS不支援。

4. 請勿修改或刪除部署後的以下6個VM以及Foundation For Mine。
   Veeam-Win-Node1,Veeam-Win-Node2, Veeam-Win-Node3,Veeam-Lin-Node1, Veeam-Lin-Node2,Veeam-Lin-Node3

5. AHV Cluster建議使用另一個Cluster Admin帳戶,用預設的admin可能會有warning。

6. 部署Mine Cluster目前只支援Windows 2019 Std.

7. 確認DNS設定。

8. 請勿刪除default Storage Container,Mine Cluster會使用到。

   如刪除會出現以下錯誤
   ![image-20230214164432111](https://kenkenny.synology.me:5543/images/2023/09/image-20230214164432111.png)

9. 部署Mine Cluster至少要四個Node。

## 架構

1. Backup Flow

<img src="https://kenkenny.synology.me:5543/images/2023/09/workflow.png" alt="img" style="zoom:48%;" />

2. VM數量:

   Foundation For Mine ***1** (Linux) ,4 vCPUs and 4 GB memory
   Backup Server ***1** (Windows)  , 8 vCPU,16GB memory
   Backup Proxy ***2** or *5 (Node>8) , 8 vCPU,16GB memory
   Backup Ropositories (Linux)* **3** or *6 (Node >8) , 8 vCPU,128GB memory

3. 網路
   <img src="https://kenkenny.synology.me:5543/images/2023/09/image-20230214160103221.png" alt="image-20230214160103221" style="zoom:50%;" />

4. Requirement
   <img src="https://kenkenny.synology.me:5543/images/2023/09/image-20230214165959249.png" alt="image-20230214165959249" style="zoom:50%;" />



## Installation

### 安裝步驟

1. 上架並連上網路。

2. 利用Foundation建立AHV Cluster (CVM建議32GB RAM)。

3. 建立Veeam的Cluster admin account。

4. 下載 NutanixFoundationForMineWithVeeam-v3.0.vmdk。

5. 上傳 NutanixFoundationForMineWithVeeam-v3.0.vmdk 至AHV Cluster。

6. 利用上傳的vmdk建立Mine Foundation VM並開機 (4 vCPU, 4GB RAM)

7. 如有DHCP 直接連線Web https://ip:8743
   如沒有DHCP則開啟VM Console並調整IP ( account/password: veeam/veeam )

   ```
   $sudo nano /etc/network/interfaces
   -----------
   auto eth0
   iface eth0 inet dhcp
   -----------
   ---------->
   -----------
   auto eth0
   iface eth0 inet static
   address yourIpAddress
   netmask yourSubnetMask
   gateway yourAddressAddress
   -----------
   sudo service networking restart
   ```

8. 登入web後點選setup
   ![img](https://kenkenny.synology.me:5543/images/2023/09/main-menu-setup.png)

9. Accept EULA
   ![end user license agreement page](https://kenkenny.synology.me:5543/images/2023/09/cluster-setup-1.png)

10. Configure a virtual network for guest VM interfaces.

11. 輸入Prism Element Cluster的資訊

12. 上傳Windows 2019 Std的iso

13. 上傳Veeam的License

14. 輸入Veeam Server的資訊(用現存的或是部署新的,
    Windows: 8 vCPUs and 16 GB memory,
    Linux: 8 vCPUs and 128 GB memory)

15. 開始部署(1~2Hrs)

16. 輸入Windows License Key

17. 部署完成。

### 安裝後的步驟

1. 新增Volume Group,attach到backup server,並將VG,bakcup server加到protection domain
2. 設定備份VM等Job



# Mine 安裝實作

## Lab架構

![image-20230221112445015](https://kenkenny.synology.me:5543/images/2023/09/Mine.png)

### 資訊欄

1. AHV Cluster , Block SN: 13SM35330024

| 項目                          | SN                  | IPMI MAC          | IPMI IP         | CVM IP       | AHV IP             |
| ----------------------------- | ------------------- | ----------------- | --------------- | ------------ | ------------------ |
| NodeA-AHV1                    | ZM137S025587        | 00:25:90:d3:c8:23 | 10.0.90.2       | 172.16.90.61 | 172.16.90.51       |
| NodeB-AHV2                    | ZM139S125484        | 00:25:90:d8:74:17 | 10.0.90.3       | 172.16.90.62 | 172.16.90.52       |
| NodeC-AHV3                    | ZM137S025585        | 00:25:90:d3:c8:20 | 10.0.90.4       | 172.16.90.63 | 172.16.90.53       |
| NodeD-AHV4                    | ZM137S025604        | 00:25:90:d3:c7:f4 | 10.0.90.5       | 172.16.90.64 | 172.16.90.54       |
| **SSD、HDD per node**         | **Memory per node** | **IPMI Account**  | **IPMI Passwd** | **Account**  | **CVM,AHV Passwd** |
| 400GB SSD *2 <br />1TB HDD *4 | 128GB(16GB *8)      | ADMIN             |                 | nutanix      |                    |

2. Veeam Cluster

   | 項目             | IP                 | Account       | vCPU | Memory |
   | ---------------- | ------------------ | ------------- | ---- | ------ |
   | Foundation-Mine  | 172.16.90.206:8743 | veeam         | 4    | 4      |
   | Veeam-Win-Node1  | 172.16.90.207      | administrator | 8    | 16     |
   | Veeam-Win-Node2  | 172.16.90.208      | administrator | 8    | 16     |
   | Veeam-Win-Node3  | 172.16.90.209      | administrator | 8    | 16     |
   | Veeam-Lin-Node1  | 172.16.90.210      |               | 8    | 128    |
   | Veeam-Lin-Node2  | 172.16.90.211      |               | 8    | 128    |
   | Veeam-Lin-Node3  | 172.16.90.212      |               | 8    | 128    |
   | Helper Appliance | 172.16.90.215      |               |      |        |
   | Backup Proxy     | 172.16.90.216      | veeam         | 4    | 4      |

   ![image-20230214181548005](https://kenkenny.synology.me:5543/images/2023/09/image-20230214181548005.png)

   資源不足無法安裝
   
   ![image-20230214182149519](https://kenkenny.synology.me:5543/images/2023/09/image-20230214182149519.png)
   
   加裝記憶體到196GB Per Node

### Installation

#### AHV Cluster 

1. Foundation四個Node,安裝成一個AHV Cluster,AOS版本建議是LTS的版本。
2. 設定DNS、NTP等基本設定,以下不贅述。

#### Mine Foundation Install

1. 下載Mine Foundation VM的vmdk (NutanixFoundationForMineWithVeeam-v3.0.vmdk)
   https://portal.nutanix.com/page/downloads?product=mine

2. 將下載後的vmdk上傳到Image Service
   ![Image-service01](https://kenkenny.synology.me:5543/images/2023/09/Image-service01.png)

   ![image-20230218150259550](https://kenkenny.synology.me:5543/images/2023/09/image-service02.png)

3. 建立For Veeam使用的Cluster Admin
   ![image-20230218150843210](https://kenkenny.synology.me:5543/images/2023/09/Create_Account01.png)

   ![image-20230218151100056](https://kenkenny.synology.me:5543/images/2023/09/Create_Account02.png)

4. 利用上傳的vmdk建立Mine Foundation VM並開機 (4 vCPU, 4GB RAM)
   ![image-20230218151259652](https://kenkenny.synology.me:5543/images/2023/09/Create_VM01.png)

   新增上傳好的硬碟

   ![image-20230218151418225](https://kenkenny.synology.me:5543/images/2023/09/Create_VM02.png)

   ![image-20230218151510134](https://kenkenny.synology.me:5543/images/2023/09/Create_VM03.png)

   新增網卡並save
   ![image-20230218151614915](https://kenkenny.synology.me:5543/images/2023/09/Create_VM04.png)

5. 設定Mine Foundation IP (預設是DHCP)
   VM開機後進入Console介面,登入帳密 veeam/veeam
   ![image-20230218151959600](https://kenkenny.synology.me:5543/images/2023/09/Change_IP01.png)

   執行以下指令修改ip

   ```
   $sudo vim /etc/network/interfaces
   ------原始-----
   auto eth0
   iface eth0 inet dhcp
   ------更改後-----
   auto eth0
   iface eth0 inet static
   address yourIpAddress
   netmask yourSubnetMask
   gateway yourAddressAddress
   -----------
   $sudo service networking restart
   ```

   ![image-20230218152415873](https://kenkenny.synology.me:5543/images/2023/09/image-20230218152415873.png)

6. 瀏覽器連線至 Foundation_Mine的8743 port
   https://172.16.90.206:8743
   ![image-20230218152630482](https://kenkenny.synology.me:5543/images/2023/09/Browser8743.png)

7. 預設帳密veeam / veeam ,登入後進行修改
   ![image-20230218152751705](https://kenkenny.synology.me:5543/images/2023/09/Change_Password.png)

8. 登入測試
   ![image-20230218152835564](https://kenkenny.synology.me:5543/images/2023/09/Browser_Login.png)

   

#### Mine Cluster Install

1. 全新部署請選擇 Setup
   ![image-20230218152932005](https://kenkenny.synology.me:5543/images/2023/09/Browser_MainPage.png)

2. 點選接受EULA後下一步
   ![image-20230218153016441](https://kenkenny.synology.me:5543/images/2023/09/Browser_EULA.png)

3. 輸入Prism Element的IP、帳密(前步驟建立的Veeam Account),點選下一步
   ![image-20230218153136261](https://kenkenny.synology.me:5543/images/2023/09/Browser_PE.png)

4. 登入後會進行檢查確認硬體是否合適

   ![image-20230218181053737](https://kenkenny.synology.me:5543/images/2023/09/Check_HW.png)

5. 上傳windows 2019的iso檔,可先上傳至PE
   ![image-20230218181155441](https://kenkenny.synology.me:5543/images/2023/09/Upload_Win2019.png)
   ![image-20230220104444863](https://kenkenny.synology.me:5543/images/2023/09/image-20230220104444863.png)

6. 選擇新增DEPLOY NEW BACKUP&REPLICATION SERVER
   ![image-20230220104517202](https://kenkenny.synology.me:5543/images/2023/09/image-20230220104517202.png)

7. 上傳Veeam License File (可先申請試用版授權來使用)
   ![image-20230220104629567](https://kenkenny.synology.me:5543/images/2023/09/image-20230220104629567.png)

8. 網路設定,共會新增3台windows 、3台Linux VM
   ![image-20230220104835929](https://kenkenny.synology.me:5543/images/2023/09/Mine_Deploy_NW.png)
   ![image-20230220104918728](https://kenkenny.synology.me:5543/images/2023/09/image-20230220104918728.png)

9. 設定windows 認證,官方文件不建議使用windows AD驗證
   ![image-20230220105137321](https://kenkenny.synology.me:5543/images/2023/09/Mine_Windows_Cred.png)

10. review設定
    ![image-20230220105219300](https://kenkenny.synology.me:5543/images/2023/09/Mine_Review.png)

11. 開始進行安裝 (順序：搜集資訊>部署Linux>部署Windows>設定Scale Out Backup Repository)
    ![image-20230220105309758](https://kenkenny.synology.me:5543/images/2023/09/image-20230220105309758.png)

    ![image-20230220105609659](https://kenkenny.synology.me:5543/images/2023/09/image-20230220105609659.png)

    ![image-20230220105914370](https://kenkenny.synology.me:5543/images/2023/09/image-20230220105914370.png)

12. 登入PE可看到VM正在部署
    ![image-20230220110200502](https://kenkenny.synology.me:5543/images/2023/09/Mine_VM.png)
    ![image-20230220110749749](https://kenkenny.synology.me:5543/images/2023/09/Mine_Windows.png)

    透由PowerShell自動部署
    ![image-20230220112550775](https://kenkenny.synology.me:5543/images/2023/09/Mine_Powershell.png)

13. 自動建立Storage Container
    ![image-20230220110529475](https://kenkenny.synology.me:5543/images/2023/09/Mine_Container.png)

14. 自動建立Volume Group
    ![image-20230220110632074](https://kenkenny.synology.me:5543/images/2023/09/Mine_VG01.png)
    ![image-20230220110652147](https://kenkenny.synology.me:5543/images/2023/09/Mine_VG02.png)

15. 部署完成,輸入windows授權啟用,可先略過
    ![image-20230220115041552](https://kenkenny.synology.me:5543/images/2023/09/image-20230220115041552.png)

16. 完成
    ![image-20230220115115335](https://kenkenny.synology.me:5543/images/2023/09/image-20230220115115335.png)

17. 點close後設定時區>settings
    ![image-20230220161551198](https://kenkenny.synology.me:5543/images/2023/09/Mine_TimeZone.png)

18. 設定時區、NTP
    ![image-20230220161838334](https://kenkenny.synology.me:5543/images/2023/09/Mine_timezone02.png)

19. 重新登入PE 
    ![image-20230220162124407](https://kenkenny.synology.me:5543/images/2023/09/Mine_AfterInstall01.png)

20. Dashboard沒出現,可參考以下說明
    https://www.veeam.com/kb4265

    ```
    1. 開啟Foundation_Mine的VM Console,並開啟ssh service
    2. ssh 登入Foundation_mine VM
    3. copy 指令並修改PE的IP、帳號密碼
    4. 執行完成後重新登入PE會看到下方圖一
    5. 關閉Foundation_mine VM的ssh service
    6. ssh登入其中一台CVM並執行指令 allssh sudo cp -rp /home/nutanix/prism/webapps/console/nirvana/ /home/apache/www/console/nirvana/
    7.即可看到圖二的Dashboard
    ```

    圖一

    ![image-20230220163841278](https://kenkenny.synology.me:5543/images/2023/09/Mine_Dashboard01.png)

    圖二
    ![image-20230220164338921](https://kenkenny.synology.me:5543/images/2023/09/Mine_Dashboard02.png)

    更新完後LCM的選項會消失,但還是可以在Setting > Upgrade Software上點選出來
    ![image-20230221135620882](https://kenkenny.synology.me:5543/images/2023/09/Mine_LCM.png)

    ![image-20230221135654995](https://kenkenny.synology.me:5543/images/2023/09/Mine_LCM02.png)

21. 點選Launch Console會連線到Bakcup Server的VM (帳號密碼為Foundation Mine時所建立的)
    ![image-20230220164527387](https://kenkenny.synology.me:5543/images/2023/09/Mine_Launch_Console.png)

22. 新增需備份的Hypervisor平台
    ![image-20230220164806207](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddServer.png)

23. 使用此方式建立的Mine Cluster會新增Nutanix AHV的Plug-in
    ![image-20230220164939182](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddServer02.png)

24. 新增Nutanix AHV Cluster (輸入PE的VIP)

    詳細設定可參考 https://helpcenter.veeam.com/archive/van/21/userguide/overview.html

    ![image-20230220165208683](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddServer03.png)

25. 新增連線帳號密碼
    ![image-20230220165407259](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddServer04.png)

26. 此處會建立一個Helper Appliance VM在目的地的AHV Cluster,需要與Backup Server同網段
    ![image-20230220165823258](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddServer05.png)

27. Helper Appliance建立完成
    ![image-20230220170044692](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddServer06.png)

28. 新增完成
    ![image-20230220170415237](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddServer07.png)

29. 建立Proxy 在備份目標上 , 備份會透過此Proxy來連線AHV Cluster
    (https://helpcenter.veeam.com/archive/van/21/userguide/add_ahv_proxy.html)
    ![image-20230220170511581](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy01.png)

30. 建立新的Backup Proxy Linux VM
    ![image-20230220170802112](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy02.png)

31. 設定Backup Proxy名稱、vCPU、Memory,點選Advance進行設定
    依照同時進行任務的數量來決定資源, EX: 同時進行4個任務就需要4個vCPU Core、4GB Memory
    ![image-20230220171350815](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy03.png)

32. 設定Backup Proxy的網路 (預設使用DHCP)
    ![image-20230220171553883](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy04.png)

33. 設定帳號權限
    ![image-20230220173342620](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy05.png)

34. 設定Backup Repository的權限
    ![image-20230220171705433](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy06.png)

35. 建立完成,跳出DNS Warning,需要在DNS Server上新增兩筆紀錄,veeam-win-node1、NX1365-VeeamProxy的紀錄
    並重新apply backup proxy的設定才會重新連線

    ![image-20230220175809202](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy_Warning.png)

    ![image-20230220175728486](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy_DNS.png)

    Backup Server 也需設定DNS尾碼,或是加一筆記錄至hosts內
    ![image-20230220190134179](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy_DNS02.png)

    同理在Linux Proxy Server也需設定 (/etc/hosts)
    ![image-20230220190559732](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy_DNS03.png)

    ![image-20230220191156878](https://kenkenny.synology.me:5543/images/2023/09/image-20230220191156878.png)

36. Finish
    ![image-20230220175918366](https://kenkenny.synology.me:5543/images/2023/09/Mine_AddProxy_Finish.png)

    

    

#### ADD Backup Job

1. 新增一個Backup Job測試,用ken-jump VM來測試
   ![image-20230220180217763](https://kenkenny.synology.me:5543/images/2023/09/Mine_Add_Job01.png)

2. 會連線到 https://172.16.90.216:8100 (Backup Proxy的IP)
   ![image-20230220180451484](https://kenkenny.synology.me:5543/images/2023/09/Mine_Add_Job02.png)

3. Dashboard畫面,NTP跟時區需要手動再去設定,點右上方齒輪可進行設定
   ![image-20230220191245336](https://kenkenny.synology.me:5543/images/2023/09/Mine_Proxy_Dashboard01.png)

   ![image-20230220191339305](https://kenkenny.synology.me:5543/images/2023/09/Mine_Proxy_Dashboard02.png)
   Backup Server

   ![image-20230221094142493](https://kenkenny.synology.me:5543/images/2023/09/Mine_ManageBKServer.png)
   可備份的Cluster

   ![image-20230221094207472](https://kenkenny.synology.me:5543/images/2023/09/Mine_ManageAHV.png)

4. 新增一個backup job
   ![image-20230220191416200](https://kenkenny.synology.me:5543/images/2023/09/Mine_New_Bakcupjob01.png)

   ![image-20230220191457379](https://kenkenny.synology.me:5543/images/2023/09/Mine_New_backupjob02.png)

5. 選擇備份的VM
   ![image-20230220191612250](https://kenkenny.synology.me:5543/images/2023/09/Mine_NEW_backupjob03.png)

6. 設定備份目的地,及還原點數量...
   ![image-20230220191719197](https://kenkenny.synology.me:5543/images/2023/09/MIne_NEW_backupjob04.png)

7. 設定排程
   ![image-20230220191816190](https://kenkenny.synology.me:5543/images/2023/09/Mine_NEW_backupjob05.png)

8. 完成設定
   ![image-20230220191847506](https://kenkenny.synology.me:5543/images/2023/09/Mine_NEW_backupjob06.png)

9. 查看備份進度
   ![image-20230220192103389](https://kenkenny.synology.me:5543/images/2023/09/Mine_NEW_backupjob07.png)

10. 備份完成
    ![image-20230221093619689](https://kenkenny.synology.me:5543/images/2023/09/Mine_Add_Backupjob.png)

11. Protech VMs可以看到從Veeam執行備份的VM以及NX1365 Cluster上原有的Snapshot
    ![image-20230221093836882](https://kenkenny.synology.me:5543/images/2023/09/Mine_ProtechedVMs.png)

12. Events查看執行紀錄
    ![image-20230221093946234](https://kenkenny.synology.me:5543/images/2023/09/Mine_Events.png)

13. Proxy Server Dashboard畫面
    ![image-20230221092833155](https://kenkenny.synology.me:5543/images/2023/09/Mine_Dashboard03.png)

14. PE上的Mine&Veeam Dashboard
    ![image-20230221093447492](https://kenkenny.synology.me:5543/images/2023/09/Mine_Dashboard04.png)

#### Restore Backup VM

目標：將備份的VM還原至AHV Cluster

1. 在Backup Proxy上的Protected VMs點選Restore
   ![image-20230221094652971](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore01.png)

2. 選擇需要的VM及還原點
   ![image-20230221094813735](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore02.png)

3. 還原模式選擇： 還原至原來的位置(原機還原) or 還原至新的位置(客製化設定)
   ![image-20230221095049681](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore03.png)

   ![image-20230221095116899](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore04.png)

4. 因為是測試還原故選擇Restore to a new location,設定VM Name
   ![image-20230221095225911](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore05.png)

   ![image-20230221095252250](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore06.png)

5. 選擇還原至哪個Storage Container
   ![image-20230221095636955](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore07.png)

6. 選擇網路,測試還原故將網路先斷線
   ![image-20230221095756632](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore08.png)

7. 還原的原因,可略過
   ![image-20230221095856886](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore09.png)

8. 還原Job總覽
   ![image-20230221095935837](https://kenkenny.synology.me:5543/images/2023/09/MIne_Restore10.png)

9. 點選Finish即會開始還原
   ![image-20230221100100496](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore11.png)

10. 目的地的AHV Cluster上會先建立Volume Group,還原完後自動刪除
    ![image-20230221100433777](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore12.png)

11. 備份完成
    ![image-20230221101215579](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore13.png)

12. NX1365 AHV Cluster即可看到該VM,登入確認OK
    ![image-20230221101150100](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore14.png)

13. 在Backup Proxy上的還原只針對整台VM或是Disk的還原
    如要針對檔案或是Application (AD,Exchange,SQL等),則要回到Backup Server上進行還原
    實測Disk or Snapshots皆可進行檔案、Application還原,但Snapshots還原會在PE上建立VM及Volume Group故時間上會較慢
    ![image-20230221101455390](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore15.png)

14. 如下圖為針對檔案的還原
    ![image-20230221101637052](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore16.png)

    可選擇還原至原來位置or複製一份到別地方
    ![image-20230221101747022](https://kenkenny.synology.me:5543/images/2023/09/Mine_Restore17.png)

以上為Nutanix Mine with Veeam的安裝及備份還原實作測試。

### Upgrading

#### Mine

1. 開啟Mine Foundation的UI介面 > Mine Platform
   ![image-20230221135803336](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Mine01.png)

2. 點選Settings

   ![image-20230221135946161](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Mine02.png)

3. 點選check updates 來更新,目前為最新版本故無需更新
   ![image-20230221140032583](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Mine03.png)

#### Backup Proxy

Version: 2.1.396

1. 連線至Backup Proxy UI https://172.16.90.216:8100
   點選右上角齒輪 > Appliance Settings > Updates > Check and view updates

   ![image-20230221130725879](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Proxy01.png)

2. 會出現可更新的項目 Security updates、DoNetCore updates
   ![image-20230221130956650](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_proxy02.png)

   ![image-20230221131514935](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Proxy03.png)

3. 勾選全部並執行更新,勾選自動重新啟動
   需先確認沒有backup jobs or restore jobs正在執行
   ![image-20230221131647251](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Proxy04.png)

   ![image-20230221131904623](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Proxy05.png)

4. 更新完成後可查看更新紀錄
   ![image-20230221131954375](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Proxy06.png)

5. 再次執行備份確認沒問題
   ![image-20230221132606909](https://kenkenny.synology.me:5543/images/2023/09/Mine_Upgrade_Proxy07.png)

#### AOS

AOS如有更新則需要在Mine Foundation的UI介面點選Maintenance > Redeploy Mine Dashboard 
來重新更新UI介面

![image-20230221143205971](https://kenkenny.synology.me:5543/images/2023/09/Mine_AOS.png)

![image-20230221143308653](https://kenkenny.synology.me:5543/images/2023/09/Mine_AOS2.png)

#### Veeam Server

目前版本: 11.0.0.837
更新版本: 12.0.0.1420
Repository版本: 11.0.0.839

![image-20230221145122428](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam01.png)

![image-20230221145705817](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam02.png)

1. 下載最新的Veeam Backup&Replication iso檔
   https://www.veeam.com/backup-replication-download.html?ad=downloads

2. 開啟iso檔,用系統管理員身份執行Setup.exe
   ![image-20230221154214097](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam03.png)

3. 執行Upgrade (需先關閉Veeam的視窗)
   ![image-20230221154303858](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam04.png)

   ![image-20230221154327332](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam05.png)

4. Accept License
   ![image-20230221154854447](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam06.png)

5. 確認更新版本,及自動更新遠端的元件
   ![image-20230221154941326](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam07.png)

6. 更新License (11 to 12)
   ![image-20230221155035130](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam08.png)

7. 選擇帳號
   ![image-20230221155146228](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam09.png)

   SQL
   ![image-20230221155209177](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam10.png)

8. 提示升級可能也會更新Datebase
   ![image-20230221155446853](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam11.png)

9. 升級前的check,Next
   ![image-20230221155539238](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam12.png)

10. 開始升級
    ![image-20230221155645278](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam13.png)

    ![image-20230221155710579](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam14.png)

11. 安裝完畢,需重新啟動
    ![image-20230221163718541](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam15.png)

12. 重新登入Veeam Console後會出現可更新的遠端元件
    ![image-20230221164224318](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam16.png)

    ![image-20230221164251762](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam17.png)

13. 更新遠端的元件,Linux Node Failed: Permission denied

    NX1365-VeeamProxy 更新中
    ![image-20230221164340902](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam18.png)

    Backup Server會直接呼叫Proxy Server的AHV Cluster進行快照、更新

    ![image-20230221165329079](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam22.png)

14. 連線到Foundation Mine來進行更新,點選Update
    ![image-20230221164456511](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam19.png)

    ![image-20230221164557209](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam20.png)

15. Linux Repository更新完畢
    版本11.0.0.839 > 12.0.0.1420

    ![image-20230221165120054](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam21.png)

16. Proxy Server 更新完成
    ![image-20230221165437075](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam23.png)

17. 登入Proxy Server https://172.16.90.216 

    介面已更新
    ![image-20230221165557868](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam24.png)

    ![image-20230221165650105](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam25.png)

18. 版本 v2.1.396 > v4.0.0.1899
    ![image-20230221165938130](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam26.png)

19. 回到PE上的Mine with Veeam Dashboard確認狀態

    Protected Instances 顯示N/A

    ![image-20230221171846726](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam28.png)

20. 回到Foundation Mine上面下載Support Bundle

    ![image-20230221172004727](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam29.png)

21. 下載後會是一個zip檔,解壓縮看到以下目錄

    ![image-20230221172059331](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam30.png)

22. 點開NutanixMineDashboard

    Error "Cannot get restore points from backup ken-jump, because it is encrypted or create by an enterprise plug-in"

    ![image-20230221172215714](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam31.png)

23. 有可能是因為目前12版本太新,有相容性問題
    ![image-20230221180824415](https://kenkenny.synology.me:5543/images/2023/09/Mine_Veeam32.png)

24. 除了Dashboard之外,其他功能顯示皆正常,也可正常備份還原。

    





