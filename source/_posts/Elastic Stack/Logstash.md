---
title: Logstash
date: {{ date }}
categories:
- Elastic Stack
---

快速开始

```sh
bin/logstash -e 'input { stdin { } } output { stdout {} }'
```

使用配置文件启动

`my.conf`

```
input { 
    stdin {}
}

output { 
    stdout {} 
}
```

```sh
.\bin\logstash.bat -f .\bin\my.conf
```

