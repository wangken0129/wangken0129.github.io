---
title: 製作Blog Part1
description: Typora+NAS＋Picgo
slug: blog-part1
date: Sep 19, 2023 11:45 CST
categories:
    - Knowledge Base Category
tags:
    - Blog
    - KB
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---



# 製作Blog Part1

把過程記錄下來，未來若有更換電腦還可以照著操作復原回來。

原本我只有使用Typora做本地紀錄，有需要再上傳Github或是匯出成 PDF or Word 分享給別人，

後來覺得有時候要用手機查看不太方便，有嘗試使用Notion因為還有手機App可使用，

但是匯入文件的時候出現不少問題故作罷，幾經測試，覺得Github Page+Hugo看起來是比較簡單好上手。



Part1 是Typora作為筆記軟體，並用Picgo上傳圖片到自架的NAS當作圖床空間。

Part2 是透過Hugo + Github Page來做靜態網站，以存放Typora的筆記。

## Typora

我使用Typora作為我的Markdown筆記軟體，操作簡單而且可以匯出很多格式。

### Install Typora

從官網下載下來安裝即可

官網: https://typora.io/



### Setting Typora

有三個步驟: 

1. 輸入授權 or 破解(?)

2. 針對圖片的存放設定，一開始我是使用如下圖的設定，

   複製圖片至Typora時會存放到上層路徑的assets資料夾，後來因為方便性改為上傳到自架的NAS上作為圖床。
   ![image-20230919115931790](https://kenkenny.synology.me:5543/images/2023/09/image-20230919115931790.png)

3. 外觀配置，官網有很多主題可以使用，只要把下載下來的主題複製到主題資料夾即可。![image-20230919125343105](https://kenkenny.synology.me:5543/images/2023/09/image-20230919125343105.png)

完成後就可以開始使用Typora了。

![image-20230919143435523](https://kenkenny.synology.me:5543/images/2023/09/image-20230919143435523.png)



## NAS自架圖床

這步驟如果沒有NAS或是使用其他圖床空間可以省略，

會使用NAS自架圖床也是剛好有NAS，而且Imgur前陣子在大掃除，

其他的圖床空間也難免會有大掃除或是停止的狀況，自架也只建議小型的Blog使用，

如果有大量的圖片需求，還是建議用雲端的像Amazon S3等等。

*NAS設定 DDNS OR DNS 還有憑證網路上有很多參考文件此處就不多加贅述，此處使用的是Synology DDNS 憑證自動更新* 



### Web Station Install

1. 新增共用資料夾 /www/wwwroot/blog/images
   ![image-20230919130918428](https://kenkenny.synology.me:5543/images/2023/09/image-20230919130918428.png)
2. 建議再新增一個使用者設定權限專門使用此資料夾
   ![image-20230919131239449](https://kenkenny.synology.me:5543/images/2023/09/image-20230919131239449.png)
3. 安裝Web Station![image-20230919130741842](https://kenkenny.synology.me:5543/images/2023/09/image-20230919130741842.png)
4. 網頁服務 --> 新增靜態網站，指定主目錄為剛剛新增的資料夾
   ![image-20230919131023338](https://kenkenny.synology.me:5543/images/2023/09/image-20230919131023338.png)
5. 新增網頁入口，對應剛剛新增的網頁服務，並設定好Port號，建議使用HTTPS
   ![image-20230919132005169](https://kenkenny.synology.me:5543/images/2023/09/image-20230919132005169.png)
6. 丟一張圖片到 /www/wwwroot/blog/images
   用網頁連線測試看是否可以讀取
   ![image-20230919131936400](https://kenkenny.synology.me:5543/images/2023/09/image-20230919131936400.png)

### FTP 設定

1. 開啟FTP服務，設定FTPS、Port號，安全性考量不勾選FTP
   ![image-20230919132202926](https://kenkenny.synology.me:5543/images/2023/09/image-20230919132202926.png)
2. 設定指定使用者根目錄，安全考量不啟用匿名登入
   ![image-20230919132448501](https://kenkenny.synology.me:5543/images/2023/09/image-20230919132448501.png)



## Picgo

配置文件
參考原文: https://zhuanlan.zhihu.com/p/382702959
Picgo: https://picgo.github.io/PicGo-Core-Doc/zh/guide/config.html
ftp-uploader: https://github.com/imba97/picgo-plugin-ftp-uploader/tree/master
ftp-basic: https://www.npmjs.com/package/basic-ftp

### Install Picgo

1. 首先要先安裝Node.js，下載安裝或是用brew install node
   https://nodejs.org/en/download

2. 安裝好後確認版本

   ```shell
   wangken@wangken-MAC ~ % npm -v
   9.6.7
   wangken@wangken-MAC ~ % node -v
   v18.17.1
   ```

3. 使用npm安裝Picgo

   ```shell
   wangken@wangken-MAC ~ % npm install picgo -g
   wangken@wangken-MAC ~ % picgo -v     
   1.5.6
   wangken@wangken-MAC ~ % picgo --help
   Usage: picgo [options] [command]
   
   Options:
     -v, --version                        output the version number
     -d, --debug                          debug mode
     -s, --silent                         silent mode
     -c, --config <path>                  set config path
     -p, --proxy <url>                    set proxy for uploading
     -h, --help                           display help for command
   
   Commands:
     install|add [options] <plugins...>   install picgo plugin
     uninstall|rm <plugins...>            uninstall picgo plugin
     update [options] <plugins...>        update picgo plugin
     set|config <module> [name]           configure config of picgo modules
     upload|u [input...]                  upload, go go go
     use [module]                         use modules of picgo
     init [options] <template> [project]  create picgo plugin's development templates
     i18n [lang]                          change picgo language
     help [command]                       display help for command
   ```

4. 安裝好後再新增ftp-uploader

   ```
   wangken@wangken-MAC ~ % picgo install picgo-plugin-ftp-uploader
   
   # added 3 packages in 1s
   # [PicGo SUCCESS]: 插件安装成功
   ```

### Setting Picgo

1. 修改Picgo配置文件，預設在~/.picgo/config.json 

   ```shell
   wangken@wangken-MAC ~ % picgo set uploader
   ? Choose a(n) uploader ftp-uploader
   ? imba97 kenkenny
   ? D:/ftpUploaderConfig.json /Users/wangken/.picgo/ftpUploaderConfig.json
   [PicGo SUCCESS]: Configure config successfully!
   
   
   wangken@wangken-MAC ~ % cat ~/.picgo/config.json   
   {
     "picBed": {
       "uploader": "ftp-uploader",
       "current": "ftp-uploader",
       "ftp-uploader": {
         "site": "kenkenny",
         "configFile": "/Users/wangken/.picgo/ftpUploaderConfig.json"
       }
     },
     "picgoPlugins": {
       "picgo-plugin-ftp-uploader": true
     }
   }%
   ```

2. 新增~/.picgo/ftpUploaderConfig.json
   這裡我多了一個secure的選項，安裝好後預設是沒有這個參數
   因為我是使用FTPS，所以後面的配置檔案也要新增secure的參數，其餘請自行修改NAS對應的路徑及連結

   ```json
   wangken@wangken-MAC .picgo % cat ftpUploaderConfig.json 
   {
     "kenkenny": {
       "url": "https://nas.domain:xxxx",
       "path": "/images/{year}/{month}/{fullName}",
       "uploadPath": "/wwwroot/blog/images/{year}/{month}/{fullName}",
       "host": "nas.domain",
       "port": portnumber,
       "username": "ftp-user",
       "password": "ftp-user-password",
       "secure": true
     }
   }
   ```

3. cd 進入  ~/.picgo/node_modules/picgo-plugin-ftp-uploader/dist
   在await client.access的地方新增 **secure: config.secure**
   詳細配置可以Google "ftp-basic" 就會有文件可以參考了

   ```
   
   wangken@wangken-MAC dist % vim index.js
   
             await client.access({
                 host: config.host,
                 port: config.port,
                 user: config.username,
                 password: config.password,
                 secure: config.secure
             });
   ```

4. 以上配置完成，不用重開機直接進入Typora修改檔案上傳的



## 驗證Typora + Picgo + NAS

1. 首先修改Typora的圖片設定，改為自訂命令，輸入 picgo upload
   ![image-20230919141634956](https://kenkenny.synology.me:5543/images/2023/09/image-20230919141634956.png)

2. 接著新增一個md檔案，複製一張圖片進去即可看到圖片上傳成功
   Typora: 點擊圖片即可看到上傳後的路徑

   ![image-20230919142209291](https://kenkenny.synology.me:5543/images/2023/09/image-20230919142209291.png)
   NAS: 點選連結或是進入NAS的File Station確認圖片
   ![image-20230919142323495](https://kenkenny.synology.me:5543/images/2023/09/image-20230919142323495.png)



以上就是Typora+Picgo上傳圖片到NAS上的設定，下一篇再來進入部落格的製作。
