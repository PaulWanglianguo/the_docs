#Docker Quicktip #2: exec it, please!
#Docker Quicktip #2: 用exec执行命令！

***

Often when creating a Docker container it is necessary to do a bit of setup before starting the main process you wanted. Sometimes this is just a one-time setup for the first time the container starts (setting up database users, importing data, etc), other times it’s setting up the environment to get your process going (as many init.d scripts also do). In any case some script is needed to run before running the main application that the container was created for.

通常在创建一个 Docker 容器，需要在容器启动主要程序之前要进行一系列的设置。有时候这些设置只是在容器首次启动时做一次设置（比如设置数据库用户，导入数据等等），有时还需要设置让容器里的程序跑起来的环境（和那些 init.d 启动脚本做的事情是一样的）。不管如何，都需要为 Docker 容器中运行的主要程序写一些脚本。

***

Let’s take an image I recently created: [github: cpuguy83/docker-postgres](https://github.com/cpuguy83/docker-postgres/tree/d59c8578fabfd2e5a417d499836cd1643eac92b4)

我们来看一下我最近在Github上已经创建好的一个镜像：[github: cpuguy83/docker-postgres](https://github.com/cpuguy83/docker-postgres/tree/d59c8578fabfd2e5a417d499836cd1643eac92b4)

***

**Dockerfile**

    FROM cpuguy83/ubuntu
     
    RUN apt-get update && apt-get install -y postgresql postgresql-contrib libpq-dev
    ADD pg_hba.conf /etc/postgresql/9.1/main/pg_hba.conf
    RUN chown postgres.postgres /etc/postgresql/9.1/main/pg_hba.conf
    ADD postgresql.conf /etc/postgresql/9.1/main/postgresql.conf
    RUN chown postgres.postgres /etc/postgresql/9.1/main/postgresql.conf
    RUN sysctl -w kernel.shmmax=4418740224 && /etc/init.d/postgresql start && su postgres -c "createuser -s -d root && psql -c \"ALTER USER root with PASSWORD 'pgpass'; CREATE USER replication REPLICATION LOGIN CONNECTION LIMIT 1 ENCRYPTED PASSWORD 'replpass'\""
     
    EXPOSE 5432
    VOLUME /var/lib/postgresql
    ADD pg_start.sh /usr/local/bin/
    RUN chmod +x /usr/local/bin/pg_start.sh
     
    CMD ["/usr/local/bin/pg_start.sh"]
    
**pg_start.sh**

    #!/bin/bash
    
    if [[ ! -z "$MASTER_PORT_5432_TCP_ADDR" ]]; then
        conn_info="host=${MASTER_PORT_5432_TCP_ADDR} user=replication password=${REPLICATION_PASS}"
     
        echo "primary_conninfo = '${conn_info}'" > /var/lib/postgresql/9.1/main/recovery.conf
        echo "standby_mode = 'on'" >> /var/lib/postgresql/9.1/main/recovery.conf 
    fi
    sysctl -w kernel.shmmax=4418740224
    su postgres -c "/usr/lib/postgresql/9.1/bin/postgres -D /var/lib/postgresql/9.1/main -c config_file=/etc/postgresql/9.1/main/postgresql.conf $PG_CONFIG"


***

There are a couple of issues with this, which I’ll address in a future post. Here I’ll focus on the CMD line of the Dockerfile and the last line of pg_start.sh.

上面的文件还存在一些问题，我们后面再说。这里我们先看 Dockerfile 的 CMD 这一行和 pg_start.sh 的最后一行。

***

First, let’s change `CMD` to `ENTRYPOINT` as we learned in the [previous](http://www.tech-d.net/2014/01/27/docker-quicktip-1-entrypoint/) article.

首先我们把`CMD`换成`ENTRYPOINT`，这个技巧我们在[前一篇文章](http://www.tech-d.net/2014/01/27/docker-quicktip-1-entrypoint/)中已经学过了。

***

The next bit is becoming a pet-peeve of mine when using some of my older images.

然后就是使用我的一些老镜像时让我恼火的事情了。

***

When calling postgres we are just doing it directly (well… through su, but still directly in terms of the process).

当调用 postgres 时，我们是通过 su 用户直接来调用其进程的。

***

Calling it this way breaks the world. With postgres it might not be too bad but with other apps it may wreck havoc.

这种调用方法可以让整个 Docker 容器崩溃。调用 postgres 还好，要是直接调用其他的程序，就有可能引起大灾难了。

***

As it stands right now if we try to stop this container docker will hang for a few seconds and then just kill it. Go ahead… try it. There is even a setting in Docker for how long to wait before killing the container (docker stop -t Nseconds, default is 10) with SIGKILL.

假设就这样直接调用了，如果我们尝试停止容器，Docker会卡住几秒钟，然后直接杀掉容器。试试吧，很刺激哦。在Docker中还有一个选项可以设置卡住多久就调用SIGKILL关闭容器（docker stop -t N，N是秒数，默认是10）。

***

Run “docker logs $container_id” to see the proof.

执行`docker logs $container_id`可以证明上面所述。

***

Why is it doing this? The signals to stop the process are being sent to the startup script and not postgres. I am not trapping signals in my startup script… nor should I be.
So how do I fix it? With exec.

为什么会这样呢？因为关闭进程的信号是发送到启动脚本的，而不是postgres。在我的启动脚本中并没有监听这个信号，当然我也不可能在启动脚本中去监听信号。

那么我要怎么解决这个问题呢？

用exec。

***

Let’s change the last line of pg_start to use exec instead:

我们修改一下 pg_start.sh 的最后一行，用 exec 来执行命令：

***
    exec su postgres -c "/usr/lib/postgresql/9.1/bin/postgres -D /var/lib/postgresql/9.1/main -c config_file=/etc/postgresql/9.1/main/postgresql.conf $PG_CONFIG"

***

There we go, docker will now cleanly shutdown my postgres process instead of SIGKILLing it.
Again, run “docker logs $container_id” for the proof.

好了，现在 Docker 会先彻底关掉 postgres 的进程，而不是用 SIGKILL 来关闭了。

***

Docker allows you to proxy all signals (this is enabled by default) to the running process in the container. Need to send HUP to the running process in the container? Send it to the docker container process. You can even use this functionality to run process monitoring on your host for your containerized processes. See [Docker host integration](http://docs.docker.io/en/latest/use/host_integration/) for an example

Docker 默认允许你把所有的信号代理给容器中正在运行的进程。如果需要发送 HUP 信号给容器中运行的进程，只需发送给 Docker 容器的进程就可以了。你还可以用这个特性，启动一个进程来监听你的 Host 文件中的代理关系，通过这样监听在 Docker 容器中运行的进程。具体请参考 [Docker host integration](http://docs.docker.io/en/latest/use/host_integration/) 这篇文章。

***
