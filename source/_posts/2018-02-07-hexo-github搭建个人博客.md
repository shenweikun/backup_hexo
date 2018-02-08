title: hexo+github搭建个人博客
categories: 建站
date: 2018-02-07 11:42:45
---
最近年关将至，手头上的工作也落实得七七八八，总而言之就是很闲的意思。于是乎，开始搭建一个属于自己的个人博客。搭建个人博客的想法并不是突发奇想，而是一直都存在，不过苦于没有时间和精力，便一直搁置着。趁着这段空闲的时间，就来实现一下自己的小梦想啦。
  
本文主要用于记录搭建本博客的过程，以便日后查阅，也给需要的同学留点小参考。
***
```
基于系统：Ubuntu16.04
```

### 前言
虽然从事IT行业，但是从来没有接触过web相关的知识，所以希望寻找一种相对简单的方式来搭建自己的博客。搜索了网络上大量的资料，最终选择了这组组合hexo+github,原因如下：  
1.搭建部署简单，完全不用担心写代码，稍微有点计算机基础的同学都能完成  
2.基于Markdown编写文章，简洁快速，不需考虑文章排版  
3.有许多优秀的主题模板可供选择  
4.免费！不需要服务器，不需要后台（让我们为互联网的免费学习开源精神干杯！）  
等等   

大概可以分为以下几个步骤：  
1.搭建环境准备  
2.安装配置Hexo  
3.配置主题  
4.申请github账号，并将Hexo与github pages联系起来  
5.如何发布文章？
***
### 环境搭建准备
大概分为以下三步：  
* git安装与配置 
* Node.js安装  

#### git安装与配置
GitHub 肯定是要 Git 的。  
* git从源中安装 （推荐）
```
sudo apt-get update 获得最近的软件包的列表； 
sudo apt-get install git
git --version
```
* 安装完成后，需要进行git基本配置
```
git config –global user.name “这里写你的名字” 
git config –global user.email “这里写你的邮箱地址”
```
* 配置完成后，需要创建验证用的公钥，因为git是通过ssh的方式访问资源库的，所以需要在本地创建验证用的文件。  
执行命令：
```
ssh-keygen -C '写上你的邮箱地址' -t rsa
```
完成后，会在用户目录~/.ssh/下建立相应的密钥文件*id_rsa.pub*。这个公钥非常重要，在后面配置github的时候会用到。


#### Node.js安装  
Hexo 是 Node.js 写的，所以需要先安装好 Node.js  
* Node.js源码安装  （推荐）  
1.在 Github 上获取 Node.js 源码：
```
sudo git clone https://github.com/nodejs/node.git
```
2.修改目录权限
```
 sudo chmod -R 755 node
```
3.使用 ./configure 创建编译文件，按照：
```
 cd node
 sudo chmod +x configure
 sudo ./configure
 sudo make
 sudo make install
```
4.查看Node版本
```
 node --version
```
* apt-get命令安装 （不推荐，命令看起来简单，但安装过程经常发生问题）
```
 sudo apt-get install nodejs
 sudo apt-get install npm
```
***
### 安装配置Hexo
安装好Node.js之后npm管理工具了，可以使用npm命令来安装Hexo
* 使用 npm 来便捷安装 Hexo：
```
 npm install -g hexo-cli
 或者：
 npm install -g hexo
```
* 创建本地Hello World
```
 hexo init myblog
 cd myblog
 npm install
```
* myblog下文件如下：
```
_config.yml 是 Hexo 的配置。
node_modules： 是每个 Node.js 项目的依赖代码库，不用管。
package.json： 是每个 Node.js 项目的依赖库说明，不用管。
scaffolds： 模板文件夹，高端话题。非高级用户不用管。
source： 文件夹就是我们放原始 Markdown 写的文章的地方。每次发布上传文章时都要先将这里的原始 Markdown 格式文件编译成 HTML 文件。source 文件夹下默认带了 _post 文件夹，且后者里边有篇默认的 Markdown 文章，即 hello-world.md 文件。
themes： 文件夹里放的是主题，主题就是大家博客的页面布局、样式等。上边安装时带了一个默认主题 landscape，自己可以往这里放自定义主题。
```
* 本地运行 myblog 站点，在 myblog 路径下运行命令：
```
hexo s
```
* 输出如下：
```
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```
* 去浏览器访问 http://localhost:4000/ 看到默认主题 landscape 风格的博客了。博客里默认自带了一篇 Hello World 文章。  此时，我们看到的页面均是本地web，如果你想让别人也看到你的博客，请网下看。

***
### 申请github账号，并将Hexo与github pages联系起来
* github账户的注册和配置  
1.Github注册（如果已经拥有账号，请跳过此步~）  
打开https://github.com/ ，在下图的框中，分别输入自己的用户名，邮箱，密码。  

{% asset_img 搭建博客_sign_up_for_GitHub.jpg  %}

**未完，待更新。。。**
***
### 如何发布文章？