---
title: "Hugo搭建个人博客教程"
date: 2018-07-22T00:11:56+08:00
draft: false
---


    一直有想整理自己博客的想法，但是由于各种原因一再搁置，第一是通过写文章让自己的技术得以沉淀，第二也可以进行交流，互相成长。第三也可以向大家展示自己的写作风格以及中间贯穿的一些思想。



# 组件介绍

* [hugo](http://gohugo.io/) 主要生成博客的工具。

* [github](https://github.com) 需要用github来托管网站。

* [markdown](https://www.markdowntutorial.com/)你需要知道的markdown语法。

* [域名](https://wanwang.aliyun.com/domain/)别人访问你网站的网址。
* [ipic](https://toolinbox.net/iPic/)图床神器，可以上传一些图片。

> <font color=red>注册好自己的域名和github之后，就可以构建自己的网站啦。</font>

# hugo

## 简介

hugo是使用go语言编写的，有几百种主题可供选择，也可以与自己个性化设置等相结合。官网上说hugo是世界上最快的网站构建框架，也是最受欢迎的开源静态站点生成器之一。凭借其惊人的速度和灵活性，hugo使网站构建变得更加有趣。

## 安装

Mac版安装

```shell

brew install hugo

```

查看是否安装成功

```

$ hugo version

Hugo Static Site Generator v0.41 darwin/amd64

```

## 使用

首先构建自己的网站，wtlinux.com是我注册的域名

```

$ hugo new site wtlinux.com

```

查看目录结构

```

$ tree wtlinux.com/

wtlinux.com/

├── archetypes

│ └── default.md

├── config.toml

├── content

├── data

├── layouts

├── static

└── themes

```

`archetypes`构建新文章是默认的配置

`config.toml`网站的主要配置文件

`content`博客文章的目录

`layouts`网站如何展示的一些前端的东西

`static`存放网站图片等静态内容

`themes`存放网站主题的地方

然后进入到[hugo主题选择](https://themes.gohugo.io/),选择自己喜欢的主题进行clone。

```

cd  wtlinux.com

git clone https://github.com/taikii/whiteplain.git themes/whiteplain

```

这时候whiteplain的主题就在themes的目录中了。

现在我们来创建一个页面：

```

hugo new about.md

```

然后查看content里面多了一个about.md文件

```

$ cat content/about.md

---

title: "About"

date: 2018-07-21T15:27:10+08:00

draft: true

---

# 关于我

2018 ❤️

```

> <font color=red>注意</font>当`draft: true`的时候，视为草稿，`hugo server -D`会显示，正式发布效果用`hugo server -w`查看。


在里面加入了一些内容，然后运行

```

hugo server -t whiteplain --buildDrafts

```

`-t`指定whiteplain主题

`--buildDrafts`使用草稿的方式运行



浏览器进行访问：http://localhost:1313/ 

![](https://ws3.sinaimg.cn/large/006tNc79gy1fthir76ckpj30s20cuq3o.jpg)

把主题中的layouts和static替换到自己网站下面

```

$ cp -R themes/whiteplain/layouts/ ./wtlinux.com/layouts

$ cp -R themes/whiteplain/static/ ./wtlinux.com/static

```



# 评论功能

hugo默认支持[Disqus](https://disqus.com/)的评论，需要去disqus进行注册create a new site

![](https://ws3.sinaimg.cn/large/006tNc79gy1fthk6gs7ynj30ii0j5jsq.jpg)

然后选择基本的套餐

![](https://ws2.sinaimg.cn/large/006tNc79gy1fthkaicbt2j30750fkq3s.jpg)

然后选择将要使用的地方

![](https://ws2.sinaimg.cn/large/006tNc79gy1fthkbisgyoj30u308egms.jpg)

然后填写自己blog的网址

![](https://ws4.sinaimg.cn/large/006tNc79gy1fthkcou8xyj30t10loack.jpg)

然后在config.toml中添加如下代码：

```

disqusShortname = "yourdisqusShortname"

```

> 例如：disqusShortname = "wtlinux"



还需要在模板中添加如下代码：

```

$ cat layouts/partials/disqus.html

<div id="disqus_thread"></div>

<script type="text/javascript">



(function() {

    // Don't ever inject Disqus on localhost--it creates unwanted

    // discussions from 'localhost:1313' on your Disqus account...

    if (window.location.hostname == "localhost")

        return;



    var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;

    var disqus_shortname = '{{ .Site.DisqusShortname }}';

    dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';

    (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);

})();

</script>

<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>

<a href="http://disqus.com/" class="dsq-brlink">comments powered by <span class="logo-disqus">Disqus</span></a>

{{ template "_internal/disqus.html" . }}

```

在`{{ partial "footer.html" . }}`下面添加`{{ partial "disqus.html" . }}`

```

<!DOCTYPE html>

<html lang="{{ .Site.LanguageCode }}">

{{ partial "head.html" . }}

<body>

{{ partial "header.html" . }}

{{ block "main" . }}{{ end }}

{{ partial "footer.html" . }}

{{ partial "disqus.html" . }}

</body>

</html>



```



再次运行，就可以看到评论的功能相关的啦

```

 hugo server -t whiteplain --buildDrafts

```

![](https://ws2.sinaimg.cn/large/006tNc79gy1fthkh8a8szj30j609yt9k.jpg)



# 代码高亮

在layouts/partials/header.html加入如下代码：

```

<script src="https://yandex.st/highlightjs/8.0/highlight.min.js"></script>

<link rel="stylesheet" href="https://yandex.st/highlightjs/8.0/styles/default.min.css">

<script>hljs.initHighlightingOnLoad();</script>

```

# 用Github Pages来作为网站的host

先创建一个project

![](https://ws4.sinaimg.cn/large/006tNc79gy1fthlbzsisfj30k30h3mze.jpg)

然后将wtlinux.com目录上传上去

```

cd wtlinux.com

git init
git add .
git commit -m "first commit"
git remote add origin git@github.com:wtli/wtlinux.com-hugo.git
git push -u origin master
```

然后接下来步骤如下：

1、在github上创建repo，托管hugo的输出文件。

2、创建repo<username>.github.io,用于托管public文件夹，注意这里repo名字一定要用自己的用户名，才会被当做是个人主页。

3、clone your project

```

$ git clone wtli.github.io

```

4、进入project目录

```

$ cd wtli.github.io

```

6、删掉public目录

```

$ rm -rf public

```

7、把public目录添加为submodule与.github.io同步

```

$ cd wtlinux.com/

$ git submodule add git@github.com:<username>/<username>.github.io.git public

```

8、添加.gitignore文件，里面写public/，在同步\<your-project\>-hugo时会忽略publice文件夹。

```

$ cat .gitignore

public/

```

10、有一个写好的scripts脚本，直接拷贝就能用，记得chmod +x depoly.sh加上运行权限

```

$ cat deploy.sh

#!/bin/bash

echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

msg="rebuilding site `date`"

if [ $# -eq 1 ]

  then msg="$1"

fi

# Push Hugo content

git add -A

git commit -m "$msg"

git push origin master

# Build the project.

hugo # if using a theme, replace by `hugo -t <yourtheme>`

# Go To Public folder

cd public

# Add changes to git.

git add -A

# Commit changes.

git commit -m "$msg"

# Push source and build repos.

git push origin master

# Come Back

cd ..

```

等一会儿，就可以在http://username.github.io/ 这个页面上看到你的网站了，每次更新，只需要运行着脚本就行。



# 域名绑定

步骤如下：

1、在wtli.github.io的project中setting，在下面可以改成www.wtlinux.com

2、在域名配置中，添加cname，然后把地址指向wtli.github.io



# 参考文章

* 宋净超的个人网站：[jimmysong.io](https://jimmysong.io/posts/building-github-pages-with-hugo/)

* 一个妹子的博客：[nanshu wang](http://nanshu.wang/)