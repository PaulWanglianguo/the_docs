## Building Complex Apps for Docker on CoreOS and Fig
## 使用 CoreOS 和 Fig 构建基于 Docker 的复杂应用

***

For Docker deployments, [CoreOS](https://coreos.com) is powerful but can be complex to setup. [Fig](http://orchardup.github.io/fig/) is simple, but hard to scale to multiple servers. This blog post will show you how to bridge the gap between building [complex multi-container apps using Fig](http://www.centurylinklabs.com/auto-loadbalancing-with-fig-haproxy-and-serf/) and deploying those applications into a production CoreOS system.

使用 Docker 来部署系统，[CoreOS](https://coreos.com) 的功能比较强大但是初期的学习曲线比较高。[Fig](http://orchardup.github.io/fig/)  相对而言比较简单，但是很难在多台服务器上做扩展。这篇博客将向您展示如何解决使用 [Fig 构建多个容器的复杂应用] (http://www.centurylinklabs.com/auto-loadbalancing-with-fig-haproxy-and-serf/) 并且把这些应用部署到基于 CoreOS 的生产环境。

***

### Complete Multi-Container Fig App
### 完整构建一个基于 Fig 的多容器应用

***

In last week’s blog post, we covered building a 4-container app in Fig. To cut to the chase, here is the ```fig.yml``` that gets you a 4-container app:

在上个星期的博文中，我们谈到了在 Fig 中构建有 4 个容器的应用。开门见山，下面这个 ```fig.yml``` 就是用来构建4个容器的应用的。

***

```
serf:
  image: ctlc/serf
  ports:
    - 7373
    - 7946
lb:
  image: ctlc/haproxy
  ports:
    - 80:80
  links:
    - serf
  environment:
    HAPROXY_PASSWORD: qa1N76pWAri9
web:
  image: ctlc/wordpress
  ports:
    - 80
  environment:
    DB_PASSWORD: qa1N76pWAri9
  links:
    - serf
    - db
  volumes:
    - /local/path/to/wordpress:/app
db:
  image: orchardup/mysql
  ports:
    - 3306
  volumes:
    - /mysql:/var/lib/mysql
  environment:
    MYSQL_DATABASE: wordpress
    MYSQL_ROOT_PASSWORD: qa1N76pWAri9
    
```

***

To show this 4-container system in action, you simply run ```fig up -d``` (which is the same command used to restart the running system).

如果要在你的环境中部署这个具有4个容器的系统，只需要简单运行 ```fig up -d``` (和重启系统的命令一样)。

***

```
$ fig up -d
Recreating ctlcblog_serf_1...
Recreating ctlcblog_db_1...
Recreating ctlcblog_web_1...
Creating ctlcblog_lb_1...

$ fig ps
     Name              Command         State                Ports               
-------------------------------------------------------------------------------
ctlcblog_serf_1   /run.sh              Up      49252->7373/tcp, 49253->7946/tcp 
ctlcblog_db_1     /usr/local/bin/run   Up      49254->3306/tcp                  
ctlcblog_web_1    /run.sh              Up      49255->80/tcp                    
ctlcblog_lb_1     /run.sh              Up      80->80/tcp                       

$ fig scale web=2
Starting ctlcblog_web_2...

$ fig ps
     Name              Command         State                Ports               
-------------------------------------------------------------------------------
ctlcblog_serf_1   /run.sh              Up      49252->7373/tcp, 49253->7946/tcp 
ctlcblog_db_1     /usr/local/bin/run   Up      49254->3306/tcp                  
ctlcblog_web_2    /run.sh              Up      49256->80/tcp                    
ctlcblog_web_1    /run.sh              Up      49255->80/tcp                    
ctlcblog_lb_1     /run.sh              Up      80->80/tcp          
```

***

As you can see, Fig is super easy to get started with and use, but the #1 question people had was how do you scale this to multiple servers?

你可以看到，Fig 的使用是非常简单和方便的，但第一个问题就是我们如何把它扩展到多个服务器上?

***

### How Do You Create CoreOS Containers From Fig Configurations?
### 你如何在 CoreOS 中基于 Fig 的配置来创建容器？

***

CoreOS is not the easiest or most approachable system in the world. Understanding how to write systemd config files and getting everything configured just right can be time consuming and frustrating.

CoreOS 不是这个世界上最简单或者说最容易上手的系统。理解并正确编写一个 systemd 配置文件是非常耗时而且让人困扰的。

***

Which is why I wrote a ```fig2coreos``` gem ([github.com/centurylinklabs/fig2coreos](github.com/centurylinklabs/fig2coreos)), which converts ```fig.yml``` to CoreOS formatted systemd configuration files automatically.

这个也就是为什么我写个一个 ```fig2coreos``` 的 gem ([github.com/centurylinklabs/fig2coreos](github.com/centurylinklabs/fig2coreos)),它可以把 ```fig.yml``` 自动转换成 CoreOS 格式的 systemd 配置文件。

***

```

$ sudo gem install fig2coreos
$ fig2coreos wordpress-app fig.yml coreos-dir
[SUCCESS] Try this: cd /Users/cardmagic/Sites/centurylinklabs.com/coreos-dir && vagrant up
$ cd coreos-dir && vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
[default] Importing base box 'coreos'...
[default] Matching MAC address for NAT networking...
[default] Setting the name of the VM...
[default] Clearing any previously set forwarded ports...
[default] Clearing any previously set network interfaces...
[default] Preparing network interfaces based on configuration...
[default] Forwarding ports...
[default] -- 22 => 2222 (adapter 1)
[default] -- 80 => 8080 (adapter 1)
[default] Running 'pre-boot' VM customizations...
[default] Booting VM...
[default] Waiting for machine to boot. This may take a few minutes...
[default] Machine booted and ready!
[default] Setting hostname...
[default] Configuring and enabling network interfaces...
[default] Exporting NFS shared folders...
Preparing to edit /etc/exports. Administrator privileges will be required...
Password:
[default] Mounting NFS shared folders...
[default] Running provisioner: shell...
[default] Running: /var/folders/9j/gkydy1sn1nsd73yd2n3l68400000gn/T/vagrant-shell20140224-18153-19i4zr8
$ cd coreos-dir && vagrant ssh
Last login: Mon Feb 24 23:57:38 UTC 2014 from 10.0.2.2 on ssh
   ______                ____  _____
  / ____/___  ________  / __ \/ ___/
 / /   / __ \/ ___/ _ \/ / / /\__ \
/ /___/ /_/ / /  /  __/ /_/ /___/ /
\____/\____/_/   \___/\____//____/
core@coreos-wordpress-app ~ $ 

```

***

For some reason (I am not sure why yet), the default version of CoreOS shipped when you ```vagrant up``` is not the latest version of coreos, which means it does not have [Fleet](https://github.com/coreos/fleet) installed and has an old version of Docker (0.7.2). Once the vagrant coreos box stays up and running long enough to download the latest version of coreos, you can apply the update by either running ```sudo reboot``` in the coreos image, or by running ```vagrant reload --provision``` in the generated directory.

由于某些原因(我也不知道为什么)，CoreOS 发布的默认版本，当你用 ```vagrant up``` 启动的时候，其实它不是最新的 CoreOS 版本，这个就意味着它没有安装 [Fleet](https://github.com/coreos/fleet)，并且上面的 Docker (0.7.2) 也是旧的.一旦 vagrant coreos box 已经启动并且下载完了最新的 CoreOS 的版本。你就可以通过在 CoreOS 的虚拟机中运行 ```sudo reboot``` 或者在 vagrant 的当前工作目录下运行 ```vagrant reload --provision``` 来执行更新的操作。

***

### What Does fig2coreos Do, Exactly?
### fig2coreos 到底做了什么呢？

***

The ```fig2coreos``` command parses your ```fig.yml``` file and generates a bunch of systemd config files. You can see the files in the directory that is created.

```fig2coreos``` 解析你的 ```fig.yml``` 并且生成一系列的 systemd 配置文件。你可以在它生成的目录中查看这些文件

***

```

$ ls coreos-dir/media/state/units/
db-discovery.1.service
lb-discovery.1.service
serf-discovery.1.service
web-discovery.1.service
db.1.service
lb.1.service
serf.1.service
web.1.service

$ cat coreos-dir/media/state/units/db.1.service
[Unit]
Description=Run db_1
After=docker.service
Requires=docker.service

[Service]
Restart=always
RestartSec=10s
ExecStartPre=/usr/bin/docker ps -a -q | xargs docker rm
ExecStart=/usr/bin/docker run -rm -name db_1 -v /mysql:/var/lib/mysql  -e "MYSQL_DATABASE=wordpress" -e "MYSQL_ROOT_PASSWORD=nqhT4hT0RT6k" -p 3306 orchardup/mysql
ExecStartPost=/usr/bin/docker ps -a -q | xargs docker rm
ExecStop=/usr/bin/docker kill db_1
ExecStopPost=/usr/bin/docker ps -a -q | xargs docker rm

[Install]
WantedBy=local.target

```

***

Which looks complicated, but is the CoreOS version of this part of your ```fig.yml``` file:

看上去很复杂的，是你 ```fig.yml``` 中关于 CoreOS 版本信息的那一部分：

***

```
db:
  image: orchardup/mysql
  ports:
    - 3306
  volumes:
    - /mysql:/var/lib/mysql
  environment:
    MYSQL_DATABASE: wordpress
    MYSQL_ROOT_PASSWORD: qa1N76pWAri9
    
```

***

In addition to the start script for MySQL, there is also an ```etcd``` auto-registry script via systemd in the db-discovery.1.service file.

除了 MySQL 的启动脚本之外，也有一个通过 systemd 实现的 ```etcd``` 自动注册脚本，它在 ```db-discovery.1.service``` 文件中。

***

```
$ cat coreos-dir/media/state/units/db-discovery.1.service
[Unit]
Description=Announce db_1
BindsTo=db.1.service

[Service]
ExecStart=/bin/sh -c "while true; do etcdctl set /services/db/db_1 '{ \"host\": \"%H\", \"port\": 3306, \"version\": \"52c7248a14\" }' --ttl 60;sleep 45;done"
ExecStop=/usr/bin/etcdctl rm /services/db/db_1

[X-Fleet]
X-ConditionMachineOf=db.1.service
```

***

This registers the service into the ```etcd``` key/value store. In this example Fig application, we used Serf to make the Docker containers self-aware, but etcd is the standard CoreOS alternative to Serf available in all CoreOS containers.

这个把服务注册到 ```etcd``` 的键值存储中。在这个 Fig 应用的例子中，我们使用 Serf 来实现 Docker 容器的自省，但是 ```etcd``` 是标准的 CoreOS 容器用来替换 Serf 的服务。

***

Do you need both Serf and etcd? No. The biggest difference between the two is where the daemons are run. In etcd, the etcd daemons are run **outside** of the Docker containers and it is up to a system like systemd to keep the etcd key/value pairs up to date. In Serf, the serf daemons are run **inside** the Docker containers, so when a container dies or goes offline, the peers of that container figure it out automatically.

你既需要 Serf 又需要 etcd 么？不，两者最大的区别在于守护进程的运行位置。etcd 守护进程是运行在 Docker 容器**外部**的，它就像 systemd 一样的一个系统来保证 key/value 的更新。Serf 的守护进程是运行在 Docker 容器**内部**的，所以当容器死亡或者下线的时候，同等的容器会被自动移除。

***

These are two very different philosophies for configuration and system management, but in the end result is essentially the same. Both Serf and etcd have the same ability to let Docker apps become more aware of their containers and the roles their containers play and interconnect.

在系统管理和配置管理中有两个不同的哲学，但是结果殊途同归。Serf 和 etcd 都可以让 Docker 的应用察觉到它们所对应的容器，它们所扮演的角色以及互相的联系。

***

### So How Do You Scale Docker To Multiple Hosts?
### 那么如何把 Docker 扩展到不同的主机上？

***

You can (in theory) use Fig on each one of your servers, each running independent Docker daemons. However linking containers becomes more complicated and network issues make it non-trivial, even with Serf.

理论上来说，你可以在你每一台服务器上使用Fig,让每一台机器运行一个独立的Docker守护进程。然而链接容器变得很复杂，而且网络问题也变得很严重，即使是用Serf也是一样。

***

I am getting closer to showing how to deploy multi-host Docker in this series of blog posts, but this post won’t show you exactly how just yet. Subscribe to get weekly updates and we will get there in due time.

在这一系列的博文中，我们差不多可以向您展示如何部署一个多主机的 Docker 服务了。但是这篇博文任然不会给你明确的答案。订阅我们每周的更新，我们最终将向您展示一切。

***

In the meantime, so that you know that I am not lying to you, watch this [2-minute YouTube Video about CoreOS Fleet](http://www.youtube.com/watch?v=u91DnN-yaJ8) to see an example of a real multi-host CoreOS deployment. You can see the power of CoreOS and Fleet in this simple video, but it doesn’t connect the dots of how to put all the components together to re-create the demo.

与此同时，你也知道我没有在骗你，看这个 [youtube上2分钟的关于CoreOS Fleet的视频](http://www.youtube.com/watch?v=u91DnN-yaJ8) 来了解一个真实的关于部署多主机CoreOS服务的例子。你可以在这个简单的视频中发现 CoreOS 和 Fleet 的强大。但是现在还没法把所有的组件组合在一起来重现视频中的例子。

***

### Conclusion
### 小结

***

We are getting very close to showing you how to deploy production multi-host Docker apps in a cloud environment. Each week, I am getting you closer and closer to the answer by introducing slightly more complexity into the example. CoreOS and Fig are just two Docker technologies being built to manage complex applications. In the future we will show you other ways you may want to accomplish this same task with tools like [Flynn](https://flynn.io/) and [Deis](http://deis.io/).

我们已经非常接近向您展示如何部署一个多主机的 Docker 应用到一个云生产环境中。每周我都通过不断向您展示一些复杂的例子来带领你越来越接近这个目标。CoreOS 和 Fig 只是两个基于 Docker 技术开发的用来部署复杂应用的工具。未来我们将向您展示另外的工具比如 [Flynn](https://flynn.io/) 和 [Deis](http://deis.io/)，它们也可以用来实现相同的目的。
