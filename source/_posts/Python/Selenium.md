---
title: Selenium
date: {{ date }}
categories:
- Python
---

### Selenium控制已打开的Chrome (mac)

1. 下载Webdriver放到`/user/local/bin`目录下

2. 添加环境变量

```shell
export PATH="/Applications/Google Chrome.app/Contents/MacOS:$PATH"
```

3. 命令打开Chrome

```sh
Google\ Chrome --remote-debugging-port=9222 --user-data-dir="ChromeProfile"
```

4. 执行代码

```python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_experimental_option("debuggerAddress", "127.0.0.1:9222")
driver = webdriver.Chrome(options=options)
driver.get("http://www.baidu.com")
```