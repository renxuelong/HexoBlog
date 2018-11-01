---
layout: post
title: HelloHexo
date: 2018-10-17 19:28:56
tags:
categories:
---

## Hexo 常用命令

1. hexo new [layout] <title> 新建一篇文章
2. hexo generate/g 生成静态文件 -d/--deploy 文件生成后立即部署网站 -w/--watch 监视文件变动
3. hexo server/s 启动本地服务器
4. hexo deploy 部署之前预先生成的静态文件
5. hexo render <file1> 渲染文件
6. hexo clean 清除缓存文件 db.json 和已生成的静态文件
7. hexo --draft 显示草稿

<!-- more -->

## Hexo 基本操作

### Front-matter 

Front-matter 是文件最上方以 **---** 分隔的区域，用于指定个别文件的变量

```
title: Hello World
date: 2018/08/08 08:08:08
```
- layout 布局
- title 标题
- date 建立日期
- updated 更新日期
- comments 开启文字评论，默认 true
- tags 标签
- categories 分类，Hexo 不支持指定多个同级分类

```
// 下面的分类设置会使 Life 成为 Diary 的子分类
categories:
- Diary
- Life

// 设置标签，可以设置多个标签，标签没有顺序和层次
tags:
- Games
- Sports
```


[官方文档](https://hexo.io/zh-cn/docs)
