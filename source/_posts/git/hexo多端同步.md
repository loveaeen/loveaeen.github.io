---
title: hexo多端同步
date: 2022-03-26 19:33:33
categories: git
tags:
excerpt: 这是一篇关于Hexo博客框架如何在git中多端同步的学习记录
---

## 摘要

这是一篇关于Hexo博客框架如何在git中多端同步的学习记录

## 步骤

```bash
git init  //初始化本地仓库
git remote add origin git@github.com:yourname/yourname.github.io.git  //将本地与Github项目对接
git checkout -b hexo  //新建并切换到hexo分支上
git add . //将必要的文件依次添加
git commit -m "Blog Source Hexo"
git push origin hexo  //push到Github项目的hexo分支上
```

## 问题

学习时在最终的 `git push origin hero`命令中遇到了 <font color='red'>Permission denied</font> 相关问题，发现需要生成新的ssh key，来添加给git。

```bash
ssh-keygen -t rsa -C 'github注册时的邮箱'
# 一路回车，最后会在～/.ssh目录中生成密钥文件
# 将id_rsa.pub公钥添加到git中
# 再执行 git push origin hexo 即可
```

