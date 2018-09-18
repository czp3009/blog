---
layout: post
title: Hexo插入图片
date: 2018-09-17 16:53:52
category: 前端
tag: hexo
---

本文撰写时的 Hexo 版本 3.7.1

虽然如何在 Hexo 中插入图片是有一篇官方中文教程的, 详见此处 https://hexo.io/zh-cn/docs/asset-folders.html

但是似乎很多人依然看不明白, 所以这里通俗的讲解一下如何在 Hexo 博文中插入图片.

## 外部图片
首先是外部图片, 也就是图片不存储在本地的情况. 这种情况下, 只需要使用通常的 Markdown 语法来引用图片即可, 例如

    ![](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b0/NewTux.svg/150px-NewTux.svg.png)

![](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b0/NewTux.svg/150px-NewTux.svg.png)

## 绝对路径图片
Hexo 在构建之后, `source` 文件夹里面的内容都会被复制到 `public` 文件夹中, 所以图片只要放在 source 文件夹中, 在构建之后, 他们都是存在的.

因此可以在 `source` 目录中创建一个目录专门用来放图片, 例如 `source/images`

然后我们只要把图片放在这么一个目录里面, 就可以在文章中以绝对路径来引用, 如下所示

    ![](/images/someImage.jpg)

## 相对路径图片
如果把图片放在统一的路径下, 很快就会因为命名问题而伤透脑筋. 而且绝大多数图片并不会在多个博文间共用, 这反而导致了图片检索的麻烦.

Hexo 也有每个文章单独的资源目录的设计. 只要在文章所在的目录中创建一个与文章名相同的目录, 这个目录中的资源文件就可以在文章中通过相对目录引用到.

假设 `_posts` 目录是这样的结构

    _posts
    ├── 分类1
    │   └── 文章1.md
    └── 分类2
        ├── 文章2
        │   └── 图片2.png
        └── 文章2.md
    
其中 `文章2` 目录内的资源文件, 就可以在 `文章2.md` 中以相对路径进行调用, 例如

    ![](图片2.png)

但是在这种情况下, 生成出来的图片的 `src` 属性也是一个相对路径, 这会导致文章在首页中无法正常显示图片.

所以我们必须要为文章的摘要和文章本体生成不一样的 `src`.

从 `Hexo 3.x` 版本开始, Hexo 已经集成了 `相对路径引用的标签插件`, 这使得我们可以在 Markdown 中插入对应的标签语句来为首页摘要和文章本体生成不同的图片地址, 语法如下

    {% asset_img slug [title] %}

比如图片名为 `kotlin.png`, 我们可以这么用

    {% asset_img kotlin.png Kotlin forever! %}

{% asset_img kotlin.png Kotlin forever! %}

`[title]` 即图片的 `title` 属性, 在浏览器中, 鼠标悬停在图片上时可以看到.

如果图片中包含空格, 必须要用 `'` 号包裹图片名, 例如

    {% asset_img 'this is kotlin.png' Kotlin forever! %}
