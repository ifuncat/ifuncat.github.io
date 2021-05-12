---
title: CentOS7 安装docker
author: ifuncat
date: 2021-05-12 22:22:22 +0800
categories: [软件安装]
tags: [软件安装, Docker]
---
#### 1. 准备工作
- 确定CentOS的版本
```shell
cat /etc/redhat-release
```
- 查看内核版本<br/>
docker支持CentOS版本 CentOS 7(64-bit)（系统内核版本为3.10及以上）, CentOS 6.5(64-bit)或更高版本（系统内核版本为 2.6.32-431即以上版本）
```shell
uname -r
```

- 检查 yum 是否可用
```shell
yum list
```
- yum安装gcc相关
```shell
yum -y install gcc
yum -y install gcc-c++
```
- 卸载旧版本(如果有的话)
```shell
yum remove docker \ docker-client \  docker-client-latest \ docker-common \ docker-latest \ docker-latest-logrotate \ docker-logrotate \  docker-selinux \ docker-engine-selinux \ docker-engine
```

#### 2. 开始安装docker
- 安装需要的软件包
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```
- 设置镜像仓库
```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
- 更新yum软件包索引
```shell
yum makecache fast
```
- 安装docker-ce
```shell
yum -y install docker-ce
```

#### 3. 开启docker并验证
- 设置开机启动docker
```shell
systemctl enable docker
```
- 启动/关闭/重启docker服务
```shell
systemctl  start/stop/restart  docker
```
- 验证
```shell
docker version
```

#### 4. 配置阿里云镜像加速
- 先在阿里云申请加速链接: [https://help.aliyun.com/document_detail/60750.html](https://help.aliyun.com/document_detail/60750.html)
- 创建目录及文件(如果不存在)

```shell
mkdir -p  /etc/docker

##进入docker目录,在其中创建配置文件
cd /etc/docker
touch daemon.json
```

- 配置加速链接<br/>
在/etc/docker/daemon.json 中加入:    { "registry-mirrors": ["https://5p4vzvb2.mirror.aliyuncs.com"] }  <br/>
注意: 复制字符串时可能缺少前几个字符串
- 配置生效

```shell
systemctl  daemon-reload
```

- 重启docker并验证

```shell
systemctl restart docker

## docker info最下面的信息registry mirrors为配置阿里云镜像加速的信息
docker info
```