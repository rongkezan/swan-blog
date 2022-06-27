---
title: Hexo
date: {{ date }}
categories:
- 工具
- Hexo
---

# Hexo

## Hexo 搭建

### 环境准备

- Node.js
- Git

### 安装Hexo

```sh
npm install -g hexo-cli
```

### 启动本地项目

```sh
# 初始化Hexo项目
hexo init demo
cd demo
npm install
# 启动本地Hexo项目
hexo server
```

### 部署到Github

1. 在Github上创建仓库 demo
2. 安装插件

```sh
npm install hexo-deployer-git --save
```

3. 修改配置文件`_config.yml` 配置远程部署仓库

```yaml
deploy:
  type: git
  repo: https://github.com/rongkezan/demo.git
  branch: gh-pages
```

4. 修改配置文件`_config.yml` 配置根目录

```yaml
url: https://rongkezan.github.io/demo
```

5. 部署

```sh
hexo deploy
```

## Hexo 主题 -- Fluid

1. Fluid官方文档：https://hexo.fluid-dev.com/docs

2. 拉取Fluid主题

```sh
git clone https://github.com/fluid-dev/hexo-theme-fluid.git themes/fluid
```

3. 修改配置文件`_config.yml` 主题配置

```yaml
theme: fluid
```

4. 创建关于页

```sh
hexo new page about
```

5. 覆盖配置

复制官方配置文件到博客目录下： `_config.fluid.yml`

官方配置文件：https://github.com/fluid-dev/hexo-theme-fluid/blob/master/_config.yml
