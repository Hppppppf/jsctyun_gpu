---
title: "Dify 安装部署指导手册"
meta_title: "Dify 安装部署指导手册"
description: "Dify 安装部署指导手册"
date: 2025-04-01T08:00:00Z
categories: ["部署手册","智算应用"]
tags: ["部署", "Dify"]
draft: false
---
# Dify 安装部署指导手册

Dify 是一个让你轻松构建 AI 应用的开源平台，简单来说就是给开发者打造的一站式AI应用开发工具。
<!--more-->
它主要有以下几个特点：

- 简单直观：你可以通过图形界面来创建和调试 AI 应用，几分钟就能发布。
- 上下文集成：可以用你自己的数据来进行文本预处理，无需深入了解技术细节。
- API 访问：提供后台服务，直接访问 Web 应用或将 API 集成到你的项目中，不用担心复杂的后台架构和部署问题。
- 数据优化：通过图形界面查看 AI 的运行日志，改进数据标注，不断提升 AI 的表现。

Dify 兼容 Langchain，支持多种大语言模型（LLM）。目前支持的模型供应商包括 OpenAI、Azure OpenAI、Anthropic、Replicate、Hugging Face、ChatGLM、Llama2、MiniMax、讯飞星火、文心一言和通义千问等。

换句话说，Dify 就是一个帮你快速、高效地开发和优化 AI 应用的万能工具箱。

此文分享使用docker方式部署dify。

## 部署前准备

### 硬件要求

在安装Dify之前，请确保您的系统满足以下最低要求：

- **CPU**: 至少2核心（推荐4核心或更高）
- **内存**: 至少4GB RAM（推荐8GB或更高）
- **存储空间**: 至少20GB可用空间
- **网络**: 稳定的互联网连接，用于拉取Docker镜像



### 软件要求

- **Docker**: 版本19.03或更高
- **Docker Compose**: 版本1.28或更高（或Docker Compose V2）

确保你已经安装了 Docker 和 Docker Compose。如果没有，请先安装：

Docker：可以从 Docker 官方网站(https://www.docker.com/get-started) 下载并安装适合你操作系统的版本。
Docker Compose：一般 Docker 会自带 Docker Compose，如果没有，你可以按照 Docker Compose 安装指南(https://docs.docker.com/compose/install/) 进行安装。



### 安装包下载

可以直接下载源码https://github.com/langgenius/dify/releases/tag/1.1.3

![img](https://i-blog.csdnimg.cn/img_convert/4adf65764f3b65272e47f153104d8c19.png)



## 安装部署

### 解压安装包

以 root 用户通过 ssh 协议登录到部署服务器, 对安装包进行解压：

```
tar -zxvf dify-1.1.3.tar.gz
```



### 安装前配置

（1）进入源码的docker文件夹

```
cd dify/docker
```

（2）复制环境配置文件

```
cp .env.example .env
```

（3）docker镜像源

由于墙的存在，所以默认的docker镜像源很难拉取项目，需要调整相关的docker配置文件

```
vim /etc/docker/daemon.json
```

**添加如下docker镜像源**

```
{
"registry-mirrors":[
    "https://9cpn8tt6.mirror.aliyuncs.com",
    "https://registry.docker-cn.com",
    "https://mirror.ccs.tencentyun.com",
    "https://docker.1panel.live",
    "https://2a6bf1988cb6428c877f723ec7530dbc.mirror.swr.myhuaweicloud.com",
    "https://docker.m.daocloud.io",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com",
    "https://your_preferred_mirror",
    "https://dockerhub.icu",
    "https://docker.registry.cyou",
    "https://docker-cf.registry.cyou",
    "https://dockercf.jsdelivr.fyi",
    "https://docker.jsdelivr.fyi",
    "https://dockertest.jsdelivr.fyi",
    "https://mirror.aliyuncs.com",
    "https://dockerproxy.com",
    "https://mirror.baidubce.com",
    "https://docker.m.daocloud.io",
    "https://docker.nju.edu.cn",
    "https://docker.mirrors.sjtug.sjtu.edu.cn",
    "https://docker.mirrors.ustc.edu.cn",
    "https://mirror.iscas.ac.cn",
    "https://docker.rainbond.cc"
    ]
}
```

调整完后如下：

```
{
    "registry-mirrors":[
                    "https://9cpn8tt6.mirror.aliyuncs.com",
                    "https://registry.docker-cn.com",
                    "https://mirror.ccs.tencentyun.com",
                    "https://docker.1panel.live",
                    "https://2a6bf1988cb6428c877f723ec7530dbc.mirror.swr.myhuaweicloud.com",
                    "https://docker.m.daocloud.io",
                    "https://hub-mirror.c.163.com",
                    "https://mirror.baidubce.com",
                    "https://your_preferred_mirror",
                    "https://dockerhub.icu",
                    "https://docker.registry.cyou",
                    "https://docker-cf.registry.cyou",
                    "https://dockercf.jsdelivr.fyi",
                    "https://docker.jsdelivr.fyi",
                    "https://dockertest.jsdelivr.fyi",
                    "https://mirror.aliyuncs.com",
                    "https://dockerproxy.com",
                    "https://mirror.baidubce.com",
                    "https://docker.m.daocloud.io",
                    "https://docker.nju.edu.cn",
                    "https://docker.mirrors.sjtug.sjtu.edu.cn",
                    "https://docker.mirrors.ustc.edu.cn",
                    "https://mirror.iscas.ac.cn",
                    "https://docker.rainbond.cc"
    ],
    "data-root":"/home/data_c/docker_data",
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}


```



### 安装部署

启动Docker容器 根据系统上的Docker Compose版本选择适当的命令来启动容器。

- 如果你有 Docker Compose V2, 使用下面的命令:

```plain
docker compose up -d
```

- 如果你有 Docker Compose V1, 使用下面的命令:

```plain
docker-compose up -d
```

执行命令后，你应该看到类似下面的输出，显示所有容器的状态和端口映射

```
[+] Running 11/11
 ✔ Network docker_ssrf_proxy_network  Created                                                                 0.1s 
 ✔ Network docker_default             Created                                                                 0.0s 
 ✔ Container docker-redis-1           Started                                                                 2.4s 
 ✔ Container docker-ssrf_proxy-1      Started                                                                 2.8s 
 ✔ Container docker-sandbox-1         Started                                                                 2.7s 
 ✔ Container docker-web-1             Started                                                                 2.7s 
 ✔ Container docker-weaviate-1        Started                                                                 2.4s 
 ✔ Container docker-db-1              Started                                                                 2.7s 
 ✔ Container docker-api-1             Started                                                                 6.5s 
 ✔ Container docker-worker-1          Started                                                                 6.4s 
 ✔ Container docker-nginx-1           Started                                                                 7.1s
```

最后检查所有容器是否启动成功

```
docker compose ps 
```

这包括3个核心服务：api / worker / web，以及6个依赖组件：weaviate / db / redis / nginx / ssrf_proxy / sandbox。

```
NAME                  IMAGE                              COMMAND                   SERVICE      CREATED              STATUS                        PORTS
docker-api-1          langgenius/dify-api:0.6.13         "/bin/bash /entrypoi…"   api          About a minute ago   Up About a minute             5001/tcp
docker-db-1           postgres:15-alpine                 "docker-entrypoint.s…"   db           About a minute ago   Up About a minute (healthy)   5432/tcp
docker-nginx-1        nginx:latest                       "sh -c 'cp /docker-e…"   nginx        About a minute ago   Up About a minute             0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp
docker-redis-1        redis:6-alpine                     "docker-entrypoint.s…"   redis        About a minute ago   Up About a minute (healthy)   6379/tcp
docker-sandbox-1      langgenius/dify-sandbox:0.2.1      "/main"                   sandbox      About a minute ago   Up About a minute             
docker-ssrf_proxy-1   ubuntu/squid:latest                "sh -c 'cp /docker-e…"   ssrf_proxy   About a minute ago   Up About a minute             3128/tcp
docker-weaviate-1     semitechnologies/weaviate:1.19.0   "/bin/weaviate --hos…"   weaviate     About a minute ago   Up About a minute             
docker-web-1          langgenius/dify-web:0.6.13         "/bin/sh ./entrypoin…"   web          About a minute ago   Up About a minute             3000/tcp
docker-worker-1       langgenius/dify-api:0.6.13         "/bin/bash /entrypoi…"   worker       About a minute ago   Up About a minute             5001/tcp
```

通过这些步骤，您应该能够成功安装Dify。



### 4、登录访问

服务启动成功后，通过浏览器访问：

```text
# 如果使用默认80端口
http://localhost/install

# 如果修改了端口，例如改为8080
http://localhost:8080/install
```

对于远程服务器，将`localhost`替换为服务器IP地址。

首次访问时，系统会引导您创建管理员账户：

1. 填写管理员姓名、电子邮件和密码
2. 点击"创建"按钮完成账户创建
3. 使用刚创建的账户登录系统



### 配置模型供应商

登录后，需要配置至少一个模型供应商才能使用Dify的核心功能：

- 配置OpenAI（最常用）

1. 点击右上角用户头像，选择"设置"

2. 在左侧导航栏选择"模型供应商"

3. 点击"OpenAI"卡片

4. 点击"添加"按钮

5. 输入您的OpenAI API密钥

6. 选择需要使用的模型（如gpt-3.5-turbo、gpt-4等）

7. 点击"保存"

   

- 配置本地部署模型

Dify也支持连接本地部署的开源模型：

1. 在"模型供应商"页面选择合适的供应商（如Ollama、Xinference等）
2. 配置API端点（如`http://host.docker.internal:11434`）
3. 选择可用模型
4. 测试连接并保存

> **提示**：如果连接本地模型，请使用`host.docker.internal`替代`localhost`或`127.0.0.1`，以确保Docker容器能正确访问宿主机服务。



## 常用知识

### 各服务功能详解

| 服务名称   | 功能描述                               | 默认镜像                         |
| ---------- | -------------------------------------- | -------------------------------- |
| api        | 核心后端API服务，处理请求和业务逻辑    | langgenius/dify-api:0.15.3       |
| worker     | 异步任务处理服务，如文档处理、向量化等 | langgenius/dify-api:0.15.3       |
| web        | 提供Web用户界面                        | langgenius/dify-web:0.15.3       |
| nginx      | 反向代理，处理请求路由和负载均衡       | nginx:latest                     |
| db         | PostgreSQL数据库，存储结构化数据       | postgres:15-alpine               |
| redis      | Redis缓存和消息队列                    | redis:6-alpine                   |
| weaviate   | 向量数据库，支持语义搜索               | semitechnologies/weaviate:1.19.0 |
| sandbox    | 安全沙箱环境，用于插件执行             | langgenius/dify-sandbox:0.2.1    |
| ssrf_proxy | 防SSRF代理，增强安全性                 | ubuntu/squid:latest              |

### 常见错误与解决方法

| 问题描述        | 可能原因                      | 解决方案                                     |
| --------------- | ----------------------------- | -------------------------------------------- |
| 80端口被占用    | 系统中已有服务使用该端口      | 修改.env中的EXPOSE_NGINX_PORT为其他可用端口  |
| 容器无法启动    | 资源不足或配置错误            | 检查系统资源和Docker日志，调整Docker资源限制 |
| 数据库连接失败  | 数据库配置错误或服务未启动    | 检查数据库容器状态和连接配置                 |
| 模型连接失败    | API密钥错误或网络问题         | 验证API密钥，检查网络连接和代理设置          |
| 文档上传失败    | 文件格式不支持或存储配置错误  | 检查文件格式，验证存储配置                   |
| 向量检索不工作  | Weaviate服务问题或配置错误    | 检查Weaviate容器状态，验证向量库配置         |
| 访问页面404错误 | Nginx配置问题或服务未正确启动 | 检查Nginx配置和容器日志                      |
