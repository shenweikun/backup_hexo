title: 如何可视化编辑markdown?
categories: 建站
date: 2018-03-14 18:00:31
tags:
---
可視化使用markdown比較簡單，只需安裝插件hexo-admin即可。
<!--more-->  
***
```
前提是你已經按照前面的步驟安裝了node，可是使用npm  
```

#### 安裝hexo-admin 
```
sudo npm install hexo-admin  
```  

#### 編輯 
1.一鍵部署  
a.在根目錄的_config.yml 中添加：
```  
admin:
   username: shenweikun
#   password_hash: a003245582e6ed5da6b999d43a7db320
   secret: hey hexo
   deployCommand: './admin_script/hexo-generate.sh'
#  expire: 60*1
```  
b.在博客根目錄下新增admin_script/hexo-generate.sh，內容如下  
```  
#!/usr/bin/env sh
rm -rf public
hexo g
```  

2.在博客根目錄執行  

``` 
hexo s
```  
會出現如下提示，按照提示在瀏覽器中打開鏈接 *http://localhost:4000/*     
   
```   
[Hexo Admin]: config admin.password_hash is requred for authentication
(node:13168) [DEP0061] DeprecationWarning: fs.SyncWriteStream is deprecated.
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```  

3.在打開的鏈接中默認會進入博客主頁，如要編輯博客，將鏈接地址改爲 *http://localhost:4000/admin/* 即可在其中編輯  
不過在admin中創建的文章不會自動分類，所以我一般直接使用
[hexo+github搭建个人博客——5.如何發布文章](http://shenweikun.top/2018/02/07/2018-02-07-hexo-github%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/#%E5%A6%82%E4%BD%95%E5%8F%91%E5%B8%83%E6%96%87%E7%AB%A0%EF%BC%9F)    
來創建文章，然後在admin中編輯