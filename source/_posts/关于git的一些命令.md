---
title: 关于git的一些命令
date: 2019-12-22 11:04:02
tags:
---
# **Git和Github的一些基础命令** 
<!-- more -->
## Git基础命令
* 初始化git仓库(会出现master分支)

    `git init`
* 查看git库的状态（有两种状态：1.已经在缓冲区的（绿色）2.未添加到缓冲区的(红色)）

    `git status`
* 提交缓冲区的所有修改到本地仓库(要加上描述方便以后回到以前的版本)

    `git commit -m "提交的说明"`
* 查看日志

    `git log`
* 版本回退：可以将当前仓库回退到历史的某个版本 

    `git reset` 
* 查看仓库的操作历史

    `git reflog`

* 删除本地仓库

    `rm -rf  .git`

## 分支命令
* 查看分支的情况，前面带*号的就是当前分支

    `git branch`
* 创建分支

    `git branch 分支名`
* 切换当前分支到指定分支

    `git checkout 分支名`
* 合并某分支的内容到当前分支

    `git merge 分支名`
* 删除分支
    
    `git branch -d 分支名`

## Git远端库相关

* 增加远程仓库,名叫origin

    `git remote add + 名字(origin) + 远程库地址`

* 移除远程仓库origin

    `git remote remove origin移除远端仓库`

* 将本地仓库master分支推送到origin远端仓库上 

    `git push -u origin master`

* 从远端库更新内容到本地

    `git pull`

## 关于SSH和登录
* 查看是否配置ssh公钥

    `cd ~/.ssh`
* 生成ssh公钥

    先执行`ssh-keygen -t rsa -C "邮箱"`，然后三下回车不然以后推送要密码。找到`cat ~/.ssh/id_rsa.pub`，他会生成两个公钥，一个私密的，一个公共的，我们找到公共的然后上到github上新建公钥然后将找到的公共的公钥用记事本打开，复制到新建的公钥上，然后就创建成功了。

    测试：输入`ssh git@github.com` 如果输出 `Hi xxx! You've successfully authenticated, but GitHub does not # provide shell access. Connection to github.com closed.` 说明成功了

* 配置名字和邮箱 (全局加上global)

    `git config --global user.name "bryan sun"` 

    `git config --global user.email "hitsjt@gmail.com"`
    