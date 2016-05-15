title: hexo + github 搭建blog
date: 2015-07-14 22:33:38
categories:
- tools
tags:
- hexo
- github
- blog
---

网络上，随便搜一下hexo、github等关键词，就能出来一大堆相当详尽的博客搭建介绍，
我也是一步步跟着做下来，本文主要记录一些自己碰到的问题。

### 安装
1. 安装[Node.js](https://nodejs.org/)和git，这里不再赘述
2. 检查npm是不是在PATH下面
 ``` bash
 $ which npm
 ```

3. 安装hexo
 ``` bash
 $ npm install -g hexo-cli
 ```

 转圈圈运行之后，报错
 ```
 npm ERR! registry error parsing json
 npm ERR! Linux 2.6.32-358.el6.x86_64
 npm ERR! argv "<nodejs-dir>/bin/node" "<nodejs-dir>/bin/npm" "install" "-g" "hexo-cli"
 npm ERR! node v0.12.6
 npm ERR! npm  v2.11.2
 npm ERR! Unexpected end of input
 ```

 可能是被墙了，加一个国内的源
 ``` bash
 $ npm config set registry https://registry.npm.taobao.org
 ```
 现在就可以顺利安装了。

### 创建blog
1. 在github上建一个名为blog的repo，然后clone到本地，在gh-pages分枝上建立
project page的方式来存放自己的blog
 ``` bash
 $ git clone https://github.com/niaokedaoren/blog.git
 $ cd blog
 $ git checkout --orphan gh-pages
 $ git rm -rf .
 $ git add .
 $ git commit -m "initial commit"
 $ git push origin gh-pages
 ```
 报错：
 error: src refspec github-pages does not match any.
 error: failed to push some refs to 'https://github.com/niaokedaoren/blog.git'
 解决： push不能为空
 ``` bash
 $ touch README
 $ git add .
 $ git commit -m "add dummy README"
 $ git push origin gp-pages
 ```
 上传一个rsa public key到github，可以实现无密钥登陆。

2. hexo建立blog
 ```bash
 $ hexo init blog
 $ cd blog
 $ npm install
 ```
 生成blog并启动hexo server，可以在 http://localhost:4000访问blog
 ```bash
 $ hexo g
 $ hexo s
 ```

3. 配置_config.yml
 - deploy
  ```
  deploy:
    type: git
    repo: ssh://git@github.com/niaokedaoren/blog.git
    branch: gh-pages
  ```
  **注意**: 如果这里repo写成https://github.com/niaokedaoren/blog.git
  就会报错：[*pushing-to-git-returning-error-code-403-fatal-http-request-failed*](http://stackoverflow.com/questions/7438313/pushing-to-git-returning-error-code-403-fatal-http-request-failed)

 - url
  ```
  url: http://niaokedaoren.github.io/blog
  root: /blog/
  ```
  否则部署到github上会无法加载css

4. 选择主题

 我选择的是[jacman](http://wuchong.me/blog/2014/11/20/how-to-use-jacman/)
 ```bash
 $ cd <blog-dir>
 $ git clone https://github.com/wuchong/jacman.git themes/jacman
 ```
 可以注册一个多说，在**themes/jacman/_config.yml**配置一下多说shortname。

5. 常用的命令
 ```bash
 $ hexo new "post name" #写新的博客
 $ hexo new draft "draft name" #写博客草稿, scaffolds目录下可以编辑layout
 $ hexo clean #清理生成的页面
 $ hexo g
 $ hexo s
 $ hexo d
 ```

### 写作工具
我用的是sublime text2，具体配置参考http://www.jianshu.com/p/378338f10263


*参考*
 1. http://cnfeat.com/2014/05/10/2014-05-11-how-to-build-a-blog/
 2. http://www.aips.me/hexo-independent-blog-new-ways.html
