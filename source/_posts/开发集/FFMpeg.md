---
title: FFMpeg
date: {{ date }}
categories:
- 开发集
---

## FFMpeg

### Windows 下安装

1. 下载页

    https://github.com/BtbN/FFmpeg-Builds/releases

2. 选择安装包

    [ffmpeg-master-latest-win64-gpl-shared.zip](https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-win64-gpl-shared.zip)

3. 将bin目录配置到环境变量Path之中

   ```
   D:\ffmpeg\bin
   ```

### 基本使用

1. 转换视频格式

```shell
ffmpeg -i test.avi test.mp4
```





