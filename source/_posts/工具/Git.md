---
title: Git
date: {{ date }}
categories:
- 工具
- Git
---
## Git 仓库操作

命令帮助

```shell
git remote --help
```

查看远程仓库

```shell
git remote -v
```

添加远程仓库

```shell
git remote add origin https://gitee.com/KeithRong/demo.git
```

删除远程仓库

```shell
git remote remove origin
```

## Git 推送操作

命令帮助

```shell
git fetch --help
git add --help
git commit --help
git push --help
```

将本地分支推送关联远程分支

```shell
git push -u origin master
```

## Git 分支操作

命令帮助

```shell
git checkout --help
git merge --help
git rebase --help
```

创建新分支

```shell
git checkout -b [branch_name]
```

删除远程分支

```shell
git push origin --delete [branch_name]
```



