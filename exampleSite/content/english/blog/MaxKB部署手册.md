---
title: "MaxKB 安装部署指导手册"
meta_title: "MaxKB 安装部署指导手册"
description: "MaxKB 安装部署指导手册"
date: 2025-04-01T08:00:00Z
categories: ["智算应用"]
tags: ["部署", "MaxKB"]
draft: false
---

# MaxKB 安装部署指导手册

## 部署前准备

### 部署服务器要求：

- 操作系统：Ubuntu 22.04 / CentOS 7.6 64 位系统
- CPU/内存：4C/8GB 以上
- 磁盘空间：100GB
  <!--more-->


### 端口要求

离线部署 MaxKB 需要开通的访问端口说明如下：

| 端口 | 作用         | 说明                                          |
| ---- | ------------ | --------------------------------------------- |
| 22   | SSH          | 安装、升级及管理使用                          |
| 8080 | Web 服务端口 | 默认 Web 服务访问端口，可根据实际情况进行更改 |



### 安装包下载

打开 [飞致云开源社区 MaxKB 社区版下载](https://community.fit2cloud.com/#/products/maxkb/downloads) 页面下载最新版本安装包，并上传至部署服务器。

登录maxkb官网进行安装包下载，此处建议下载**v1.10.2-lts**版本

**v1.10.2-lts**版本下载链接：https://community.fit2cloud.com/#/products/maxkb/downloads

![image-20250329163512416](C:\Users\fan1k\AppData\Roaming\Typora\typora-user-images\image-20250329163512416.png)



## 安装部署

### 解压安装包

以 root 用户通过 ssh 协议登录到部署服务器, 对安装包进行解压：

```
tar -zxvf maxkb-v1.10.2-lts-offline.tar.gz
```



### 安装前配置

MaxKB 安装目录、服务运行端口、数据库配置等信息可在安装包解压后中的 install.conf 文件进行配置。

```
## 安装目录
MAXKB_BASE=/opt
## Service 端口
MAXKB_PORT=8080
## docker 网段设置
MAXKB_DOCKER_SUBNET=172.19.0.0/16
# 数据库配置
## 是否使用外部数据库
MAXKB_EXTERNAL_PGSQL=false
## 数据库地址
MAXKB_PGSQL_HOST=pgsql
## 数据库端口
MAXKB_PGSQL_PORT=5432
## 数据库库名
MAXKB_PGSQL_DB=maxkb
## 数据库用户名
MAXKB_PGSQL_USER=root
## 数据库密码
MAXKB_PGSQL_PASSWORD=Password123@postgres
```

**注意**：首次安装之前可在 install.conf文件中的修改参数，安装时则根据修改后的参数执行安装。完成安装后如需再次修改配置参数，则需要在 ${MAXKB_BASE}/maxkb/.env（默认是 /opt/maxkb/.env）文件中进行修改，并且在修改完后需执行 `mkctl reload` 命令重新加载配置文件。



### 执行安装脚本

```
# 进入安装包解压缩后目录  
cd maxkb-v1.10.2-lts-offline/

执行安装命令
bash install.sh
```

![安装](https://maxkb.cn/docs/img/index/install.jpg)

### 登录访问

待所有容器状态显示为`healthy`后，即可通过浏览器访问地址 `http://目标服务器 IP 地址:8080`，并使用默认的管理员用户和密码登录 MaxKB。

```bash
用户名：admin
默认密码：MaxKB@123..
```

![登录](https://maxkb.cn/docs/img/index/login.jpg)

```bash
docker run -itd --privileged  --name=ds-v3-w8a8 --net=host \
   --shm-size 500g \
   --device=/dev/davinci0 \
   --device=/dev/davinci1 \
   --device=/dev/davinci2 \
   --device=/dev/davinci3 \
   --device=/dev/davinci4 \
   --device=/dev/davinci5 \
   --device=/dev/davinci6 \
   --device=/dev/davinci7 \
   --device=/dev/davinci_manager \
   --device=/dev/hisi_hdc \
   --device /dev/devmm_svm \
   -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
   -v /usr/local/Ascend/firmware:/usr/local/Ascend/firmware \
   -v /usr/local/sbin/npu-smi:/usr/local/sbin/npu-smi \
   -v /usr/local/sbin:/usr/local/sbin \
   -v /etc/hccn.conf:/etc/hccn.conf \
   -v /root/ds-v3/ds-v3-w8a8:/root/ds-v3/ds-v3-w8a8 \  ## 修改为权重路径
   swr.cn-south-1.myhuaweicloud.com/ascendhub/mindie:2.0.T3.1-800I-A2-py311-openeuler24.03-lts \ # 镜像名
   bash
   
   # 进入容器
   docker exec -it ds-v3-w8a8 bash
```



## 离线升级（按需可选）

离线升级与安装操作过程基本一样，即下载新版本安装包上传解压后，再次执行安装命令进行升级。

```
# 进入新版本目录
cd maxkb-v1.x.y-offline

# 运行安装脚本
bash install.sh

# 查看 MaxKB 运行状态
mkctl status
```

**注意：** 升级前请先对数据库进行备份。
