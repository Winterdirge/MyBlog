---
layout:     post
title:      "扬帆，起航（二）"
subtitle:   "YG的博客搭建之旅"
date:       2019-07-11 10:31:51 GMT+8
author:     "YangGuang"
header-style: text
tags:
    - Jekyll
    - 博客搭建
    - 技术
---

# 回顾
上篇文章介绍了Jekyll在本地和服务器上的环境配置，并利用Jekyll自带服务跑起来一个简单的博客系统，本篇文章主要介绍如何将这个系统跑在服务器上。

# 模版选择
由于本文旨在记录搭建博客的一个过程，并不会对界面布局以及每个页面的详细代码进行解读，因此为了方便快捷，这里我们通过现有的Jekyll模版构建。大家可以从[这里](http://jekyllthemes.org/)找到你喜欢的主题，笔者这里采用的是[hxpro的主题](https://github.com/Huxpro/huxpro.github.io)，大家也可以使用[onevcat的主题](https://github.com/onevcat/OneV-s-Den)，根据自己的喜好选择即可。笔者这里选用的是hxpro的主题进行搭建，笔者现在的博客也是用的这个主题。

# 主题下载
这个没啥说的，从Github将克隆代码到本地即可。

# 本地预览
这个也没的说，代码clone下来以后，到该目录运行`jekyll serve`，就可以在本地看效果了，还是`localhost:4000`

# 上传服务器
在上一步运行`jekyll serve`以后，大家可能会发现文件夹下多了一个`_site`目录，该目录下存放着每一篇文章生成的html文件，这些文件就是用户在浏览器下真正访问的地方，所以我们只需要将这个目录上传到服务器，按理说就可以访问到这些文章，事实上也是如此。

## nginx配置
笔者不太懂后台，也只会改改nginx的配置文件。对于本文来说，我们只需要修改nginx的root字段即可，在服务器`/etc/nginx`下找到`nginx.conf`文件，打开后找到如下字段

```
server {
    listen       80 default_server;
    listen       [::]:80 default_server;
    server_name  _;            
    root         /home/xxx;

    //此处省略代码若干

    location / {
    }
                                    
    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```
我们需要做的仅仅是修改root后面的具体文件地址即可，以笔者的配置为例，笔者这里将其改为`/home/myname/www/blog`，就是告诉nginx当用户向它请求时，它需要返回`blog`文件夹下的`index.html`，到这里nginx已经配置完成，是不是很简单。
~~还请后台的大佬们别喷我，我是真的不懂后台~~

## 上传文件
在这一步我们需要做的事情就比较简单了，只需要将`_site`文件夹下的东西，上传到服务器`/home/myname/www/blog`目录下即可，使用`scp`命令即可。
```
scp -r ~/blog/_site username@192.168.0.1:/home/myname/www/blog
```
## 最后一步
重载nginx服务，让其配置生效，使用如下命令即可。
```
nginx -s reload
```
# 最终效果
到目前为止，你的博客已经可以对外访问了，赶快输入你的域名去看看你自己的博客吧。当然，博客的搭建过程还是比较简单的，难得是你用什么内容来充实你的博客，是技术，是情感，还是艺术，这都由你自己来书写。

# 自动部署
是不是觉得每次更新都要build一下，然后在scp到服务器很麻烦，有没有什么方法可以在写完代码以后让服务器帮你自动构建，答案是有的。

笔者是直接把博客相关代码全都提交到服务器，设置Git post-receive钩子，每次提交代码的时候都会执行一遍部署脚本，来达到自动部署的目的。

在服务器创建Git仓库
```
mkdir blog.git
cd blog.git
git init --bare
```
设置`post-receive`脚本，找到`blog.git/hooks`文件加下的`post-receive.sample`文件，cp一份改名为`post-receive`，如果没有此文件，就直接创建，之后添加如下内容，具体的目录需要读者自己修改
```shell
#!/bin/sh
GIT_REPO=/home/myname/www/blog.git
TMP_GIT_CLONE=/home/myname/tmp/blog
PUBLIC_WWW=/home/myname/www/blog

git clone $GIT_REPO $TMP_GIT_CLONE
jekyll build -s $TMP_GIT_CLONE -d $PUBLIC_WWW
rm -rf $TMP_GIT_CLONE
exit
```
然后赋予改脚本执行权限
```
chmod +x post-receive
```
至此自动部署脚本已完成，提交代码即可观察效果。

# HTTPS
由于在阿里云可以申请免费的ssl证书，那为什么不把自己的博客搞成https的呢，于是笔者说干就干，申请流程在此不再赘述。此步骤假设你已经拿到证书的`.pem`和`.key`，我们需要做的就是上传这两个文件到服务器，然后修改nginx的配置文件。

找到之前修改80端口的地方，修改为下面内容
```
server {
    listen       80 default_server;
    listen       [::]:80 default_server;
    return 301 https://your_domain_name$request_uri;
}
```
找到443端口，修改ssl_certificate字段以及ssl_certificate_key字段，分为为`.pem`和`.key`文件在服务器上的地址，其他配置按需修改。

```
server {
    listen       443 ssl http2 default_server;
    listen       [::]:443 ssl http2 default_server;
    server_name  your_domain_name;
    root         /home/yourname/www/blog;

    ssl_certificate "/etc/pki/nginx/xxxx.pem";
    ssl_certificate_key "/etc/pki/nginx/xxxx.key";
    //省略部分代码
}
```
至此HTTPS已经设置完成，愉快的开始你的博客之旅吧。

于2019.7.11 11:59:40 理想国际大厦