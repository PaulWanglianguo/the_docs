##A reveal.js Docker Base Image with ONBUILD
##使用ONBUILD构建reveal.js的Docker镜像

Docker 0.8 introduced the ONBUILD instruction for Dockerfiles. I'll admit, when I read the release announcement, I glossed right over this new feature. Now that I've had time to read more about it, I can see its potential for creating build environments.

Docker0.8发布了Dockerfile的一个新的指令ONBUILD。我必须承认，当我在阅读[发布公告](http://blog.docker.io/2014/02/docker-0-8-quality-new-builder-features-btrfs-storage-osx-support/)的时候，我忽视了这个新的特性。现在我有充分的时间[再读一遍它](http://docs.docker.io/en/latest/reference/builder/#onbuild)，我发现了它在构建环境上的潜力。

Let me show you by containerizing reveal.js.

让我通过把reveal.js容器化来向您展示这个新特性。

###The Base Image

### 基础镜像

Reveal.js bills itself as "a framework for easily creating beautiful presentations using HTML." Most reveal.js features work just fine when we load the presentation directly from the filesystem. But some features, like Markdown support, require that we load the presentation from a running web server. A Docker container can tie everything up in one tidy package that we can move and run anywhere.

[Reveal.js](http://lab.hakim.se/reveal-js/#/)把自己定义为“使用HTML技术来展示漂亮幻灯片的一个简易框架”。当我们直接从文件系统装在幻灯片的时候，reveal.js的大多数的特性是正常工作的。但是有些特性，比如Markdown的支持，需要我们通过一个Web服务器来装载幻灯片。一个Docker的容器可以把所有东西装载到一个盒子里，方便我们在任何地方移动和执行。

We could write a Dockerfile that installs NodeJS, downloads reveal.js, installs its npm dependencies, and inserts our Markdown slides into the container on build. We could then copy and build this Dockerfile everywhere we want to develop or run a slideshow server. Or we could keep cramming new markdown files into a single container by rebuilding or scping them in a backdoor. Or we could build a generic image that, when run, requires us to volume mount our slides from the host or another container.

我们可以写一个Dockerfile让它在构建的时候安装NodeJS,下载reveal.js,安装它的npm依赖以及把我们的Markdown的幻灯片插入到容器中。接下来我们就可以在任何我们要开发或者执行幻灯片服务器的地方来复制并构建这个Dockerfile.我们也可以可以通过重构的方式或者scp的后门来不断地把新的Markdown文件塞入到单一的容器中。或者我们可以构建一个通用的镜像，当我们执行的时候，要求我们把主机上的幻灯片通过挂载的方式共享到另外一个容器中。

ONBUILD介绍了一种新的方法。通过它，我们可以构建一个基本镜像来完整地配置reveal.js和web服务器，但是延迟插入我们的幻灯片的内容直到我们构建一个子的Dockerfile。Dockerfile中的关键部分如下

```
FROM ubuntu:13.10
MAINTAINER Peter Parente <parente@cs.unc.edu>

# elided dependency setup, then ...

ONBUILD ADD slides.md /revealjs/md/

# elided final setup
```

We can build the base image once and store it in the Docker registry via docker push, or setup a trusted build. (I've done the latter, which shows the full Dockerfile, if you're interested).

我们可以一次构建一个基础镜像并且通过Docker push的方式把它存储到Docker的registry中，或者建立一个可信的构建。(我已经[实现了后者](https://index.docker.io/u/parente/revealjs/),如果你有兴趣，可以查看完整地Dockerfile)

### The Slideshow Image
### 幻灯片展示镜像

Later, when we have a new slideshow, we can put a one-line Dockerfile:

后来当我们有一个新的幻灯片的时候，我们可以写一个只有一行的Dockerfile

```
FROM parente/revealjs
```
in the same folder as our slides.md:

在和我们slides.md共同的目录下：

```
# Docker + reveal.js

### A reveal.js Docker Base Image with ONBUILD

---

## Write more slides
```

and do a build:

并且执行一次构建

```
$ docker build -t slideshow .
```

During this build, Docker executes the delayed ONBUILD ADD slides.md /revealjs/md/ instruction. It pulls our local slides.md file into the child image. The result is a portable Docker image that can run our slideshow server anywhere Docker runs.

在这次构建的过程中，Docker执行延迟了的ONBUILD ADD slides.md 、revealjs/md/ 的指令。它把本地的slides.md文件拉到子镜像中。执行的结果是一个可移植的Docker镜像，可以在任何有Docker的地方运行我们的幻灯片服务器。

```
$ docker run -d -P myslides
```
And of course all the standard docker run options still apply: a fixed port, container naming, running in the foreground, etc.

当然它支持所有docker run的标准选项:一个固定的端口，容器命名，在前台运行等等。

### A Development Story
### 一个开发故事

The build and run steps above are perfect for producing a completed slideshow container. Cycling between build and run while developing the slides is less than ideal, however.

上面的构建和执行的步骤可以完美的生成一个完整地幻灯片展示的容器。在开发幻灯片的过程中不断地在构建和执行中循环却不怎么理想。

Mixing techniques can alleviate this pain. During development, we can volume mount the slides from the host into a running instance of the base image. This setup lets us make changes to the slides.md on the host, refresh the browser, and see the impact immediately, without tearing down, rebuilding, and re-running the container.

混合的技术可以用来减轻这个问题。在开发的过程中，我们可以通过把主机上的幻灯片通过挂在的方式在一个基础镜像的容器上共享。这样当我们在主机上对slides.md进行修改的时候，通过刷新浏览器，我们可以立刻看到发生的变化，不需要把服务关掉，重新构建再重新执行容器。

```
$ docker run -d -P -v `pwd`:/revealjs/md parente/revealjs
```

When we're satisfied with our work, we can build the final, fully contained image for migration and deployment.

当我们满意我们的工作后，我们可以构建一个最终的版本，一个完整地可以用来迁移和部署的镜像。

```
$ docker build -t slideshow .
```

更多的细节，可以查看[源代码](https://github.com/parente/dockerfiles/tree/master/revealjs)或者[parente/revealjs repository](https://index.docker.io/u/parente/revealjs/)