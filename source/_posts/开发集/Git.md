---
title: Git
date: {{ date }}
categories:
- 工具
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

## Linux记住Git账号密码

配置Git用户名

```sh
git config --global user.name "your username"
```

配置Git用户邮箱

```sh
git config --global user.email "your email"
```

配置Git记住密码

```sh
git config --global credential.helper store
```

拉取Git仓库后会让你输入一次账号密码，之后就不用再输入了。

刚才的配置信息保存在 root 目录下的 `.gitconfig` `.git-credential`

