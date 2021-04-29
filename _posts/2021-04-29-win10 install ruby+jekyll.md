---
title: Win10安装ruby+jekyll
author: ifuncat
date: 2021-04-29 22:22:22 +0800
categories: [软件安装]
tags: [win10软件,软件安装]
---

## Win10安装ruby+jekyll
<br>

### 1.安装ruby
#### 1.下载ruby+devkit2.7.3-1(x64)
注意不要下载3.x版本, 64位, 下载链接: [https://rubyinstaller.org/downloads/](https://rubyinstaller.org/downloads/)
#### 2.安装.exe文件
注意以下几点
- 建议安装在C盘以外的目录, 不要把路径设置有空格、中文、特殊符号
- 一路选择下一步即可, 安装程序会自动帮助建立ruby的环境变量, 无需自己配置
- 最后一步要勾选 run 'ridk install' to setup MSYS2, 并且安装全部三个组件
- 执行 ruby -v 命令, 验证安装ruby 成功
  
#### 3.更新gem
```bash
##更新gem, 可能有点慢
gem update --system

##替换gem源
gem sources --add https://gems.ruby-china.com/
gem sources --remove https://rubygems.org/

##查看新的gem源
gem source -l
```
### 2.安装jekyll
#### 1.安装前提
按照上述操作, 安装好了ruby, gem
#### 2.使用gem 安装jekyll
```bash
gem install jekyll

##验证安装成功
jekyll -v
```