---
title: Win10安装Ruby+jekyll
author: ifuncat
data: 2021-04-29 10:30:00 +0800
categories: [Blogging,软件安装]
tags: [win10软件安装]
math: true
mermaid: true
---

<a name="MBIKq"></a>
### 1. win10 安装ruby环境
<a name="7s0Ax"></a>
#### 1. 下载ruby+devkit2.7.3-1(x64)
注意不要下载3.x版本, 64位, 下载链接: [https://rubyinstaller.org/downloads/](https://rubyinstaller.org/downloads/)
<a name="QmGtU"></a>
#### 2. 安装exe
   注意以下几点:

1. 选择在C盘以外的目录, 千万不要把路径设置有空格、中文、特殊符号
1. 一路选择下一步即可, 自动帮助建立ruby的环境变量, 无需自己配置
1. 最后一步要勾选 run 'ridk install' to setup MSYS2, 并且安装全部三个组件
1. 执行 ruby -v 命令, 验证安装ruby 成功

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2973346/1619534711136-1ef09a16-45dd-458e-8c9b-0fd6f7632895.png#align=left&display=inline&height=462&margin=%5Bobject%20Object%5D&name=image.png&originHeight=497&originWidth=646&size=181052&status=done&style=none&width=600)
<a name="gZw5T"></a>
#### 3. 更新gem
```
##更新gem, 可能有点慢
gem update --system

##替换gem源
gem sources --add https://gems.ruby-china.com/
gem sources --remove https://rubygems.org/

##查看新的gem源
gem source -l
```
<a name="mzKkg"></a>
### 2. 安装jekyll
<a name="kFAqV"></a>
#### 1. 前提
按照上述操作, 安装好了ruby, gem
<a name="v3DVb"></a>
#### 2. gem 安装jekyll
```
gem install jekyll

##验证安装成功
jekyll -v
```


