title: 怎么把hexo的相关信息修改为自己？
categories: 建站
date: 2018-03-13 19:20:23
tags:
---
使用hexo搭建网站，网站的大部分配置信息都可以在_config.yml文件中进行修改。本编主要介绍_config.yml中的内容。
<!--more-->  
***
### _config.yml介绍
1. 网站配置文件在位置： myblog/_config.yml （即创建网站文件夹的根目录下）
2. [官方配置说明](https://hexo.io/docs/configuration.html) 
3. 注意细节： 配置文件中每个属性的冒号（：）后面应该添加空格
4. 以下用我的配置文件内容为例子进行说明 


#### 网站基本信息配置 
``` 
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
# 博客的标题
title: 空白的博客
# 副标题
subtitle: 需要效率的事情交给机器人就好了，我们真正擅长的是浪费时间。
# 网站的描述，主要用于SEO，告诉搜索引擎一个关于您站点的简单描述，通常建议在其中包含您网站的关键词。
description: 记录生活与职业中的点滴
# 网站的作者
author: yourname
# 语言，默认是英文的，我这里修改为中文zh-CN
language: zh-CN
# 时区，一般留空，默认使用您电脑的时区。时区列表：America/New_York, Japan, 和 UTC 。
timezone:
``` 

#### 网址信息配置
``` 
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
# 修改成你自己的网址，你可以通过此网址来访问你的网站
url: https://yourname.github.io/
root: /
# 文章链接地址格式 。即文章存放的目录。
permalink: :year/:month/:day/:title/
permalink_defaults:
``` 

#### 目录配置 
``` 
# Directory
# 资源文件夹，这个文件夹用来存放内容。
source_dir: source
# 公共文件夹，这个文件夹用于存放生成的站点文件。
public_dir: public
# 标签文件夹，实际位于source/tags
tag_dir: tags
# 归档文件夹
archive_dir: archives
# 分类文件夹
category_dir: categories
# Include code 文件夹
code_dir: downloads/code
# 国际化（i18n）文件夹
i18n_dir: :lang
# 跳过指定文件的渲染，您可使用 glob 表达式来匹配路径。
skip_render:
``` 

#### 写作配置
``` 
# Writing
# 新文章的文件名称
new_post_name: :title.md # File name of new posts
# 预设布局  
default_layout: post
# 是否将标题转换成标题形式（首字母大写）   
titlecase: false # Transform title into titlecase
# 在新标签中打开链接 
external_link: true # Open external links in new tab
# 把文件名称转换为 (1) 小写或 (2) 大写  
filename_case: 0
# 是否渲染草稿
render_drafts: false
# 启动 Asset 文件夹（创建文章的时候会创建同名文件夹，文件夹可以用于存放图片，建议启动）
post_asset_folder: true
# 把链接改为与根目录的相对位址 
relative_link: false
# 显示未来的文章
future: true
# 代码块的设置
highlight:
  enable: true #使能
  line_number: true #显示行号
  auto_detect: false #自动检测语言
  tab_replace:
``` 

#### 主页设置
``` 
# Home page setting
# path: Root path for your blogs index page. (default = '')
# per_page: Posts displayed per page. (0 = disable pagination)
# order_by: Posts order. (Order by date descending by default)
index_generator:
  path: ''
  per_page: 10 
  order_by: -date
``` 

#### 分类&标签 
``` 
# Category & Tag
# 默认分类
default_category: uncategorized
# 分类别名
category_map:
	编程: programming
	生活: life
	其他: other
# 标签别名
tag_map:
``` 

#### 日期和时间格式 
``` 
# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss
``` 

#### 分页设置 
``` 
# Pagination
## Set per_page to 0 to disable pagination
per_page: 10 #每页
pagination_dir: page
``` 

#### 扩展 
``` 
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: jacman  #这里是配置主题，位于myblog/themes/目录下
stylus:
  compress: true
# 部署部分的设置，我这里与我的git相绑定，你们可以改成自己的git地址
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:yourname/yourname.github.io.git
  branch: master
``` 

#### 可视化插件相关 
``` 
admin:
   username: yourname
#   password_hash: 
   secret: hey hexo
   deployCommand: './admin_script/hexo-generate.sh'
#  expire: 60*1
```