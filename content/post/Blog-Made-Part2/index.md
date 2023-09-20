---
title: 製作Blog Part2
description: Hugo + Github Page + Stack Theme
slug: blog-part2
date: 2023-09-20T08:15:00+08:00
categories:
    - Knowledge Base Category
tags:
    - Blog
    - KB
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---



# 製作Blog Part2

前篇寫了 Typora + Picgo + NAS 的筆記製作過程，

接下來要將做的筆記用Hugo + Github Page + Stack Theme 的方式做成一個靜態網站。



## 預先準備

1. **Homebrew**

   Homebrew 安裝參考：https://brew.sh

2. **Git**

   Git 安裝參考：[https://git-scm.com/book/zh-tw/v2/開始-Git-安裝教學](https://git-scm.com/book/zh-tw/v2/開始-Git-安裝教學)



## Hugo

### 簡介

Hugo是一個由Go語言開發的靜態網頁產生器，

老實說跟另一個叫Hexo的產生器比起來，Hexo的主題、中文文件比較多，

以部落格的好看性來說應該是Hexo較優，而且是由台灣人開發出來的，

但Hexo的安裝可能還要考慮Node.js的相依性等，我在安裝時遇到了一些問題懶得去解決，

所以最後就選擇Hugo，相對來說安裝上就簡單很多，反正簡單、好用就好，

可以參考下方的主題來選擇自己要的產生器。

Hugo主題: https://themes.gohugo.io/tags/blog

Hexo主題: https://hexo.io/themes/index.html



### Install Hugo

官方文件：https://gohugo.io/installation/macos/



1. 確認Homebrew版本，推薦Mac用戶都要安裝這個套件軟體

   ```
   wangken@wangken-MAC ~ % brew -v
   Homebrew 4.1.11
   ```

2. 透過Homebrew安裝Hugo，裝好後確認hugo版本

   ```
   wangken@wangken-MAC ~ % brew install hugo
   
   wangken@wangken-MAC ~ % hugo version
   hugo v0.118.2-da7983ac4b94d97d776d7c2405040de97e95c03d+extended darwin/arm64 BuildDate=2023-08-31T11:23:51Z VendorInfo=brew
   ```



### Test Hugo



1. 開啟終端機，新增一個hugo的網站目錄

   ```
   wangken@wangken-MAC ~ % hugo new site test-hugo
   Congratulations! Your new Hugo site was created in /Users/wangken/test-hugo.
   
   Just a few more steps...
   
   1. Change the current directory to /Users/wangken/test-hugo.
   2. Create or install a theme:
      - Create a new theme with the command "hugo new theme <THEMENAME>"
      - Install a theme from https://themes.gohugo.io/
   3. Edit hugo.toml, setting the "theme" property to the theme name.
   4. Create new content with the command "hugo new content <SECTIONNAME>/<FILENAME>.<FORMAT>".
   5. Start the embedded web server with the command "hugo server --buildDrafts".
   
   See documentation at https://gohugo.io/.
   ```

2. cd進入專案目錄 test-hugo

   ```
   wangken@wangken-MAC ~ % cd test-hugo 
   wangken@wangken-MAC test-hugo % ls -l
   total 8
   drwxr-xr-x  3 wangken  staff  96  9 20 12:05 archetypes
   drwxr-xr-x  2 wangken  staff  64  9 20 12:05 assets
   drwxr-xr-x  2 wangken  staff  64  9 20 12:05 content
   drwxr-xr-x  2 wangken  staff  64  9 20 12:05 data
   -rw-r--r--  1 wangken  staff  83  9 20 12:05 hugo.toml
   drwxr-xr-x  2 wangken  staff  64  9 20 12:05 i18n
   drwxr-xr-x  2 wangken  staff  64  9 20 12:05 layouts
   drwxr-xr-x  2 wangken  staff  64  9 20 12:05 static
   drwxr-xr-x  2 wangken  staff  64  9 20 12:05 themes
   ```

3. 啟動hugo，輸入hugo serve

   ```
   wangken@wangken-MAC test-hugo % hugo serve
   Watching for changes in /Users/wangken/test-hugo/{archetypes,assets,content,data,i18n,layouts,static}
   Watching for config changes in /Users/wangken/test-hugo/hugo.toml
   Start building sites … 
   hugo v0.118.2-da7983ac4b94d97d776d7c2405040de97e95c03d+extended darwin/arm64 BuildDate=2023-08-31T11:23:51Z VendorInfo=brew
   
   WARN  found no layout file for "html" for kind "taxonomy": You should create a template file which matches Hugo Layouts Lookup Rules for this combination.
   WARN  found no layout file for "html" for kind "home": You should create a template file which matches Hugo Layouts Lookup Rules for this combination.
   
                      | EN  
   -------------------+-----
     Pages            |  3  
     Paginator pages  |  0  
     Non-page files   |  0  
     Static files     |  0  
     Processed images |  0  
     Aliases          |  0  
     Sitemaps         |  1  
     Cleaned          |  0  
   
   Built in 6 ms
   Environment: "development"
   Serving pages from memory
   Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
   Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
   Press Ctrl+C to stop
   ```

4. 將 http://localhost:1313 貼到瀏覽器上，確認有顯示出 **Page Not Found**

   表示Hugo運作正常，會顯示這個訊息是因為Hugo會需要主題來運作，而且也沒有任何的文章發布

   ![image-20230920121219994](https://kenkenny.synology.me:5543/images/2023/09/image-20230920121219994.png)

   

5. 終端機輸入Control+C 結束hugo網站，到此hugo的安裝就完成了，

   後續我會直接套用Stack主題的模板，如要其他模板可以參考其他文章來建立。

   

## Github Pages



### 申請Github帳號

1. 進入Github官網，點右上角的Sign up，接著輸入必要的資訊即可

   https://github.com/

   ![image-20230920111615527](https://kenkenny.synology.me:5543/images/2023/09/image-20230920111615527.png)

### 套用模板與基本設定

1. 點開Stack 主題模板的連結，下方有說明可以參考

   https://github.com/CaiJimmy/hugo-theme-stack-starter

2. 點擊 Use this template --> Create a new repository

   ![image-20230920122233724](https://kenkenny.synology.me:5543/images/2023/09/image-20230920122233724.png)

3. 在Repository name的地方要輸入 『 github帳號.github.io 』，

   這樣github page的連結才會是根網站像 https://wangken0129.github.io 

   如果輸入其他名稱連結會變成 https://wangken0129.github.io/其他名稱

   ![image-20230920122343907](https://kenkenny.synology.me:5543/images/2023/09/image-20230920122343907.png)

4. 匯入完成後會看到有兩個branch，**master**跟**gh-pages**，

   其他兩個是如果上一步驟有勾選 Include all branches就會複製過來

   ![image-20230920145122008](https://kenkenny.synology.me:5543/images/2023/09/image-20230920145122008.png)

5. 複製過來後原文件說明是可以直接在綠色 Code地方新增一個codespace，這是可以直接在github上面直接去編輯文件

   但因為後續管理文件方便，所以我選擇回到筆電終端機上建立一個資料夾來當作工作目錄。

6. 這邊可以設定好 github page 所用的branch，最點選右邊的Settings --> Pages

   根據下圖的設定，Branch那邊要選擇 **gh-pages**

   ![image-20230920150015698](https://kenkenny.synology.me:5543/images/2023/09/image-20230920150015698.png)

7. 接下來可以在github上先新增一個token，用來在筆電上跟github做溝通，

   點選右上角的圓圈--> Settings

   ![image-20230920150444875](https://kenkenny.synology.me:5543/images/2023/09/image-20230920150444875.png)

8. 拉到最下面有個Developer settings

   ![image-20230920150654570](https://kenkenny.synology.me:5543/images/2023/09/image-20230920150654570.png)

9. Personal access tokens --> Tokens (classic) 新增一組Token，

   這組Token請小心保存，而且只會出現一次，可以根據需求設定到期時間、名稱等

   ![image-20230920150749028](https://kenkenny.synology.me:5543/images/2023/09/image-20230920150749028.png)

### 設定模板

1. 回到剛剛的Github Repository上面，點選 Code --> SSH 並複製下方那串SSH

   ![image-20230920151123036](https://kenkenny.synology.me:5543/images/2023/09/image-20230920151123036.png)

2. 開啟筆電終端機，新增一個資料夾作為工作目錄，並進入此資料夾

   ```shell
   wangken@wangken-MAC ~ % mkdir workspace
   wangken@wangken-MAC ~ % cd workspace 
   wangken@wangken-MAC workspace % 
   ```

3. git 登入，最後面使用 authentication token，就是上面複製的token

   ```shell
   wangken@wangken-MAC workspace % gh auth login
   
   ? What account do you want to log into? GitHub.com
   ? You're already logged into github.com. Do you want to re-authenticate? Yes
   ? What is your preferred protocol for Git operations? HTTPS
   ? Authenticate Git with your GitHub credentials? Yes
   ? How would you like to authenticate GitHub CLI?  [Use arrows to move, type to filter]
     Login with a web browser
   > Paste an authentication token
   ```

   

4. 用git初始化工作目錄並設定branch為master，複製

   ```shell
   wangken@wangken-MAC workspace % git init -b master
   已初始化空的 Git 版本庫於 /Users/wangken/workspace/.git/
   ```

5. git 新增遠端來源並確認

   ```shell
   wangken@wangken-MAC workspace % git remote add origin git@github.com:wangken0129/wangken0129.github.io.git
   wangken@wangken-MAC workspace % git remote -v
   origin	git@github.com:wangken0129/wangken0129.github.io.git (fetch)
   origin	git@github.com:wangken0129/wangken0129.github.io.git (push)
   
   ```

   

6. 將來源的branch 拉下來，此時就會看到github上有的資料夾以及檔案

   ```shell
   wangken@wangken-MAC workspace % git pull git@github.com:wangken0129/wangken0129.github.io.git 
   remote: Enumerating objects: 247, done.
   remote: Counting objects: 100% (247/247), done.
   remote: Compressing objects: 100% (128/128), done.
   remote: Total 247 (delta 109), reused 195 (delta 68), pack-reused 0
   接收物件中: 100% (247/247), 141.87 KiB | 470.00 KiB/s, 完成.
   處理 delta 中: 100% (109/109), 完成.
   來自 github.com:wangken0129/wangken0129.github.io
    * branch            HEAD       -> FETCH_HEAD
    
    wangken@wangken-MAC workspace % ls -l
   total 32
   -rw-r--r--  1 wangken  staff  1066  9 20 15:16 LICENSE
   -rw-r--r--  1 wangken  staff  2874  9 20 15:16 README.md
   drwxr-xr-x  4 wangken  staff   128  9 20 15:16 assets
   drwxr-xr-x  3 wangken  staff    96  9 20 15:16 config
   drwxr-xr-x  6 wangken  staff   192  9 20 15:16 content
   -rw-r--r--  1 wangken  staff   130  9 20 15:16 go.mod
   -rw-r--r--  1 wangken  staff   199  9 20 15:16 go.sum
   drwxr-xr-x  3 wangken  staff    96  9 20 15:16 static
   ```

   

7. 在config/_default/config.toml內的有幾項需要先設定

   baseurl = ”https://Github帳號名稱.github.io“

   title = "自定義"

   defaultContentLanguage = "zh-tw"

   disqusShortname = "disqus建立的Shortname" (這是部落格底下的留言板功能，可先忽略)

   可以去官網註冊帳號，並建立一個Site，官網： https://disqus.com/profile/signup/intent/

   ```shell
   wangken@wangken-MAC workspace % vi config/_default/config.toml 
   
   # Change baseurl before deploy
   baseurl = "https://wangken0129.github.io"
   languageCode = "en-us"
   paginate = 5
   title = "Ken's blog"
   
   # Theme i18n support
   # Available values: en, fr, id, ja, ko, pt-br, zh-cn, zh-tw, es, de, nl, it, th, el, uk, ar
   defaultContentLanguage = "zh-tw"
   
   # Set hasCJKLanguage to true if DefaultContentLanguage is in [zh-cn ja ko]
   # This will make .Summary and .WordCount behave correctly for CJK languages.
   hasCJKLanguage = false
   
   # Change it to your Disqus shortname before using
   disqusShortname = "wangken0129"
   ```

   

8. 基本設定這些就可以建立網站了，後續的一些設定可以參考官方文件，幾乎都可以在config/_default 目錄底下去設定

   官方文件：https://stack.jimmycai.com/

9. 後續如果有文章想新增的話，根據此主題的架構要新增目錄在content/post裡面

   並依照下方的目錄結構去新增，並且md檔案前面會需要加上front matter

   ~~~shell
   # 文章結構，每篇都會有index.md
   content
   └── post
       └── my-first-post
           ├── index.md
           ├── image1.png
           └── image2.png
           
   # front matter範例，以此文章為範例，放在md檔案最上面前後都要加 ```
   
   ```
   # 文章標題
   title: 製作Blog Part2
   # 文章描述
   description: Hugo + Github Page + Stack Theme
   # slug
   slug: blog-part2
   # 時間，固定格式UTC+08:00
   date: 2023-09-20T07:15:00+08:00
   # 文章分類
   categories:
       - Knowledge Base Category
   # 文章標籤
   tags:
       - Blog
       - KB
   weight: 1       # You can add weight to some posts to override the default sorting (date descending)
   ```
   ~~~

   

10. 設定完成後即可push到github上面，指令如下：

    ```
    git add .
    git commit -m "msg"
    git push --set-upstream origin master
    ```

    

11. 成功push後可以點進Github 上的Repository --> Actions

    這邊就是套用此模板的好處之一，已經自動幫你建立好workflows，讓你不用去做其他設定就可以把網站做好

    ![image-20230920154906316](https://kenkenny.synology.me:5543/images/2023/09/image-20230920154906316.png)

12. 連線確認網站OK

    ![image-20230920160059559](https://kenkenny.synology.me:5543/images/2023/09/image-20230920160059559.png)

## 後記

完成部落格的設定後，再來就要把之前做的筆記套用Front Matter放上來跟之後的撰寫了，

針對Hugo的一些配置檔的調整，我會使用Visual Studio Code來做編輯跟調整，

因為針對.toml檔案的編輯還是比較友善，

像下方的params.toml，我有去修改[sidebar]下方的值，

![image-20230920163349713](https://kenkenny.synology.me:5543/images/2023/09/image-20230920163349713.png)

其他筆記的撰寫我就還是用Typora，左邊可以分為檔案的目錄檢視，

![image-20230920163708312](https://kenkenny.synology.me:5543/images/2023/09/image-20230920163708312.png)

或是切換到大綱檢視，分為文章裡面的大小標題等等，

![image-20230920163841527](https://kenkenny.synology.me:5543/images/2023/09/image-20230920163841527.png)

最後因為會頻繁地做git push的動作，所以我也參考網路的Script來自動化的push，

希望最後是能變成全部都自動化，有的話再寫上來分享。

```
#!/bin/sh

# If a command fails then the deploy stops
set -e

printf "\033[0;32mDeploying updates to GitHub...\033[0m\n"

# Build the project.
# hugo # if using a theme, replace with `hugo -t <YOURTHEME>`

# Go To workspace folder
cd hugo-blog

# Add changes to git.
git add .

# Commit changes.
msg="rebuilding site $(date)"
if [ -n "$*" ]; then
	msg="$*"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin master

# come back
cd ..
```



