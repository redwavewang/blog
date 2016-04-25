+++
date = "2016-04-22T16:57:27+08:00"
description = ""
draft = false
tags = ["hugo"]
title = "使用Hugo创建个人博客"
topics = ["blog"]

+++

Hugo 是一个用 Go 语言编写的静态网站生成器,简单易用.Hugo可以在任何平台上使用,只需要下载二进制文件即可使用,使用起来完全无痛!

<!--more-->

## Step 1. 安装Hugo

在OSX下通过brew即可安装:

```
$ brew install hugo
```

其他平台可以直接下载预编译好的可执行文件,如Windows可以下载[二进制文件](https://github.com/spf13/hugo/releases),下载后即可使用.

## Step 2. 创建网站

将Hugo放置到系统路径下,执行如下命令即可创建网站:

```
$ hugo new site blog
```

执行命令后,会在当前目录下生成一个blog目录,其中就包含了目前网站的基本配置,目前会用到如下几个文件或文件夹:

- **config.xml**: 这个是我们网站的配置文件,后续会看到,针对不同的主题,这个配置文件会有一些差异.
- **content**: 所有的网站内容都是放在该目录下
- **static**: 静态文件,如图片等数据都是存放在该目录下

## Step 3. 添加内容

执行如下内容,创建第一篇日志:

```
$ hugo new post/first-commit.md
```

这样会在content目录下生成一个包含hello-world.md的post目录.打开该文件,可以看到如下内容:

```toml
+++
date = "2016-04-22T16:57:27+08:00"
description = ""
draft = true
title = "hello-world"

+++
```

顾名思义,这些内容就是这篇日志的一些基本信息,如标题,描述,创建时间等.随便写一些内容进去:

```toml
+++
date = "2016-04-22T16:57:27+08:00"
description = ""
draft = true
title = "hello-world"

+++

hello world

<!--more-->
```

这样第一篇日志就创建成功了.注意,可以通过添加一行<\!--more-->来分隔简介和内容,这样在主页上就只会显示该项前的内容.相当于为首页提供一个提要.

## Step 4. 创建主题

下一步,我们需要为日志获取它的主题,目前hugo支持的主题可以在[这里](http://themes.gohugo.io/)浏览.我目前选择的是BlackBurn主题:

```sh
$ mkdir themes
$ cd themes
$ git clone https://github.com/yoshiharuyamashita/blackburn.git
```

下载成功后,将theme\blackburn\exampleSite\config.toml拷贝到blog目录下,查看该文件,修改对应的一些属性项:

```toml
baseurl = "http://replace-this-with-your-hugo-site.com/"
languageCode = "en-us"
title = "Blackburn Theme Demo"
theme = "blackburn"
author = "Yoshiharu Yamashita"
copyright = "&copy; 2016. All rights reserved."
canonifyurls = true
paginate = 10

[indexes]
  tag = "tags"
  topic = "topics"

[params]
  # Shown in the home page
  subtitle = "A Hugo Theme"
  brand = "Blackburn"
  googleAnalytics = "Your Google Analytics tracking ID"
  disqus = "Your Disqus shortname"
  # CSS name for highlight.js
  highlightjs = "androidstudio"
  dateFormat = "02 Jan 2006, 15:04"

[menu]
  # Shown in the side menu.
  [[menu.main]]
    name = "Home"
    pre = "<i class='fa fa-home fa-fw'></i>"
    weight = 0
    identifier = "home"
    url = "/"
  [[menu.main]]
    name = "Posts"
    pre = "<i class='fa fa-list fa-fw'></i>"
    weight = 1
    identifier = "post"
    url = "/post/"
  [[menu.main]]
    name = "About"
    pre = "<i class='fa fa-user fa-fw'></i>"
    weight = 2
    identifier = "about"
    url = "/about/"
  [[menu.main]]
    name = "Contact"
    pre = "<i class='fa fa-phone fa-fw'></i>"
    weight = 3
    url = "/contact/"

[social]
  # Link your social networking accouns to the side menu
  # by entering your username or ID.
  twitter = "*"
  facebook = "*"
  instagram = "*"
  github = "yoshiharuyamashita"
  stackoverflow = "*"
  linkedin = "*"
```

## Step 5. 调试网站

有了主题和内容,就可以调试网站了,执行命令:

```sh
$ hugo server --theme=blackburn
```

可以看到提示,打开[localhost:1313](http://localhost:1313),可以看到,什么东西都没有,是怎么回事呢?

原来Hugo提供了一个草稿的步骤,默认我们创建的文档都是草稿状态,需要将该状态修改为非草稿状态才会在页面中显示出来,否则只能通过如下命令来查看:

```sh
$ hugo server --theme=blackburn --buildDraft
```

为了将文档修改为非草稿状态,执行命令:

```sh
$ hugo undraft content/post/hello-world.md
```

或者直接修改hello-world.md文件头中的draft属性,将其修改为false即可.

再次执行:

```sh
$ hugo server --theme=blackburn
```

打开[localhost:1313](http://localhost:1313)就可以看到我们的网页了.


## Step 6. 发布网站

执行如下命令:

```sh
$ hugo --theme=blackburn
```

会在当前目录下生成一个public文件夹,这个就是我们发布出来的网站了,通过
github的网页服务,就可以展示我们的静态博客了.

## Step 7. CDN加速

默认下载的主题中有一些css，js文件的加速链接，但是由于在国内网速的问题，需要修改为国内的一些替代CDN服务的链接，目前我遇到的有如下这些：

```
https://cdnjs.cloudflare.com/ajax/libs/pure/0.6.0/
--->
https://cdn.bootcss.com/pure/0.6.0/


https://maxcdn.bootstrapcdn.com/font-awesome/4.5.0/css/font-awesome.min.css
--->
https://cdn.bootcss.com/font-awesome/4.5.0/css/font-awesome.css


https://fonts.googleapis.com/css?family=Raleway
google字体没有找到，可以直接注释掉

//cdnjs.cloudflare.com/ajax/libs/highlight.js
--->
//cdn.bootcss.com/highlight.js


//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.1.0/highlight.min.js
--->
//cdn.bootcss.com/highlight.js/9.2.0/highlight.min.js
```
