---
title: Linux 网桥
date: {{ date }}
categories:
- Linux
---

安装网桥管理工具包 `bridge-utils`

```sh
yum install bridge-utils -y
```

```sh
# 查询网桥
brctl show

# 创建网桥br1
brctl addbr br1

# 删除网桥br1
brctl delbr br1

# 将eth0端口加入网桥br1 
brctl addif br1 eth0

# 删除eth0端口加入网桥br1 
brctl delif br1 eth0
```

