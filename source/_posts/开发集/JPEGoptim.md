---
title: JPEGoptim
date: {{ date }}
categories:
- 开发集
---

## JPEGoptim

### 1. 安装 `JPEGoptim`

**macOS**

```
brew install jpegoptim
```

**Ubuntu/Debian**

```
sudo apt-get install jpegoptim
```

### 2. 使用 `JPEGoptim`

- **基本压缩**:
  ```bash
  jpegoptim input.jpg
  ```
  默认情况下，这将尝试提供尽可能小的文件大小，同时保持原始质量。

- **指定质量**:
  使用 `-m` 或 `--max` 选项 followed by a quality value (0-100). A lower value will result in a smaller file size but might reduce visual quality.
  ```bash
  jpegoptim -m60 input.jpg
  ```

- **覆盖原文件**:
  默认情况下, `jpegoptim` 会覆盖原始文件。如果你不想这样做，可以使用 `-d` 或 `--dest` 选项指定输出目录。
  ```bash
  jpegoptim -d./optimized/ input.jpg
  ```

- **保留/删除 EXIF 数据**:
  使用 `--strip-all` 选项可以删除所有非必要的元数据 (例如 EXIF)。
  ```bash
  jpegoptim --strip-all input.jpg
  ```

  如果你想保留元数据，使用 `--preserve` 选项。
  ```bash
  jpegoptim --preserve input.jpg
  ```

- **处理多个文件**:
  你可以同时指定多个文件或使用通配符。
  ```bash
  jpegoptim *.jpg
  ```

### 3. 在 Java 中使用 `JPEGoptim`

如果你想在 Java 应用中使用 `JPEGoptim`，你可能需要调用外部进程。这是一个简单的示例：

```java
public class JPEGoptimCompressor {
    public static void compress(String inputFile) throws IOException, InterruptedException {
        String[] command = {
            "jpegoptim",
            "-m80",         // Set quality to 80
            "--strip-all",  // Remove all non-essential metadata
            inputFile
        };
        
        Process process = Runtime.getRuntime().exec(command);
        process.waitFor();
    }
    
    public static void main(String[] args) throws IOException, InterruptedException {
        compress("input.jpg");
    }
}
```

此代码假设 `jpegoptim` 已经安装在系统路径上。

`JPEGoptim` 是一个强大的工具，可以帮助你高效地优化 JPEG 图像。通过合理地调整参数，你可以实现文件大小和图像质量之间的良好平衡。