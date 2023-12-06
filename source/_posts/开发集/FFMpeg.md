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

2. 将视频文件转换为480P

```sh
# -c:v libx264：指定视频编码器为 libx264（适用于 MP4 文件）。
# -c:a copy：复制原始音频流，不进行音频转码。
ffmpeg -i input.mkv -s 720x480 -c:v libx264 -c:a copy output.mp4
```

3. 剪切视频

```sh
ffmpeg -i input.mp4 -ss 00:10:00 -to 00:15:00 -c copy output.mp4
```



