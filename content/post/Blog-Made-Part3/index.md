---
title: 製作Blog Part3 (半自動腳本)
description: 利用Shell Script將md檔案複製到Blog路徑並上傳Github
slug: blog-part3
date: 2023-09-23T15:47:05+08:00
categories:
    - Knowledge Base Category
tags:
    - Blog
    - KB
    - Linux
    - Shell Script
    - Automation
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---



# 製作Blog Part3

**工欲善其事，必先利其器**

我依照我目前的使用情境寫了三個Shell Script，

來達到半自動上傳筆記到Github的效果，

為何是半自動呢? 因為寫筆記時覺得還要特別加上Hugo的Front Matter有點麻煩，

所以用後製的方式來撰寫Front Matter，而Script執行過程會有交互式的問答，

如此一來就可以順便填入Date、Tags、Category等參數，

而有時候也會有只執行其中一個腳本的時候，所以分了三個來寫。



## 半自動上傳筆記到Github

### 資料夾結構

我的資料夾會依照不同的廠牌or產品來分類，

所以每個廠牌資料夾內都會有個Notes的資料夾，

目前總共分為SUSE、Redhat、Nutanix等三個資料夾，

例如：

```shell
wangken@wangken-MAC Desktop % tree -L 1 RedHat 
RedHat
├── Certification
├── Courses
├── Document
├── Notes
├── iso
```

再來會有個工作目錄是Pull下來Repository的上層資料夾，

會在上面執行腳本。

工作目錄：

```shell
wangken@wangken-MAC myblog % pwd
/Users/wangken/myblog
wangken@wangken-MAC myblog % ls -l
total 32
-rwxr-xr-x@  1 wangken  staff  4342  9 23 17:35 cp_to_blog.sh
-rwxr-xr-x@  1 wangken  staff   499  9 23 17:44 deploy-blog.sh
-rwxr-xr-x   1 wangken  staff   457  9 19 15:20 gitpush.sh
drwxr-xr-x  16 wangken  staff   512  9 21 10:26 hugo-blog
```

### Script 1 複製到工作目錄

這個稍微有點戎長，但也是讓我在腳本執行時看得比較清楚一點

主要內容為：

1. 定義廠牌筆記的資料夾路徑
2. 選擇廠牌的資料夾為何
3. 尋找該資料夾內的.md檔案
4. 利用該檔案的名稱建立文章的資料夾在工作目錄上
5. 輸入Front Matter的參數 ( Category、Tags )
6. 建立含Front Matter的預設index.md在文章的資料夾內
7. 將筆記的內容貼到index.md的下方

cp_to_blog.sh

```shell
#!/bin/bash

# If a command fails then the deploy stops
set -e

# Define Path
blog_path='/Users/wangken/myblog/hugo-blog/content/post/'
ntnx_path='/Users/wangken/Desktop/Nutanix/Notes'
redhat_path='/Users/wangken/Desktop/RedHat/Notes/*'
suse_path='/Users/wangken/Desktop/SUSE/Notes'


# Enter product to identify notes path
echo "Choose product to identify notes path"

pdtype=("suse" "redhat" "nutanix" "Quit")

select type in "${pdtype[@]}"; do
    case $type in 
         "suse")
             pd='suse'
              break
        ;;
         "redhat")
             pd='redhat'
              break
        ;;
         "nutanix")
             pd='nutanix'
          break
            ;;
         "Quit")
        echo "end selection"
        exit
        ;;
         *) echo "invalid option $REPLY";;
         esac
done
echo "choosing $pd to use "
echo "--------------------"
echo "--------------------"
sleep 0.5

# Check if name input with error
# And process the next step
if [[ $pd != 'suse' && $pd != 'rhel' && $pd != 'nutanix' && $pd == '' ]];then

echo "Please check again "

elif [[ $pd == 'suse' && $pd != "" ]];then

echo "Check OK"
echo "--------------------"
echo "--------------------"
echo "Notes directory is $suse_path"
sleep 0.5

echo "--------------------"
echo "--------------------"
echo "Select md file to blog"
sleep 0.5

#Select notes path .md file to blog
files=($suse_path/*.md)

elif [[ $pd == 'redhat' && $pd != "" ]];then

echo "Check OK"
echo "--------------------"
echo "--------------------"
echo "Notes directory is $redhat_path"
sleep 0.5

echo "--------------------"
echo "--------------------"
echo "Select md file to blog"
sleep 0.5

#Select notes path .md file to blog
files=($redhat_path/*.md)

elif [[ $pd == 'nutanix' && $pd != "" ]];then

echo "Check OK"
echo "--------------------"
echo "--------------------"
echo "Notes directory is $ntnx_path"
sleep 0.5

echo "--------------------"
echo "--------------------"
echo "Select md file to blog"
sleep 0.5

#Select notes path .md file to blog
files=($ntnx_path/*.md)

# End loop
fi

PS3='Select file to blog, or 0 to exit: '
select file in "${files[@]}"; do
    if [[ $REPLY == "0" ]]; then
        echo 'Bye!' >&2
        exit
    elif [[ -z $file ]]; then
        echo 'Invalid choice, try again' >&2
    else
        break
    fi
done

# Using file name to path_name 
path_name=$(basename $file .md)
echo "Choosing $path_name"
echo "-------------------"
echo " "

# Create blog post directory
echo "Create blog post directory using $path_name"
echo "-------------------"
echo " "
mkdir -p "$blog_path/$path_name" && post_path=$_
echo "$post_path has been created"
echo "-------------------"
echo " "
echo "Input Front Matter Attribute"
echo "-------------------"
echo " "
sleep 0.5

# Front Matter attribute

# date
now=$(date -u +"%Y-%m-%dT%T+08:00")

# category
categories=("Lab Category" "Knowledge Base Category" "Quit")
select type in "${categories[@]}"; do
    case $type in
         "Lab Category")
                    category='Lab Category'
          break
        ;;
         "Knowledge Base Category")
                    category='Knowledge Base Category'
          break
        ;;
         "Quit")
        echo "end selection"
        exit
        ;;
         *) echo "invalid option $REPLY";;
          esac
done

echo "choosing $category"
echo "-------------------"
echo "-------------------"
echo " "

# tag1
read -p "Input main tag: " tag
echo "-------------------"
echo "-------------------"
echo " "

# tag2
read -p "Input second tag: " tag2
echo "-------------------"
echo "-------------------"
echo " "

# tag3
read -p "Input third tag: " tag3
echo "-------------------"
echo "-------------------"
echo " "

# Generate default index.md
echo "Generate Default Front Matter In $post_path"
echo "-------------------"
echo "-------------------"
sleep 0.5
cat > $post_path/index.md <<EOF
---
title: $path_name
description: $path_name
slug: $path_name
date: $now
categories:
    - $category
tags:
    - $tag
    - $tag2
    - $tag3
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
EOF
echo " "
echo "Copy $file to $post_path"
cat $file >> $post_path/index.md

echo " "
echo "-------------------"
echo "-------------------"
echo "End of the create index.md"
echo " "
sleep 1
```

### Script 2 上傳到Github

主要內容為：

1. 進入工作目錄
2. 新增改變的內容到git ( git add )
3. 確認改變的項目並加入訊息 ( git commit )
4. 推送到Github的Master branch ( git push )

gitpush.sh

```shell
#!/bin/sh

# If a command fails then the deploy stops
set -e

printf "\033[0;32mDeploying updates to GitHub...\033[0m\n"

# Build the project.
# hugo # if using a theme, replace with `hugo -t <YOURTHEME>`

# Go To Public folder
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

# come back zero
cd ..
```

### Script 3 結合Script 1&2

主要內容為：

1. 定義前面兩個Script作為變數
2. 詢問是否要複製筆記 ( 執行Script 1 )
3. 若回答y (yes)，會再問一次，回答n (no)，則進行下一步
4. 詢問是否執行推送到Github上 ( 執行Script 2 )

deploy-blog.sh

```shell
#!/bin/bash

#Define Script
cp=/Users/wangken/myblog/cp_to_blog.sh
push=/Users/wangken/myblog/gitpush.sh

# Ask for copy or not
while true; do
read -p "Do you want to copy md file? (y/n) " yn1
case $yn1 in
    [Yy]* ) $cp;;
    [Nn]* ) break;;
    * ) echo "Please answer yes or no.";;
esac
done
# Ask for Push or not
while true; do
read -p "Do you want to push to github? (y/n) " yn2
case $yn2 in
    [Yy]* ) $push; break;;
    [Nn]* ) break;;
    * ) echo "Please answer yes or no.";;
esac
done
```

### 執行結果截圖

執行deply-blog.sh ，詢問廠牌後會用前面定義好的筆記資料夾來搜尋筆記

![image-20230923174928576](https://kenkenny.synology.me:5543/images/2023/09/image-20230923174928576.png)

填寫參數後選擇自動上傳到github (y)

![image-20230923174957203](https://kenkenny.synology.me:5543/images/2023/09/image-20230923174957203.png)

確認 index.md 前面已自動加入了Front Matter

![image-20230923175024090](https://kenkenny.synology.me:5543/images/2023/09/image-20230923175024090.png)

以上就是我半自動化讓筆記上傳到Github的過程，再加上原先Hugo Stack的範本，

讓部落格網站可以自動更新文章，如有其他建議歡迎不吝指教，謝謝。
