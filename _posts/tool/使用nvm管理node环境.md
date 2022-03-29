---
layout: post
title: 使用 nvm 管理 node 环境
date: 2022-02-14 23:24:52
permalink: /tool/nvm/guide.html
categories:
- 工具
tags:
- Node
- 开发环境

---

![](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/blog/20220215220905.png)

<!--more-->

## 安装nvm

### 安装

两种方式：

cURL:

```shell
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.39.1/install.sh | bash
```

Wget:

```shell
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.39.1/install.sh | bash
```

可以在nvm的[release页面](https://github.com/creationix/nvm/releases "release页面")查看最新版本，替换链接中的版本号。

### 配置环境变量

把下面代码添加到 `~/.zshrc`

```shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

> 新版本的 nvm 已经可以自动配置环境变量到当前用户的 rc 文件了

记得

```shell
source ~/.zshrc
```

查看nvm版本，验证配置是否成功

```shell
nvm --version
```

## 使用nvm管理node环境

### 安装node

查看远程node版本

```shell
nvm ls-remote
```

安装node

```shell
nvm install v10.15.0
```

想安装什么版本就安装什么版本，想安装几个就安装几个。

### 切换node版本

查看本地node版本列表

```shell
nvm ls
```

切换node版本

```shell
nvm use v10.15.0
```

### 设置默认node版本

如果安装了多个版本的nodeJs可能需要指定default版本.这样每次新打开一个终端都会使用default版本的node

```shell
nvm alias default v10.15.0
```

## 使用nrm快速切换npm源

有时候需要切换 NPM 镜像。相比每次切换时都手动指定相应参数，使用[nrm](https://github.com/Pana/nrm)要方便的多。

nrm 是一个 NPM 源管理器，允许你快速地在如下 NPM 源间切换：

- [npm](https://www.npmjs.org/)
- [cnpm](http://cnpmjs.org/)
- [strongloop](http://strongloop.com/)
- [european](http://npmjs.eu/)
- [australia](http://npmjs.org.au/)
- [nodejitsu](https://www.nodejitsu.com/)
- [taobao](http://npm.taobao.org/)

### 安装

```shell
npm install -g nrm
```

列出可选的源

```shell
nrm ls
```

![](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-01-14%20%E4%B8%8B%E5%8D%883.02.21.png)

带 `*` 的是当前使用的源，上面的输出表明当前源是cnpm源。

切换到taobao

```shell
nrm use taobao
```

增加源

```shell
nrm add  <registry> <url> [home]
```

删除源

```shell
nrm del <registry>
```

测试速度

```shell
nrm test npm  
```

测试所有源响应时间

```shell
nrm test
```
