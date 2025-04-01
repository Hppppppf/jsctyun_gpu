---
title: "Open WebUI 安装部署指导手册"
meta_title: "Open WebUI 安装部署指导手册"
description: "Open WebUI 安装部署指导手册"
date: 2025-04-01T08:00:00Z
categories: ["部署手册","智算应用"]
tags: ["部署", "Open WebUI"]
draft: false
---

# Open WebUI安装部署指导手册
Open WebUI 是一个可扩展、功能丰富且用户友好的自托管 AI 平台，旨在完全离线运行。它支持各种 LLM 运行器，如 Ollama 和 OpenAI 兼容的 API，并内置了 RAG 推理引擎，使其成为强大的 AI 部署解决方案。

1、框架开源：Open WebUI 是开源的，开发者可以自由使用它，适用于各种Web 应用开发；
2、用户界面构建：它提供了丰富的 UI 组件和工具，帮助开发者快速构建响应式、易于使用的界面；
3、跨平台兼容：支持主流浏览器和操作系统，确保开发的 Web 应用在不同平台上都能流畅运行；
4、灵活性与扩展性：Open WebUI 具有高度的灵活性，允许与其他框架和工具集成，支持复杂的交互设计和功能扩展；
5、开发者友好：提供简洁的 API 和文档，使得开发者能够轻松上手，快速构建功能齐全的 Web 界面；
6、Open WebUI 适用于需要创建直观、交互性强的 Web 界面的项目，能够提升开发效率并简化用户界面的设计。

此文分享使用docker方式部署Open WebUI。

## 部署前准备

### 硬件要求

在安装Open WebUI之前，请确保您的系统满足以下最低要求：

| 硬件类型 | 最低配置 | 推荐配置         | 核心作用                       |
| -------- | -------- | ---------------- | ------------------------------ |
| **CPU**  | 4核      | 8核及以上        | 支持基础模型推理与并发请求     |
| **内存** | 4GB      | 16GB+            | 保障大模型加载与多任务流畅运行 |
| **磁盘** | 20GB     | 100GB+ SSD       | 存储模型文件、日志及缓存数据   |
| **GPU**  | 非必需   | NVIDIA RTX 3090+ | 加速模型推理（需CUDA支持）     |



### 软件要求

- **Docker**: 版本19.03或更高
- **Docker Compose**: 版本1.28或更高（或Docker Compose V2）

确保你已经安装了 Docker 和 Docker Compose。如果没有，请先安装：

Docker：可以从 Docker 官方网站(https://www.docker.com/get-started) 下载并安装适合你操作系统的版本。
Docker Compose：一般 Docker 会自带 Docker Compose，如果没有，你可以按照 Docker Compose 安装指南(https://docs.docker.com/compose/install/) 进行安装。



## 安装部署

### 拉取docker镜像

原始镜像下载很慢，可使用代理方式拉取镜像：

- 代理拉取镜像

```plain
docker pull ghcr.dockerproxy.net/open-webui/open-webui:latest
```

- 重命名镜像

```
docker tag ghcr.dockerproxy.net/open-webui/open-webui:latest ghcr.io/open-webui/open-webui:latest
```

- 删除代理镜像

```
docker rmi ghcr.dockerproxy.net/open-webui/open-webui:latest
```



### 安装部署

### 使用默认配置安装

- **如果 Ollama 在您的计算机上**，请使用以下命令：

```
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:latest
```



- **如果 Ollama 位于其他服务器上**，请使用以下命令：
  要连接到另一台服务器上的 Ollama，请将更改为服务器的 URL：OLLAMA_BASE_URL​

```
docker run -d -p 3000:8080 -e OLLAMA_BASE_URL=https://example.com -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:latest
```



### 安装仅用于 OpenAI API

- **如果您只使用 OpenAI API**，请使用以下命令：

```
docker run -d -p 3000:8080 -e OPENAI_API_KEY=your_secret_key -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:latest
```



这个命令都有助于 Open WebUI 和 Ollama 的内置轻松安装，确保您可以快速启动并运行所有内容。

安装后，您可以访问 Open WebUI。  



### 登录访问

服务启动成功后，通过浏览器访问：

```text
# 拉起容器时使用的3000端口
http://localhost:3000
```

对于远程服务器，将`localhost`替换为服务器IP地址。

首次访问时，系统会引导您创建管理员账户：

1. 填写管理员姓名、电子邮件和密码
2. 点击"创建"按钮完成账户创建
3. 使用刚创建的账户登录系统
