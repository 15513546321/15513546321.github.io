---
layout: post
title:  "博客使用说明"
date:   2019-06-27 13:31:01 +0800
categories: blog
tag: blog
---

* content
{:toc}


打开 msysgit.github.io，安装完成后，进入到要托管的项目根目录，下载并安装最新版本的msysgit

在我的github上面找到博客下载地址，地址如下

![地址]({{ '/styles/images/address.jpg' | prepend: site.baseurl  }})

[诸葛亮](#)
诫子书				{#zhugeliang}

右键启动Git Bash命令行

在命令框中执行git clone https://github.com/15513546321/15513546321.github.io.git

将博客下载到本地后，找到_posts，在这个文件夹下可以按照指定格式生成博客

上传博客到github，按顺序执行

右键博客所在文件夹，打开Git Bash

git config --global user.email "m15513546321_2@163.com"
git config --global user.name "15513546321"

git add .    （注：别忘记后面的.）

git commit  -m  "提交信息"  （注：“提交信息”里面换成你需要，如“first commit”）

git push -u origin master   （注：此操作目的是把本地仓库push到github上面，此步骤需要你输入帐号和密码）

访问15513546321.github.io
