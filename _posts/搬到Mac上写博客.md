---
title: 搬到Mac上写博客
date: 2016-12-02 00:11:10
categories:
- tools
tags:
- blog
---

### 缘起
今年双十一在老婆的鼓励之下，入手了一个MacBook Air。我觉得苹果电脑好用的精髓就是，从此可以告别鼠标了，不论干啥事，都很舒坦。熟悉了几天之后，就萌生了把之前的hexo博客写作环境搬到Mac上来。于是乎就折腾了一番。


### 具体步骤
---
#### 一.必备的工具
1. node.js下的巨慢无比。
2. git：xcode安装好就可以了。


#### 二.hexo
1. 安装完成hexo之后
 ```
 $ npm install hexo-deployer-git --save
 $ mkdir blog
 $ hexo init blog
 $ cd blog
 $ rm -rf source
 $ git clone git@github.com:niaokedaoren/blog-src.git source
 $ git clone https://github.com/wuchong/jacman.git themes/jacman
 ```
2. 拷贝source目录下备份的 _config.yml到相应目录下即可。
3. 修改一下scaffolds的post.md
 ```
 ---
 title: {{ title }}
 date: {{ date }}
 categories:
 tags:
 ---
 ```