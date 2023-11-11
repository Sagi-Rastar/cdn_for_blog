---
title: 【建站过程】hexo-fluid主题添加图集瀑布流功能
date: 2023-11-11 16:30:59
tags:
  - 过程记录
  - 个人博客
  - 网站
  - 建站
---
# 00-前言

根据上一篇博客[【建站过程】关于我使用obsidian加hexo部署个人博客的过程](https://sagi-rastar.github.io/2023/11/10/%E3%80%90%E5%BB%BA%E7%AB%99%E8%BF%87%E7%A8%8B%E3%80%91%E5%85%B3%E4%BA%8E%E6%88%91%E4%BD%BF%E7%94%A8obsidian%E5%8A%A0hexo%E9%83%A8%E7%BD%B2%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2%E7%9A%84%E8%BF%87%E7%A8%8B/) ，目前我的个人站点已经有了一个比较舒服的以文字为主的工作流，接下来就是分享图片的工作流了。

# 01-搭建方案

通过查阅和参考其他大佬们的博客，发现hexo5有的一个功能 `注入代码` 是一个不错的方案。本次初步实现图集瀑布流的功能主要参考了[@四维树大佬的这篇博客](https://4dtree.github.io/2022/06/21/Hexo-Fluid%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E7%9B%B8%E5%86%8C%E5%8A%9F%E8%83%BD/) 受益匪浅，感谢！

其中，对于 `scripts` 文件夹，如果之前没有使用过hexo注入代码功能的朋友，可能会感到疑惑，可以先去[官方文档](https://hexo.fluid-dev.com/docs/advance/#hexo-%E6%B3%A8%E5%85%A5%E4%BB%A3%E7%A0%81) 了解一下注入代码的功能：
>Hexo 注入代码的方法：
>*编写注入代码，需要在博客的根目录下创建 `scripts` 文件夹，然后在里面任意命名创建一个 js 文件即可。*

主要的过程在这位大佬的博客里面叙述的非常详细，此处就不重造轮子，重点记录一下后面添加图片以及站点显示的一些文本如何对应源代码设置：
![](image-20231111164045874.png)
1. 导航栏的显示，考虑多语言修改
```yml
# file: "/_config.fluid.yml"
menu:
    - { key: "home", link: "/", icon: "iconfont icon-home-fill" }
    - { key: "archive", link: "/archives/", icon: "iconfont icon-archive-fill" }
    - { key: "category", link: "/categories/", icon: "iconfont icon-category-fill" }
    - { key: "tag", link: "/tags/", icon: "iconfont icon-tags-fill" }
    - { key: "about", link: "/about/", icon: "iconfont icon-user-fill" }
    - { key: "links", link: "/links/", icon: "iconfont icon-link-fill" }
    - { key: "pic", link: "/pic/", icon: "iconfont icon-images" } # 此处
```
```yml
# file: "/node_modules/hexo-theme-fluid/languages/zh-CN.yml"
pic:
  menu: '图集'
  title: '图集'
  subtitle: '图集'
```
2. 页面打字机效果主标题
```
# file: "source/pic/index.md"
---
title: 图集 # 此处
date: 2023-11-11 15:59:01
layout: pic
---
```
![](image-20231111165046333.png)

3. 图集分类
4. 图片名称
都是本地路径下的文件夹与文件名
# 03-结语

感谢大佬们的博客，此次实现图片瀑布流的过程十分迅速。
后期对于显示效果上的优化以及图片的丰富，可以之后再次参考一下大佬们的源代码：
-  `scripts/createPhotoList.js` ：获取图片信息
-  `source/pic/index.md` ：设置容器样式
-  `source/js/photoWall.js` ：绘制图片容器
-  `scripts/injector.js` ：注入代码