这篇博文涵盖了使用 [Mono](http://www.mono-project.com/Main_Page) 在 [Docker](https://www.docker.io/) 轻量容器中运行简单 .NET 应用的方法。我在 Windows 上的 Vagrant/VirtualBox VM上运行 Docker 。它运行的很好，也很快。在 Docker 网站有 [安装指导](http://docs.docker.io/en/latest/installation/windows/) 。

##构建基础镜像

首先是要创建安装有 Mono 的 Docker 镜像。我们将使用它作为基础镜像，以使容器可以真正运行应用。为了得到最新的 Mono 版本（写本文时是3.2.6），我使用 [由 Timotheus Pokorra 创建](http://software.opensuse.org/download/package?project=home:tpokorra:mono&package=mono-opt) 的、安装在 Ubuntu 12.04 LTS 上的 Docker 镜像。下面是它的 [Dockerfile](http://docs.docker.io/en/latest/use/builder/) ：

```
FROM ubuntu:12.04	
MAINTAINER friism
	
RUN apt-get -y -q install wget
RUN wget -q http://download.opensuse.org/repositories/home:tpokorra:mono/xUbuntu_12.04/Release.key -O- | apt-key add -
RUN apt-get remove -y --auto-remove wget	
RUN sh -c "echo 'deb http://download.opensuse.org/repositories/home:/tpokorra:/mono/xUbuntu_12.04/ /' >> /etc/apt/sources.list.d/mono-opt.list"	
RUN apt-get -q update		
RUN apt-get -y -q install mono-opt
```

下面是对应的解释：

1. 安装wget
2. 添加仓库密匙到apt-get
3. 删除wget
4. 在源列表中添加openSUSE仓库
5. 从仓库安装Mono

刚开始我本想将安装 mono 的过程放在一个命令中，因为其实现步骤也只是简单的提交。如果已经安装了 wget 那更没人会关注提交的问题了，因此我没有像别人那样通过多个 run 命令来实现。通过 Dockerfile 就可以直接构建一个镜像了:

```
$ docker build -t friism/mono .
```

接下来运行包含刚刚生成的镜像的容器来完成 Mono 安装:

```
vagrant@precise64:~/mono$ docker run -i -t friism/mono bash
root@0bdca65e6e8e:/# /opt/mono/bin/mono --version	
Mono JIT compiler version 3.2.6 (tarball Sat Jan 18 16:48:05 UTC 2014)
```

出现 `installed/opt` 说明安装成功了!

##运行控制台应用

首先，部署一款简单的控制台应用:

```
using System;	
namespace HelloWorld	
{
        public class Program
        {
                static void Main(string[] args)
                {
                        Console.WriteLine("Hello World");
                }
        }
}
```

本例中, 首先使用 VS 的命令 msbuild 来预构建版本，之后将结果显示到控制台：

```	
msbuild /property:OutDir=C:\tmp\helloworld HelloWorld.sln
```

当然, 也能从容器中使用 invokedxbuild or gmcs 来实现相同的目的。

配置容器去运行应用的 Dockfile 很简单:

```
*FROM friism/mono
MAINTAINER friism*

*ADD app/ .
CMD /opt/mono/bin/mono `ls *.exe | head -1`*
```

这个会用到上面创建的 friism/monoimage 文件。其编译后的文件也会位于 /appfolder 文件夹下：	

```
$ ls appHelloWorld.exe
```

CMD命令会使用 mono 来运行构建结果中的可运行项。下面构建运行看看：

```
$ docker build -t friism/helloworld-container ....$ docker run friism/helloworld-containerHello World
```

**成功！**

##Web应用

运行 self-hosting 的 [OWIN](http://www.asp.net/vnext/overview/owin-and-katana) web 应用也很简单. 我这里使用 [OWIN/Katana-on-Heroku post](http://friism.com/running-owin-katana-apps-on-heroku) 上的代码

```
$ ls app/	
HelloWorldWeb.exe  Microsoft.Owin.Diagnostics.dll  Microsoft.Owin.dll  Microsoft.Owin.Host.HttpListener.dll  Microsoft.Owin.Hosting.dll  Owin.dll
```

Dockerfile 开启容器的 5000 端口， CMD 会启动 web 应用并开始监听 5000 端口:

```
FROM friism/mono	
MAINTAINER friism

ADD app/ .
EXPOSE 5000	
CMD ["/opt/mono/bin/mono", "HelloWorldWeb.exe", "5000"]
```

启动容器，将 80 端口映射到运行 Docker 的机器上的 5000 端口:

```
$ docker run -p 80:5000 -t friism/mono-hello-world-web
```

接下来可以从 http://localhost/ 直接访问部署好的 OWIN 。

如果你是一名 DotNET 开发人员，希望这篇文章能将 Docker 带入你的视野。如果想了解更多关于如何使用 Docker 来部署应用的方法，可以看看 [Automated deployment with Docker – lessons learnt](https://www.hiddentao.com/archives/2013/12/26/automated-deployment-with-docker-lessons-learnt/) 这篇文章。

---
#####这篇文章由 [Michael Friis](http://friism.com/michael-friis) 发表，点击 [此处](http://friism.com/running-net-apps-on-docker) 可查阅原文。 [开源中国社区](http://www.oschina.net/) 的 [Ley](http://my.oschina.net/Ley11) 和 [petert](http://my.oschina.net/u/1422355) 贡献对本文的翻译。

#####The article was contributed by [Michael Friis](http://friism.com/michael-friis) , click [here](http://friism.com/running-net-apps-on-docker) to read the original publication.
