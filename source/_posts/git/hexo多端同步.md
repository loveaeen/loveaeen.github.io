---
title: hexo多端同步
date: 2022-03-26 19:33:33
categories: git
tags: hexo
excerpt: 这是一篇关于Hexo博客框架如何在git中多端同步的学习记录
---

## 摘要

这是一篇关于Hexo博客框架如何在git中多端同步的学习记录

## 步骤

### 文件同步

```bash
git init  //初始化本地仓库
git remote add origin git@github.com:yourname/yourname.github.io.git  //将本地与Github项目对接
git checkout -b hexo  //新建并切换到hexo分支上
git add . //将必要的文件依次添加
git commit -m "Blog Source Hexo"
git push origin hexo  //push到Github项目的hexo分支上
```

### 图片同步

当我想要在博客与Typora软件中都想要看到文章内引用的本地图片时，发生了渲染错误。博客的服务器端不支持Typora的本地路径引用方式。

为此我查看了一下Fluid的文档，发现官方默认指定了图片存放路径 `public/img` 下。

既然官方限制了，那我便只好更改Typora插入图片时的相关规则

1. Typora -> 偏好设置 -> 图像 -> 插入图片时 (复制到指定路径) `~/hexo/blog/public/img/${filename}` 并勾选下方的所有子项
2. 格式 -> 图像 -> 设置图片根目录 -> 选中 `~/hexo/blog/public/` 路径，这一步是保证了博客与Typora都能预览到图片内容

## 问题

学习时在最终的 `git push origin hero`命令中遇到了 <font color='red'>Permission denied</font> 相关问题，发现需要生成新的ssh key，来添加给git。

```bash
ssh-keygen -t rsa -C 'github注册时的邮箱'
# 一路回车，最后会在～/.ssh目录中生成密钥文件
# 将id_rsa.pub公钥添加到git中
# 再执行 git push origin hexo 即可
```

