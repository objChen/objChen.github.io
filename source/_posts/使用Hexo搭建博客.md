---
title: 使用Hexo搭建博客
date: 2015-03-19 21:20:50
tags:
---

# Hexo简介

Hexo是一个基于Node.js的静态博客软件，可用Linux/Windows/Mac部署，也可直接托管在Github上，简单易学。
用hexo写博客的步骤是：

1. 在本地用markdown语法写一篇文章；
2. 在本地生成静态网页，并启动本地server调试
3. 调试完成后，由于Github Pages可以解析静态页面，因此直接上传到gitpages就可以。

# 安装Hexo

下面的过程基于Debian 4.13.4-2，其他系统的安装类似，也可以从[官方文档](https://hexo.io/docs/index.html)找到。

安装Node.js:

安装Hexo:

> npm install -g hexo-cli

# 初始化Hexo

1. 本地新建一个目录如blog
2. 进入该目录，执行hexo init
3. 运行npm install
4. 最后运行hexo generate && hexo server (或者简写为hexo g && hexo s)，会在http://localhost:4000启动一个本地服务器供调试。
5. 如果http://localhost:4000可以正常访问显示“Hello World”，说明搭建成功啦！

# 部署到Github Pages

首先需要确保拥有一个github账号，并按照Github Pages文档创建一个repository。
该repository的名称有严格规定：<username>.github.io
如我的用户名是objchen，对应的repository名称就是objchen.github.io。

为了将来免密码登录，需要把本机的公钥(~/.ssh/id_rsa.pub)复制到github上，然后用如下命令测试：

> $ ssh -T git@github.com

输入yes，如果看到”You’ve successfully authenticated”字样，就说明ssh公钥上传成功了。

接下来修改blog目录中的_config.yml配置文件，找到deploy字段，修改为(用户名改为自己的)：

```
deploy:
  type: git
  repository: git@github.com:<username>/<username>.github.io.git
  branch: master
```

然后执行下述命令生成并推送到github:

> $ npm install hexo-deployer-git -save && hexo g && hexo d

