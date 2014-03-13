#Run Several Processes in Docker Container

#在 Docker 容器中运行多个进程

***

What I like the most about Docker project is new opportunity to deploy and distribute software. Many times I’ve been to situation when I wanted to play with some software and get exited about, but after I read installation manual my excitement totally gone. Non trivial applications, requires quite a lot dependencies: runtimes, libraries, databases.

我最喜欢 Docker 的地方就是它那全新的软件部署和发布方式。我经常会看到令人兴奋的软件，兴冲冲的准备研究一下，然后被其安装说明在第一时间扼杀掉我的兴趣。稍微大一点的软件往往都需要很多依赖：运行时，库，数据库。

***

With docker, the installation instruction got reduced to something like:

在Docker中，这些安装步骤被精简成这样子：

***

    $ docker pull vendor/package
    $ docker run vendor/package

***

Simply like that, forget about missing Java Runtime on your server. It suits perfectly for TCP/HTTP applications.

基本上就是这么几行命令了，不需要你在服务器上装什么 Java 运行时环境，非常适合 TCP/HTTP 的程序。

***

Being messing around [Seismo](https://github.com/seismolabs/seismo) project I realized, I want to go exactly same way. Since it has few dependencies now, MongoDB and NodeJS – it should be easier to anyone to try it, even if they do not use that setup. I was happy to see, that GitHub currently offers great support for Docker. Namely, if you have repo with Dockerfile inside, each time you push the code, docker image got rebuild and pushed to public [index](https://index.docker.io/).

被自己的 [Seismo](https://github.com/seismolabs/seismo) 项目搞的焦头烂额后，我还是希望能够按原计划把它实现。因为这个项目的依赖很少，只有 MongoDB 和 NodeJS ，任何人都可以很轻松的上手，虽然他们不经常安装设置。令人开心的是，Github 现在对 Docker 提供了很好地支持，它会检测你的项目，如果发现其中有 `Dockerfile`，那么当你 push 代码的时候，Docker 镜像会自动重建然后 Github 会把新的镜像 push 到 Docker 的 [公共镜像列表](https://index.docker.io/) 中。

***

I’ve created `Dockerfile` that would build up image, ready to have Seismo run inside.

我创建了一个 `Dockerfile`，可以构建一个运行着Seismo的镜像。

    FROM    ubuntu:latest
    
    # Git
    RUN apt-get install -y git
    
    # MongoDB
    RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
    RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/10gen.list
    RUN dpkg-divert --local --rename --add /sbin/initctl
    RUN ln -s /bin/true /sbin/initctl
    RUN apt-get update
    RUN apt-get install mongodb-10gen
    RUN mkdir -p /data/db
    
    # NodeJS
    RUN apt-get update --fix-missing && apt-get upgrade -y
    RUN apt-get install -y wget curl build-essential patch git-core openssl libssl-dev unzip ca-certificates
    RUN curl http://nodejs.org/dist/v0.10.22/node-v0.10.22-linux-x64.tar.gz | tar xzvf - --strip-components=1 -C "/usr"
    RUN apt-get clean && rm -rf /var/cache/apt/archives/* /var/lib/apt/lists/*
    
    # Seismo
    RUN git clone https://github.com/seismolabs/seismo.git /seismo
    RUN cd /seismo; npm install
    ENV PORT 8080
    EXPOSE 8080
    
    WORKDIR /seismo
    ENTRYPOINT ["./bin/run.sh"]
 
***

It’s based on latest Ubuntu server, installs Git, MongoDB and NodeJS runtime and clones Seismo itself inside image.

这个镜像基于最新的服务器版 Ubuntu，安装了 Git，MongoDB 和 NodeJS 的运行环境，并且把 Seismo 的库克隆到镜像内部。

***

But, I’ve met a problem to start few processes inside the container. Since I need both MongoDB for storage and NodeJS for API server, it’s required both be running inside one container. If shell script just starts one, mongod for example, node app.js is not executed.

但是，在容器内部启动多个进程的时候，我遇到了一些问题。因为我需要 MongoDB 作为存储，还需要 NodeJS 做 API 服务器，需要在容器中同时运行这两个程序。如果shell脚本只启动了一个，比如，只启动了 `mongod`，那么`node app.js`就不会执行。

***

I was a little worried, thinking it’s not possible to run more that one process inside container.

我有点儿小慌张，以为容器内部无法同时运行多个进程。

***

But solution was found. I’ve created another shell script that starts mongod as background process and starts node after.

但我还是找到了解决方案。我重新写了脚本，把mongod作为后台进程开起来，然后再启动nodejs。

***

    #!/bin/bash
    mongod & node ./source/server.js

***

That worked as charm.

完美解决问题！
