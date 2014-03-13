#From zero to fully working CI server in less than 10 minutes with Drone & Docker



#用 Drone 和 Docker 10 分钟搭建全功能的 CI 服务器

***

What is Drone? The official description is: "Drone is a Continuous Integration platform built on Docker".

什么是 [Drone](https://github.com/drone/drone) ？ 官方的解释：“ Drone 是基于 Docker 构建的持续集成平台”。

***

What is [Docker](http://docker.io)? Again, official description is :"Docker is an open-source project to easily create lightweight, portable, self-sufficient containers from any application".

什么是 [Docker](http://docker.io) ？官方的解释：“ Docker 是一个开源项目，它是一个容易创建的轻量级、易移植和对任何程序都是全功能的容器。”

***

10 minutes. This is all you need. In fact, there is an extra buffer in those 10 minutes to get your configuration working.

10 分钟，你只需要 10 分钟。事实上，10 分钟外还需要额外的时间让你的配置工作起来。

***

10 minutes is by order of magnitude less time than it would take you to get a Jenkins server up & running. I have never been a fan of Jenkins, and probably won't ever be, but let's keep this for another post, maybe.

10 分钟和让一台 [Jenkins](http://jenkins-ci.org) 服务器启动和运行所需的时间差了个数量级。我从来都不是 [Jenkins](http://jenkins-ci.org) 的粉丝，或许永远都不会是。还是让我们在其它文章中讨论这个问题吧。

***

REQUIREMENTS

需要的条件

***

We will assume that you have a Ubuntu 13.04 (64 bit) server with a routable IP or address. On Digital Ocean, which I love (referral here, thanks!), you can even have a VPS pre-built with Docker, 0.8 as of time of writing.

我们假设你有一台可以访问的 64 位 Ubuntu 13.04 版本服务器。在 Digital Ocean (我非常喜欢 Digital Ocean，请使用我的[推荐链接](https://www.digitalocean.com/?refcode=9b3537dd733f)，谢谢～) 你可以使用预先安装好 Docker 版本的 VPS ，在写这篇文章的时候 Docker 的版本是 0.8 。

***

You can also use a Ubuntu 12.04 64 bit box in theory for Drone.

理论上来说 64 位 Ubuntu 12.04 服务器也可以运行 Drone 。

***

Your application needs to be hosted on GitHub. BitBucket coming soon apparently, but for now, this works only with GitHub which is fine by me. I love GitHub.

你的应用需要托管在 [Github](http://github.com) ，貌似 [Bitbucket](http://bitbucket.org) 很快就会支持。目前只支持 [Github](http://github.com) 对我来说没什么问题，我喜欢 [Github](http://github.com) 。

***

STEP 0: INSTALL DOCKER (OPTIONAL, SKIP IF DOCKER IS ALREADY INSTALLED)

第 0 步：安装 Docker (可选，如果你已经安装了 Docker 可以跳过此步)

***

On your Ubuntu machine, run this:

在你的 Ubuntu 机器上执行：


```
curl -s https://get.docker.io/ubuntu/ | sudo sh
```

***

You can verify the installation with:

你可以使用下面的命令验证安装：

```
sudo docker run -i -t ubuntu /bin/bash
```

***

STEP 1: INSTALL DRONE

第 1 步：安装 Drone

***

This is very simple to install!

Drone 安装非常简单！

***

You gotta do:

你要做的就是：

***

```
wget http://downloads.drone.io/latest/drone.deb
sudo dpkg -i drone.deb
```
***

The README also adds "sudo start drone" but it was already started for me.

在 README 中有 “sudo start drone” 启动的部分，但是安装程序已经为我启动了 Drone 。

***

To finish the installation, navigate to http://my-server-ip-or-addr:80/install and follow the steps in the wizard (account creation). Once logged in, keep that browser tab open.

安装完成后，用浏览器打开 http://my-server-ip-or-addr:80/install ，根据向导创建账号。一旦登陆，请保持浏览器一直打开。

***

STEP 2: REGISTER A NEW APPLICATION ON GITHUB

第 2 步：在 Github 注册一个新的应用

***

You now have Drone up and running, but we need to configure GitHub access so that it can setup hooks to build your code after code push or pull requests. First step for that is to register a new application on GitHub. [It is done right here](https://github.com/settings/applications/new).

现在你已经有一个 Drone 的服务在运行了，但是我们必须配置一个 Github 的应用使得 Drone 在 Github 的 Pull 或 Push 后能够 Build 你的代码。[在这里注册一个新的程序](https://github.com/settings/applications/new)。

***

* Pick up a name you, it could be "my Drone server" or whatever makes sense to you
* Set the homepage URL to http://my-server-ip-or-addr/
* The description is up to you, just like the name
* The authorization callback URL should be set to http://my-server-ip-or-addr/auth/login/github

* 选一个名字，可以叫“ my Drone server ” 或者其它任何对你有意义的名字
* 设定主页地址是 http://my-server-ip-or-addr/
* 描述就像名字一样，对你有意义就可以
* Github OAuth 认证的回调地址是 http://my-server-ip-or-addr/auth/login/github

***

STEP 3: GET YOUR GITHUB CLIENT ID & SECRET CONFIGURED IN DRONE

第三步：在 Drone 中设置 Github 的 Client ID 和 Client Secret 

***

After the registering your new app on GitHub, you were given a client ID and secret, copy them in the "GitHub OAuth Consumer Key and Secret" section of http://my-server-ip-or-addr/account/admin/settings

在 Github 注册完成新应用后，你会得到 Client ID 和 Client Secret，把它们拷贝到 http://my-server-ip-or-addr/account/admin/settings 页面的 "GitHub OAuth Consumer Key and Secret" 部分。

***

STEP 4: LINK YOUR PROJECT

第四步：链接你的项目

***

Click on the "New Repository" button (which will get you to http://my-server-ip-or-addr/new/github.com), click "Link Now", accept and then once on the "Repository Setup" page of Drone, you get to fill the repository details (GitHub owner + repository name).

点击  “New Repository”  按钮页面会转跳到 http://my-server-ip-or-addr/new/github.com 这里，点击 “Link Now” 按钮，一旦接受了 Drone 的 Repository Setup 页的，你会得到 Github 账户内所有代码仓库的信息(账户名+仓库名)。

***

Boom! Your project is almost ready to be built. You need only one final step now...

Boom！你的项目几乎已经可以进行集成测试了，你现在只需要完成最后一步...

***

STEP 5: CONFIGURE YOUR .DRONE.YML

第 5 步：配置你的 .drone.yml 文件

***

The .drone.yml file is the configuration file for the build steps, services, notifications, etc.

.drone.yml 文件是 Drone 用来配置编译步骤、各种服务和通知等的配置文件。

***

Here is a simple example for a Ruby on Rails project:

下面是一个简单的 Ruby on Rails 项目的配置文件：

```
image: ruby2.0.0
script:
  - cp config/database.drone.yml config/database.yml
  - bundle install
  - psql -c 'create database test;' -U postgres -h 127.0.0.1
  - bundle exec rake db:schema:load
  - bundle exec rspec spec
services:
  - postgres
notify:
  email:
    recipients:
      - email@example.com
```

***

You are wondering what that database.drone.yml file contains? See this gist.

你一定很疑惑 database.drone.yml 是什么，[请看这个gist](https://gist.github.com/jipiboily/0cad2550be91f5c9b5d)

***

There is a variety of images for Go, Python, Haskell, PHP, Scala, Node, etc. Drone provides us with [official images](https://github.com/drone/drone#images) but you are not restricted to only those.

这里有很多 Go、Python、Haskell、PHP、Scala、Node 的镜像。Drone 为我们提供了这些[官方镜像](https://github.com/drone/drone#images)，但是你可以使用其它的镜像。

***

There is also a bunch of services available to you.

[这里还有很多服务的镜像](https://github.com/drone/drone#databases)

***

All the details on how to configure your .drone.yml file can be found in the README and drone.io's documentation.

可以在 [README](https://github.com/drone/drone#builds) 和 [drone.io](http://drone.io/) 的[官方文档](http://docs.drone.io/)中找到如何设置 .drone.yml 文件的说明。

***

When you are done, commit, push and watch the build. Rinse & repeat until your .drone.yml is working just fine.

当你完成这些后，commit，push 和 watch 你的代码时 Drone 就会 Build 。反复重写你的 .drone.yml 直到没有问题。 

***

For your information, I noticed that the first build takes a few minutes.

当第一次 Build 代码的时候会花上一些时间。

***

EXTRA, EXTRA!

特别的提示！

***

Here a few things that you might want to consider that Drone offers you:

下面是一些 Drone 提供给的，你可能需要的功能：

***

* Email notifications (SMTP settings at the bottom of http://my-server-ip-or-addr/account/admin/settings)
* SSL is available to you and you should really consider it (at the top of http://my-server-ip-or-addr/account/admin/settings)
* HipChat notifications
* Continuous deployment is also available to you

* 邮件通知( SMTP 设置在  http://my-server-ip-or-addr/account/admin/settings 底部 )
* SSL 设置( SSL 设置在 http://my-server-ip-or-addr/account/admin/settings 顶部 )
* HipChat 通知
* 可持续部署

***
CONCLUSION

结论

***
That's it, it just works. Should you use that instead of Jenkins, Codeship or other hosted CI as a Service? It's up to you. This project was just publicly released a week ago so I am not sure that I would use that for my job's code just yet, but I will pay close attention to this.

就是这些，让你可以部署一个全功能的 CI 服务器。是不是应该使用它替代 Jenkins、Codeship 或者其它的 CI 服务么？这完全取决于你。这个项目是一周前发布的，我还不能确定是否把它用在我工作的项目中，但是我会保持关注。

***
Let me know if you like it or not!

让我知道你喜欢还是不喜欢 Drone！

***
PS: if you love Docker and Heroku-like PaaS, you might also be interested by my post about getting Dokku (you're very own Heroku-like setup) installed.

另外：如果你喜欢 Docker 和 类似 Heroku 的 PaaS 服务，你可能会感兴趣我的另一篇关于 [Dokku 安装](http://jipiboily.com/2013/install-dokku-postgresql-with-docker-for-your-rails-app-or-whatever-else-almost) 的文章。

---
#####这篇文章由 [Jean Philippe](https://github.com/jipiboily) 在2014年2月13日发布，点击 [这里](http://jipiboily.com/2014/from-zero-to-fully-working-ci-server-in-less-than-10-minutes-with-drone-docker)可以阅读原文。

#####These article was contributed by [Jean Philippe](https://github.com/jipiboily), click [here](http://jipiboily.com/2014/from-zero-to-fully-working-ci-server-in-less-than-10-minutes-with-drone-docker) to read the original publication.

