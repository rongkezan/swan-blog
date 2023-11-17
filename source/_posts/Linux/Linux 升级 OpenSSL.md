---
title: Linux 升级 OpenSSL
date: {{ date }}
categories:
- Linux
---

在CentOS 7上升级OpenSSL至新版本（如1.1.1以支持TLS 1.3）通常涉及到从源代码编译安装，因为CentOS 7的默认软件仓库可能不包含最新版本的OpenSSL。以下是从源代码编译安装OpenSSL的基本步骤：

### 备份旧版本的OpenSSL

在开始之前，建议备份现有的OpenSSL安装，以防升级过程中出现任何问题。

### 安装必要的工具和依赖项

1. 安装编译工具和开发库：
   ```bash
   sudo yum install -y make gcc perl pcre-devel zlib-devel
   ```

### 从源代码编译OpenSSL

1. **下载OpenSSL源代码**：
   - 访问 [OpenSSL官方网站](https://www.openssl.org/source/) 或 [OpenSSL GitHub仓库](https://github.com/openssl/openssl) 来获取最新的源代码压缩包。
   - 使用`wget`下载：
     ```bash
     wget https://www.openssl.org/source/openssl-1.1.1k.tar.gz
     ```
   - 确保替换URL中的版本号为您想要安装的最新版本。

2. **解压源代码**：
   ```bash
   tar -zxf openssl-1.1.1k.tar.gz
   cd openssl-1.1.1k
   ```

3. **配置安装**：
   - 配置安装路径（在这里使用`/usr/local/openssl`）和共享库：
     ```bash
     ./config --prefix=/usr/local/openssl --openssldir=/usr/local/openssl shared zlib
     ```

4. **编译和安装**：
   ```bash
   make
   sudo make install
   ```

### 更新系统配置

1. **配置动态链接器**：
   - 添加新安装的OpenSSL库到动态链接器配置，并运行`ldconfig`来更新缓存：
     ```bash
     echo "/usr/local/openssl/lib" | sudo tee -a /etc/ld.so.conf.d/openssl-1.1.1k.conf
     sudo ldconfig
     ```

2. **更新系统OpenSSL链接**：
   - 创建软链接以确保系统使用新版本的OpenSSL（慎重操作，这可能会影响系统其他部分对OpenSSL的依赖）：
     ```bash
     sudo ln -sf /usr/local/openssl/bin/openssl /usr/bin/openssl
     ```

3. **验证新版本**：
   - 验证安装的OpenSSL版本：
     ```bash
     openssl version
     ```

### 注意事项

- 升级系统核心组件（如OpenSSL）可能会影响到依赖这些组件的其他软件。务必在升级前备份重要数据，并准备好回滚计划。
- 在生产环境中，建议在升级前在一个测试环境中验证所有更改。
- 如果您在升级过程中遇到问题，可能需要咨询更详细的系统管理指南或专业的技术支持。

这个过程可能会因为具体的系统配置和已安装软件的不同而有所变化，因此请根据实际情况进行调整。