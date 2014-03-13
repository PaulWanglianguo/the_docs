#Docker commandline Tutorial | Running Apache http server inside docker container

#Docker命令行教程 | 在Docker容器中运行Apache HTTP服务器

***

This article will serve as a very basic hands on tutorial of docker commandline. I will be assuming that you are already aware of docker. If not, you can read about docker in one of my other posts here.

本文是一篇 Docker 命令行的基础教程，我假设读者已经对 Docker 有所了解，如果您确实不太了解 Docker，请参考我在另一篇 [文章](http://2mohitarora.blogspot.com/2013/11/docker-is-best-fit-for-continuous.html) 中对 Docker 的介绍。

***

###What are we trying to do here?
###本文中我们要做什么？

***

Run Apache http server inside a docker container and access it from host machine.

在 Docker 容器中运行 Apache HTTP 服务器，并从 Docker 所在的主机访问该服务器。

***

###What is the purpose?
###本文的目的为何？

***

Familiarize ourself with basic docker commands and feel its power.

本文的目的在于让读者熟悉基本的 Docker 命令，感受 Docker 的强大。

***

###Where am i running everything?
###本文中软件的运行环境是什么？

***

I am running everything on my Mac, We will first be creating a CentOS VM using [Vagrant](http://www.vagrantup.com/). Within CentOS VM, we will be installing docker and then run apache http server inside a docker container. (You don't have to necessarily run docker in VM, you can directly run it on your machine. If you decide to run it directly on your machine, please ignore Step 1 below)

文中所有内容都是在我的 Mac 上完成的，我们首先要用 [Vagrant](http://www.vagrantup.com/) 来创建一个 CentOS 虚拟机。在虚拟机中，我们安装 Docker，然后在 Docker 容器中运行 Apache HTTP 服务器（读者也可以在自己的机器上直接安装 Docker，跳过第一个步骤）。

***

######**译者注：Windows和Mac系统暂时都无法直接安装Docker，如果读者想直接安装Docker，需要选择一个适当的Linux发行版，具体请参考 [Docker官方文档](http://docs.docker.io/en/latest/) 中的Installation一节**

***

###Pre-requisites
###所需软件

***

You should have Oracle Virtual Box and Vagrant installed on your machine (only if you decide to run docker inside VM).
Let's get started.

您需要在电脑中安装 [Oracle Virtual Box](https://www.virtualbox.org/) 和 [Vagrant](http://www.vagrantup.com/) 。如果已经装好，那我们就开始吧！

***

###Step 1: Create a CentOS VM 
###步骤1：创建 CentOS 虚拟机

***

Inside a directory on your machine (~/vagarnt in my case), create a file named Vagrantfile with following contents

在本地目录（本文中使用~/vagrant目录）中创建一个`Vagrantfile`文件，内容如下：

***

    # -*- mode: ruby -*-
    # vi: set ft=ruby :
    VAGRANTFILE_API_VERSION = "2"
    Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
      config.vm.box = "centos65"
      config.vm.box_url = "https://github.com/2creatives/vagrant-centos/releases/download/v6.5.1/centos65-x86_64-20131205.box"
      config.vm.network :forwarded_port, guest: 80, host: 8080
      config.vm.network :public_network
      config.ssh.forward_agent = true
    end

***

I am not going to explain What Vagrant and Vagrantfile does, if you are not aware of them, please read about them. Also create a directory called htdocs parallel to Vagrantfile and inside htdocs directory create a file index.html with following contents.

在这里，我就不解释 Vagrant 和 Vagrantfile 了。`Vagrantfile`创建好后，在 Vagrantfile 所在目录中再创建一个 `htdocs` 目录并在其中创建一个 `index.html` 文件，html文件内容如下：

***

    This file is being server by http server running inside docker container

***

Once Vagrantfile and htdocs directory is ready. Execute following commands

Vagrantfile 文件和 htdocs 目录都创建好了，现在我们来执行下面的命令：

***

    vagrant up # 启动虚拟机，这一步可能会让我们选择网卡
    vagrant ssh # 虚拟机启动后，通过ssh登陆到虚拟机中
   
***   

###Step 2: Install docker 
###步骤2：安装 Docker

***

Run following commands to install docker

执行下面的命令来安装 Docker：

***

    sudo yum -y update # Update Installed packages
    sudo yum install docker-io # Install Docker 
    sudo service docker start # Start Docker

***

###Step 3: Setup Docker
###步骤3：设置 Docker

***

Docker needs a base image to work. This is the image from which all our containers will be running (either directly or indirectly). To pull the base image

Docker 需要一个基本的镜像才能运行，我们的所有容器都是（直接或间接）基于这样一个镜像来运行的，下面的命令把一个基本镜像 pull 到本地：

***

    sudo docker pull centos # Download base image
    
***

######**译者注：在大陆是无法顺利下载官方镜像的，解决方案请参考 [《使用 Golang 为 Docker 编写一个简单的 HTTP 代理》](http://www.dockboard.org/docker-http-proxy-with-golang/) 中的解决方案。**

***

###Step 4: Create first custom container image
###步骤4：为我们的容器创建第一个镜像

***

    # 以centos镜像作为基础镜像，我们启动自己的容器并在其中执行/bin/bash命令
    # 注：-t -i参数用于创建一个虚拟的命令行。
    sudo docker run -t -i centos /bin/bash 
    
***    
    
You have successfully started your first docker container and you are inside the container. Execute following commands inside the container

现在我们已经成功的运行了自己的第一个容器，并且进入到容器的命令行界面中。在容器中，我们执行下面的命令：

***

    yum -y update # 更新软件包
    yum install which # 安装which命令
    yum install git # 安装Git
    
***

Press Ctrl + d to kill the container.

安装完成后，按 `Ctrl + d` 来退出容器的命令行。

***

    #执行sudo docker ps -a，可以看到被我们终止的容器
    CONTAINER ID        IMAGE               COMMAND             CREATED......
    da9031d3568f        centos:6.4          /bin/bash           5 minutes ago.....

***

Commit our changes to a new container

把我们所做的改变提交到一个新的容器：

***

    # 这里我们创建一个自己的基础容器，容器中安装好了文章中所需的常用工具。读者的容器id可能与文章中的有所不同，以上一步docker ps -a的结果为准。
    sudo docker commit da90 custom/base

***

Once new container is committed

容器成功提交后，执行 `sudo docker images` ，我们会看到刚才提交的容器（如下面的结果所示）。我们就以这个容器为基础容器，再来创建一个新的容器。

***

    
    REPOSITORY          TAG                 IMAGE ID            CREATED            
    custom/base         latest              05b6cecd370b        2 minutes ago      
    centos              6.4                 539c0211cd76        10 months ago      
    centos              latest              539c0211cd76        10 months ago...

***

###Step 5: Start a new container and install apache
###步骤5：创建新的容器，并安装 Apache

***

    # 以custom/base容器为基础，运行一个新的容器。
    sudo docker run -t -i custom/base /bin/bash 
    # 安装httpd
    yum install httpd

***

###Step 6: Commit the changes again
###步骤6：再次提交新的容器

***

Press Ctrl + d to kill the container.

按 `Ctrl + d ` 来退出容器的命令行，然后执行命令：

***

    # This will commit our httpd related changes of step 5 and create a new base container image called custom/httpd. Container id will be different for you, execute sudo docker ps -a to find the container id. 
    # 这个命令会把步骤5中我们安装httpd带来的改变提交到新的名为custom/httpd的容器镜像中。你的容器id可能会和文章中有所不同，以sudo docker ps -a命令的结果为准。
    
    sudo docker commit aa6e2fc0b94c custom/httpd 

***

I am sure you have already realized that we have just created a reusable container image with http server installed on it. This idea can be used for anything and everything. You can create container for each component of your technology stack and they can be used on developer machine as well as production environment.

你应该已经发现了，我们创建了一个带有 HTTP 服务器并可以复用的容器镜像。你可以根据这种思想，为自己所需的每个组件都创建一个容器，然后把这些容器复用于开发环境或者生产环境。

***

###Step 7: Running http server
###步骤7：运行 HTTP 服务器

***

    
    # -v will Mount a volume from VM to the container which was also shared from host to Vagrant VM.
    # -v参数把主机共享给虚拟机的一个卷挂载到容器中
    # -p forward VM port 80 to container port 80; VM port 80 is mapped to host port 8080 in Vagrantfile 
    # -p参数把虚拟机的80端口映射到容器的80端口；虚拟机的80端口在Vagrantfile中被绑定到主机的8080端口，也就是：主机8080->虚拟机80->容器80
    sudo docker run -t -i -p 80:80 -v /vagrant/htdocs:/var/www/html custom/httpd /bin/bash
    
    # 启动Apache
    apachectl -k start 

***

###Step 8: Open a browser and test
###步骤8：在浏览器中测试

Hit http://localhost:8080; you should see the index.html we created in step 1.

在浏览器中浏览http://localhost:8080，你应该可以看到步骤1中html文件的内容。

***

###Conclusion:
###总结
I am sure you must have realized the power of docker by now. It create lightweight reusable images that fits perfectly in continuous delivery use case. We will be doing the exercise done in this tutorial again but using Docker files concept. Stayed tuned.

我想，你现在一定已经感受到 Docker 的强大了。用 Docker 可以创建轻量级、可复用的镜像，这样的镜像非常适合连续安装软件的情况。后面的文章中，我会教大家如何通过 Dockerfile 来完成本文中的工作，敬请期待。

######**译者注：连续安装软件的情况，作者意思是指通过 Docker 镜像中“层”（ layer ）的概念，连续的层中，每一层都安装了所需的一种或几种软件。**
